---
layout: post
title: Creating a Kubernetes Operator in .NET
date: 2023-09-12 10:00:00 -0400
description: >
  {{TBD}}
img: kubernetes-operator-dotnet/hero-image.jpeg
fig-caption: A man orchestrating a process with many computers
tags: [Kubernetes, dotnet, .NET]
---

> **Note:** The code seen in this post is based on KubeOps 7.6.0.  There is work in progress on version 8.0, but that is a preview build of the library and was not available when I started this research.  "Your mileage may vary."

The [Kubernetes Operator Pattern][kubernetes-operator-pattern] is a Kubernetes development pattern aimed at simplifying complex Kubernetes workload deployments.  The idea of a Kubernetes Operator is similar to what you would think of when you think about a human who is operating a system - they know how the system is deployed, have a deep working knowledge of how it works, and they know how to react if a problem were to arise.  Kubernetes Operators work in a similar manner - they deploy and manage Kubernetes workloads, and they can also handle some error conditions for a given workload.

Some use cases for a Kubernetes Operator might include:

* Deploying a complex application
* Handling upgrades to an application
* Simulating a failure
* Handling leadership election within a distributed application without an internal election process

Examples of commonly used Kubernetes Operators include:

* [GitHub Actions Runner Controller][github-arc]
* [PostgreSQL Operator][pgsql-operator]
* [Ansible AWX Operator][awx-operator]
* [Velero][velero]

## Anatomy of an Operator

![CNCF Kubernetes Operator Pattern Diagram][cncf-operator-pattern-diagram]
*Kubernetes Operator Pattern High Level Diagram*

A Kubernetes Operator consists of an Operator (or Controller) workload and [Custom Resource Definitions (CRDs)][crd].  At a high level, the Operator watches for changes to the CRDs, and based off of those changes it'll create and/or modify resources within the Kubernetes cluster.  Managing these resources is handled by a combination of Reconcilers, Finalizers, and Webhooks.

### Reconcilers

A Reconciler is exactly what it sounds like - it *reconciles* the resources which are managed by the operator.  Put another way, the reconciller is responsible for maintaining the desired state of whatever the Operator is managing.  It is the most heavily exercised component of the operator, and it's the only **required** component within an Operator.  Reconcilers are usually called for every event that triggers a Controller since they are responsible for ensuring the desired state of the system.

### Finalizers

Finalizers are used to implement "cleanup" tasks when a Kubernetes resource is deleted.  This can include cleaning up resources, calling external API's to unregister services, and deleting other CRDs which could trigger other events within an Operator.  An object within a Kubernetes cluster cannot actually be deleted until **all** finalizers for that resource type have been completed; finalizers always run before an object is deleted.

> **NOTE** In the C# library I'm showing in this example, finalizers are only attached to entities.  In practice, a finalizer could be attached to any Kubernetes resource and can be used to execute tasks based on any Kubernetes resource.

### Webhooks

Webhooks are used to extend the [normal admission behavior][kubernetes-admission-behavior] of the Kubernetes API.  Kubernetes object admission has [two phases - mutation and validation][kubernetes-object-admission-phases].  The types of webhooks which can be created follow those two phases:

* **Mutation** - these webhooks run when an object is created which matches a defined set of rules in the corresponding `MutatingAdmissionWebhook` configuration object.  These webhooks run in the *mutating* phase of object initialization, and will run in series.  The objective of these webhooks is to *mutate* a new object (such as filling in originally unset fields or adding additional labels).  This type of webhook should be used with caution, as it could lead to confusion among users when they get back an object that's different than what they intended to create.
* **Validation** - these webhooks run in the *validation* phase of a Kubernetes object (which occurs after the mutating phase), and extend the object validation behavior.  They validate objects which match configuration set in a corresponding `ValidatingAdmissionWebhook` configuration object.  Because these are called after the mutation phase, these webhooks may not modify the objects.  These webhooks are all called in parallel.  Object validation failure will occur if any one of the validation webhooks fails.

## Writing an Operator

Kubernetes operators can be written in a [variety of languages][writing-your-own-operator].  Since I'm familiar with C\#, I decided to write my sample operator in C\#.  The code samples in the remainder of this document will also be in C\#.  The premise of this operator is similar to the premise of any other operator - a controller will be monitoring for changes to a CRD Object, and will act based on the state change of that object.  In the case of this example, I'll be creating an operator which deploys a ConfigMap and a Deployment consisting of an Ubuntu pod.

### KubeOps NuGet Package

I'll be using the [KubeOps][kubeops] library for authoring my Operator in these examples.  This library is listed in the [official Kubernetes documentation][writing-your-own-operator] which is why I picked it.  Behind the scenes it is making use of the [KubernetesClient CSharp][kubernetesclient-csharp] library.  The KubeOps library extends the base AspNetCore project type and relies on the included DI framework and project patterns, so if you are familiar with this type of project these concepts should feel familiar.

The KubeOps library handles a lot of things for us, such as creating a Dockerfile for building the controller image, and creating the kubectl files to deploy RBAC, CRDs, and workloads required for our operator.  This allows us to work almost fully within C# for creating our operator.

### Creating the project

We will first create our solution and add the KubeOps NuGet package.  For this example, I'm going to use the *web* template which creates an empty baseline AspNetCore project.

``` bash
dotnet new web -n YourSolutionNameHere
dotnet install KubeOps
```

In the project, create folders for our Entities, Controllers, Finalizers, Webhooks, and Reconcilers.  These folders will hold all of the parts required for our Operator.

### Defining the CRD Entity

Operators work based off of [CRD objects][crd], so first we need to define our CRD entity.  In KubeOps, this is accomplished in C# code.  For our example, we'll create a `ubuntu.example.local/v1alpha1` object, which translates to an entity type named `ubuntu` within the `v1alpha` version of the `example.local` API.  To do this, we need to create three classes:

* **Spec** - specifies the user-defined fields within the CRD
* **Status** - specifies the status fields used by the operator, if any.
* **`CustomKubernetesEntity` Implementation** - Defines the actual entity.  Generally this will be an empty class.

Additionally, I generally will define what I want my CRD to look like down below my entity in a comment just as a reference.  My empty Entity class looks like this:

``` cs
public class UbuntuV1Alpha1Entity:CustomKubernetesEntity<UbuntuV1Alpha1Entity.EntitySpec, UbuntuV1Alpha1Entity.EntityStatus>
{
    public class EntitySpec
    {

    }

    public class EntityStatus
    {

    }
}
/*
apiVersion: example.local/v1alpha1
kind: Ubuntu
metadata:
  name: ubuntu-example
  namespace: default
  labels:
    example.local/version: v1alphal1
spec:
  ubuntuImage: latest
  ubuntuReplicas: 2
  ubuntuCommand: ['/bin/sleep', '3650d']
  favoriteFootballTeam: 'Buccaneers'
*/
```

Creating the entity spec is as easy as defining C# properties.  To add some documentation to these properties, there are some [validation attributes][kubeops-validation-documentation] which can be used to document and add validation to the entity.  This is all used to generate an OpenAPI schema.  Once complete, the `EntitySpec` class contains this:

``` cs
public class EntitySpec
{
    public string UbuntuImage { get; set; } = "latest";
    
    [RangeMinimum(Minimum = 1)]
    public int UbuntuReplicas { get; set; } = 1;

    [Description("A list of commands and command args to pass into the `cmd` property of the pod")]
    public List<string> UbuntuCommand { get; set; } = new List<string>();

    [Required]
    public string FavoriteFootballTeam { get; set; } = string.Empty;
}
```

Lastly, we need to decorate our Entity class to define our CRD API definition.  To do this, use the `KubernetesEntity` attribute and specify the Group, Version, and Kind parameters:

``` cs
[KubernetesEntity(Group = "example.local", ApiVersion = "v1alpha1", Kind = "Ubuntu")]
public class UbuntuEntity:CustomKubernetesEntity<UbuntuEntity.EntitySpec, UbuntuEntity.EntityStatus>
{
    # Class definition code removed
}
```

### Creating the Controller and Reconciler

Now that our CRD is created, we need a controller to listen for changes to that CRD, and we need a reconciler to create and manage the Kubernetes resources created for that entity.  First, let's create our reconciler since that's what the controller will call.

Our reconciler needs to be able to create and update a `deployment` resource and a `configmap` resource.  For organization purposes, I'm going to create separate methods within the reconciler for handling these resources.  We'll be using an `IKubernetesClient` to handle calling the Kubernetes API.

``` cs
public class UbuntuV1Alpha1Reconciler
{
    private readonly ILogger<UbuntuV1Alpha1Reconciler> _logger;
    private readonly IKubernetesClient _kubernetesClient;

    public UbuntuV1Alpha1Reconciler(ILogger<UbuntuV1Alpha1Reconciler> logger, IKubernetesClient kubernetesClient)
    {
        _logger = logger;
        _kubernetesClient = kubernetesClient;
    }

    public async Task ReconcileAsync(UbuntuV1Alpha1Entity entity)
    {
        await ReconcileConfigMap(entity);
        await ReconcileDeployment(entity);
    }
}
```

The code to do the actual resource reconciliation in this example is pretty simple - if the resource doesn't exist create it, and if the resource does exist update it.  I'm removing the code to create/update the actual resource objects, but this is the basic flow of what it looks like:

``` cs
private async Task ReconcileConfigMap(UbuntuV1Alpha1Entity entity)
{
    var @namespace = entity.Namespace();
    var entityName = entity.Name();

    _logger.LogInformation($"Fetching configmap for entity {entityName}");

    var configMap = await _kubernetesClient.Get<V1ConfigMap>(entityName, @namespace);

    if (configMap == null)
    {
        //ConfigMap resource doesn't exist, so create it...
        configMap = new V1ConfigMap(); //Fill in the appropriate parameters
        _kubernetesClient.Create(configMap);
    }
    else
    {
        //Update the ConfigMap object parameters that we already have
        await _kubernetesClient.Update(configMap);
    }
}
```

Now that the reconciler is created, let's create our Controller.  Our controller class will implement the `IResourceController` interface from the KubeOps library.  There are three overrideable methods within an IResourceController implementation:

* `ReconcileAsync` is called whenever an event occurs which triggers reconciliation for an entity
* `StatusUpdatedAsync` is called whenever a status change occurs for an entity
* `DeletedAsync` is called whenever an entity is deleted from the cluster

For this example, we will only implement the `ReconcileAsync` method since our CRD doesn't have any Status paramers and there's no additional logic that needs to occur on resource deletion.  The logic inside of `ReconcileAsync` is going to be very simple - just call the reconciler.

``` cs
public class UbuntuV1AlphaController:IResourceController<UbuntuV1Alpha1Entity>
{
    private readonly ILogger<UbuntuV1AlphaController> _logger;
    private readonly UbuntuV1Alpha1Reconciler _reconciler;

    public UbuntuV1AlphaController(ILogger<UbuntuV1AlphaController> logger, UbuntuV1Alpha1Reconciler reconciler)
    {
        _logger = logger;
        _reconciler = reconciler;
    }

    public async Task<ResourceControllerResult?> ReconcileAsync(UbuntuV1Alpha1Entity entity)
    {
        _logger.LogInformation($"{nameof(UbuntuV1AlphaController)}.{nameof(ReconcileAsync)} called for entity {entity.Name()} - calling reconciler.");
        await _reconciler.ReconcileAsync(entity);

        return null;
    }
}
```

### RBAC Setup

In order for our controller to be able to operate, it needs permissions to monitor the cluster for the CRDs as well as to create the necessary resources - in our case that's `ConfigMap` and `Deployment` resources.  To accomplish this, we need to add `EntityRbac` attributes to our controller, and we'll use one for each RBAC assignment we want to grant access to.

> **NOTE**: By default, KubeOps will generate a `ClusterRole` and `ClusterRoleBinding` for the `default` service account.  This can be changed by updating the generated yaml files.

``` cs
[EntityRbac(typeof(UbuntuV1Alpha1Entity), Verbs = RbacVerb.All)]
[EntityRbac(typeof(V1ConfigMap), Verbs = RbacVerb.All)]
[EntityRbac(typeof(V1Deployment), Verbs = RbacVerb.All)]
public class UbuntuV1AlphaController:IResourceController<UbuntuV1Alpha1Entity>
{
    //Class internals intentionally removed
}
```

### Creating the Finalizer

Our Controller and Reconciler will create Kubernetes objects when they're invoked, so there needs to be a way to "clean up" those objects when our CRD is deleted.  This will prevent abandoned resources from occupying the cluster.  This is handled by a *finalizer*.

``` cs
public class UbuntuV1Alpha1Finalizer : IResourceFinalizer<UbuntuV1Alpha1Entity>
{
    private readonly ILogger<UbuntuV1Alpha1Finalizer> _logger;
    private readonly IKubernetesClient _kubernetesClient;

    public UbuntuV1Alpha1Finalizer(ILogger<UbuntuV1Alpha1Finalizer> logger, IKubernetesClient kubernetesClient)
    {
        _logger = logger;
        _kubernetesClient = kubernetesClient;
    }

    public async Task FinalizeAsync(UbuntuV1Alpha1Entity entity)
    {
        await DeleteDeploymentAsync(entity);
        await DeleteConfigmapAsync(entity);
    }
}
```

In this example, the `DeleteDeploymentAsync` and `DeleteConfigmapAsync` methods just call the Kubernetes API to delete the created Deployment and Configmap resources, respectively.

To register our Entity to make use of our finalizer, we register it using the `IFinalizerManager` which is included in the KubeOps library.  This gets called from the `ReconcileAsync` method in the Controller.

``` cs
await _finalizerManager.RegisterFinalizerAsync<UbuntuV1Alpha1Finalizer>(entity);
```

### Setting up the Program.cs file

The last step in all of this is setting up the `Program.cs` file.  This uses the familiar builder pattern that is used by a typical AspNetCore application to configure dependency injection and set the "starting point" for the application.  The only difference is that it will call a couple of extension methods included in the KubeOps library.

``` cs
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddKubernetesOperator();
builder.Services.AddScoped<UbuntuV1Alpha1Reconciler>();

var app = builder.Build();
app.UseKubernetesOperator();

await app.RunOperatorAsync(args);
```

Since the Reconciler class we created is just essentially a "helper class", it has to be explicitly included in the dependency injection configuration.  The rest of the classes we've written implement an interface in the KubeOps library, and the `AddKubernetesOperator()` extension and the `RunOperatorAsync()` extension handle all of the routing and dependency injection for those classes.

### Generating the Kubernetes Assets

In order to build and package our operator, we need a Dockerfile.  Likewise, in order to deploy our operator workload and all relevant CRDs, Cluster Roles and workloads, we need to generate some kubeconfig files which contain the required resources.  The KubeOps library makes this very easy for us by generating those assets as part of the `dotnet build` process.  It does this by coming pre-packaged with a console utility which is then called with [a series of commands][kubeops-utility-commands] as part of the build process.  The default build parameters are generally adequate, however they can be customized [by setting some build property settings][kubeops-build-props-settings].  Additionally, any of the documented commands can be invoked outside of the build process by passing the command into a `dotnet run`.  For example, to generate the CRD objects without running a full build you could call `dotnet run generator crd`.

> **NOTE** There is a known issue currently with running a `dotnet build` command when your operator contains webhooks.  This is due to a requirement on the `csffl` library to generate certificates; if this library is not installed on your system it will fail to generate the operator kubeconfig file.  The operator can still be run locally, and the CRD kubeconfig files are still generated and can still be deployed.  This is handled in the Dockerfile so it will be successful with the `docker build` later.

### Testing the operator locally

Testing our operator locally requires three steps: deploy the CRDs, run the operator, and create a CRD object in our Kubernetes cluster.  The [KubeOps Command Utility][kubeops-utility-commands] we talked about before can handle installing our CRDs for us - the `dotnet run install` command deploys all of the CRD objects in the project.  This command does not install the Cluster Roles and Cluster Role bindings because it is intended to be used for testing purposes.

The second step is to run our operator.  I'm using Visual Studio Code, and I'm going to use the debugger within VSCode to do this.  The operator can also just be run using `dotnet run` in a console, but running it in the Visual Studio Code debugger gives the added benefit of being able to step through the operator functions if we need to debug.  Either way, what will happen is a console application will be launched which runs a Kestrel web server.

Lastly, we need to deploy an instance of the CRD.  In the case of our Ubuntu CRD we created, that looks like this:

``` bash
cat << EOF | kubectl apply -f -
apiVersion: example.local/v1alpha1
kind: Ubuntu
metadata:
  name: ubuntu-example
  namespace: default
  labels:
    example.local/version: v1alphal1
spec:
  ubuntuImageTag: latest
  ubuntuReplicas: 2
  ubuntuCommand: ['/bin/sleep', '3650d']
  favoriteFootballTeam: 'eagles'
EOF
```

After running that command, we'll see a series of information logs showing that the object was created and seen by the controller, and that the reconciler was then called and did its job.  To confirm that the operator called the reconciler and all resources were created, we can query the kubernetes cluster for pods and configmaps.  In the case above, we deployed two ubuntu pods (in a single deployment) and one configmap:

``` sh
kubectl get deployments
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
ubuntu-example   2/2     2            2           2m4s

kubectl get po       
NAME                              READY   STATUS    RESTARTS   AGE
ubuntu-example-6f7b8f5878-lxztc   1/1     Running   0          2m10s
ubuntu-example-6f7b8f5878-qf4z7   1/1     Running   0          2m10s

kubectl get configmap
NAME               DATA   AGE
ubuntu-example     1      2m13s
```

Periodically, you'll also see the messages in the logs for reconciliation of this entity without any event issued by the Kubernetes cluster.  These will be listed with an event type of "Reconcile".  This is normal, and helps to ensure the desired state of the system is enforced.  

When we delete our entity, you'll see that the Finalizer gets called first, followed by our Webhooks, followed then by the Controller.  The finalizer will clean up the relevant resources, and then the Controller will run whatever is in the `DeletedAsync` method if one is specified.  Once the finalizer and reconciler are finished running, you'll be able to query the cluster and see that the configmap, deployment, and two pods are deleted along with our CRD.

### Deploying the operator using Kustomize

Until this point, we haven't deployed the controller into the cluster.  We did deploy the CRD, but the controller was running on our local machine and monitoring the Kubernetes cluster for changes to the CRDs.  Now that we've debugged our controller and we see that it works, it's time to deploy it into our Kubernetes cluster.  This is once again made easy by using the KubeOps library.

If you paid attention to the build log output, you noticed these messages:

``` text
Generating Dockerfile
Generating CRDs
Generating Rbac roles
Generating Operator yamls
Generating Installer yamls
```

This is because on building a project with KubeOps, it automatically runs a series of [commands][kubeops-utility-commands] which generate a sample Dockerfile, CRD deployments, RBAC resources, and Operator deployment and installation files.  These files are all formatted as [Kustomize][kustomize-website] files, and they're a good starting point for being able to deploy our Operator into a Kubernetes cluster.

#### Building the Docker image

The first step is going to be building our Docker image and adding it to our image repository.  In this case, I'm going to use a local repository for the output of the `docker build` command, but if you're familiar with building and pushing Docker images to a remote repository this process should feel very familiar.  The other thing is I'm going to just use the Dockerfile which was output by the KubeOps build process, which is a pretty "normal" Dockerfile for a standard AspNetCore application.  All I have to do is run `docker build . -t 'ubuntu-operator:latest'` and let Docker do the rest of the work for me.  Once this process completes, I'll see a new image named "ubuntu-operator" with a tag of "latest" in my local image repository.

``` text
docker image ls
REPOSITORY          TAG          IMAGE ID       CREATED         SIZE
ubuntu-operator     latest       0e9ac2e5f588   4 minutes ago   312MB
...
```

#### Customize the Kustomize Files and Deploy

Before we can just deploy our Kustomize files, we have one modification to make.  The generated files assume that our docker image is going to be named "operator" but we built it under the name "ubuntu-operator".  When we try to run our Kustomize files to deploy our operator, this will cause an error when it tries to pull the image.  In the config/operator/deployment.yml file, find the `spec.template.spec.containers.image` property and change that to be the image name of the operator.  In our case, that's "ubuntu-operator:latest".  And with that done, we can open a shell in the config/install directory and run `kubectl apply -k .` to deploy the operator.  As a default, the install script will create a new namespace named {controller}-system and deploy the operator workloads into this namespace.  Viewing the logs on the operator pod will produce the same logs that we had while debugging locally, and creating an instance of the CRD like we did before will create an Ubuntu deployment and a configmap.  Likewise, deleting the CRD will delete the deployment and configmap.

To uninstall the operator, run `kubectl delete -k .` which will run the Kustomize templates and delete the relevant resources.

## Conclusions

The Kubernetes Operator pattern can simplify deploying complex workloads in a Kubernetes cluster.  They can be written in a variaty of popular programming languages, such as C#, with the only requirement being that they need to be able to run within a Kubernetes cluster.  They work by reading the state of the Kubernetes cluster and managing resources based on CRDs, other resources, and external services.  

[kubernetes-operator-pattern]: https://kubernetes.io/docs/concepts/extend-kubernetes/operator/
[crd]: https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/
[github-arc]: https://docs.github.com/en/actions/hosting-your-own-runners/managing-self-hosted-runners-with-actions-runner-controller/quickstart-for-actions-runner-controller
[pgsql-operator]: https://postgres-operator.readthedocs.io/en/latest/
[awx-operator]: https://github.com/ansible/awx-operator
[velero]: https://github.com/vmware-tanzu/velero
[kubernetes-admission-behavior]: https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/
[kubernetes-object-admission-phases]: https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#admission-control-phases
[writing-your-own-operator]: https://kubernetes.io/docs/concepts/extend-kubernetes/operator/#writing-operator
[kubeops]: https://github.com/buehler/dotnet-operator-sdk/tree/master
[kubernetesclient-csharp]: https://github.com/kubernetes-client/csharp
[cncf-operator-pattern-diagram]: https://www.cncf.io/wp-content/uploads/2022/07/k8s-operator-1800x1013.webp
[kubeops-validation-documentation]: https://github.com/buehler/dotnet-operator-sdk/tree/master/src/KubeOps#validation
[kubeops-build-props-settings]: https://github.com/buehler/dotnet-operator-sdk/blob/master/src/KubeOps/README.md#ms-build-extensions
[kubeops-utility-commands]: https://github.com/buehler/dotnet-operator-sdk/blob/master/src/KubeOps/README.md#commands
[kustomize-website]: https://kustomize.io/