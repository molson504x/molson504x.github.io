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

Kubernetes operators can be written in a [variety of languages][writing-your-own-operator].  Since I'm familiar with C\#, I decided to write my sample operator in C\#.  The code samples in the remainder of this document will also be in C\#.  The premise of this operator is similar to the premise of any other operator - a controller will be monitoring for changes to a CRD Object, and will act based on the state change of that object.  In the case of this example, I'll be creating a deployment of [OWASP Mutillidae II][owasp-mutillidae], which is a vulnerable web application used as a lab environment for pen-testing.  I chose this because it is a workload which consists of creating 5 deployments and 5 services.

### KubeOps NuGet Package

I'll be using the [KubeOps][kubeops] library for authoring my Operator in these examples.  This library is listed in the [official Kubernetes documentation][writing-your-own-operator] which is why I picked it.  Behind the scenes it is making use of the [KubernetesClient CSharp][kubernetesclient-csharp] library.  The KubeOps library extends the base AspNetCore project type and relies on the included DI framework and project patterns, so if you are familiar with this type of project these concepts should feel familiar.

We will first create our solution and add the KubeOps NuGet package.  For this example, I'm going to use the `web` template which creates an empty baseline AspNetCore project.

``` bash
dotnet new web -n YourSolutionNameHere
dotnet install KubeOps
```

### Setting up the Program.cs file

### Defining your CRD as a C# Object

### Creating the Controller

### Testing the operator locally

### Deploying the operator using Kustomize

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
[owasp-mutillidae]: https://owasp.org/www-project-mutillidae-ii/
[kubeops]: https://github.com/buehler/dotnet-operator-sdk/tree/master
[kubernetesclient-csharp]: https://github.com/kubernetes-client/csharp
[cncf-operator-pattern-diagram]: https://www.cncf.io/wp-content/uploads/2022/07/k8s-operator-1800x1013.webp