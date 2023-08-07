---
layout: post
title: My Thoughts On The GitHub Actions Runner Controller
date: 2023-08-07 10:00:00 -0400
description: 'GitHub Actions can be run using one of two options: Self-Hosted Runners or GitHub-Hosted Runners.  Traditionally, Self-Hosted runners are hosted on dedicated VM's, but GitHub is developing a Kubernetes Operator Pattern workload called the GitHub Actions Runner Controller.  In this post, I'll explore the process of deploying, customizing, and using the Actions Runner Controller and give my recommendations on when to use it and not to use it.'
img: github-actions-runner-controller/hero.png
fig-caption: GitHub Actions Runner Controller Logo
tags: [GitHub Actions, GitHub, DevOps]
---

> **NOTE**
> At the time of writing, GitHub Actions Runner controller is still in beta, so I would not consider it production-ready.  

The [GitHub Actions Runner Controller (ARC)](https://github.com/actions/actions-runner-controller/blob/master/docs/about-arc.md) is a Kubernetes-based solution to hosting the GitHub Actions Self-Hosted Runners within a containerized environment.  It follows the [Kubernertes Operator pattern](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/), which essentially consists of an "operator" or controller workload which then deploys job pods on an as-needed basis within the Kubernetes cluster and is configured based on a set of [Custom Resource Definitions (CRDs)](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/).  Essentially what happens is there's an "Operator" or controller running which is polling the cluster for updates to a set of CRDs, and if it sees updates to these CRDs it'll perform some sort of operation.  For example, if the Operator sees a new `RunnerDeployment` get created, it'll automatically deploy a new `Runner` resource and a new Pod which will host this runner.

![GitHub ARC Process Diagram](https://user-images.githubusercontent.com/53718047/183928236-ddf72c15-1d11-4304-ad6f-0a0ff251ca55.jpg)

By using ARC, it becomes easier to:

* Deploy self-hosted runners on a Kubernetes cluster with a simple, repeatable set of commands
* Auto-scale runners based on demand
* Setup across GitHub editions (including GitHub Enterprise and GitHub Enterprise Cloud)

## tl;dr

This blog post is a bit long.  There's a lot of details around GitHub ARC that I wanted to cover from setting up a very basic lab environment, enabling scaling, and some observations I've made while testing it out.  Because of the length of this post, I'm going to lead off with my recommendations and conclusions.

When able, I would recommend using either GitHub-Hosted Runners or GitHub ARC.  VM-based runners should be a "last-resort" in my opinion.  The way a VM-based runner works is different enough from a GitHub-Hosted runner that it can lead to inconsistencies between each mechanism.  GitHub ARC is designed to closely mimic GitHub-Hosted runners in terms of their ephemeral nature, pre-loaded software (it's not an exact match, but it's closer than what you get with a self-hosted runner), and scalability (being able to scale from 0 and spin up and down runners on an as-needed basis).  Of course, I understand that there are circumstances where GitHub ARC would not work for a solution, and in those cases VM-based runners are a good solution.  But if you already have a Kubernetes cluster that has some processing power to spare I would recommend using GitHub ARC.

## Setting up ARC

ARC is designed to run on a Kubernetes cluster.  I'm going to host it on Azure Kubernetes Service (AKS).  If you don't have a cluster that you can deploy into, it can also be deployed in a local environment running [minikube](https://minikube.sigs.k8s.io/docs/start/).  Another prerequisite of ARC is [cert-manager](https://cert-manager.io/docs/installation/).  Cert-Manager can be deployed easily with a single `kubectl apply` command:

``` sh
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.8.2/cert-manager.yaml
```

Another thing you'll need is a Personal Access Token (PAT) for ARC to authenticate to GitHub.   To generate a PAT token, log in to your GitHub Account and navigate to "Create New Token".  You'll need to give it all "repo" access.  Store this token value in a safe place for now - we'll need it to configure ARC.

> **NOTE**:
> You can set up GitHub ARC using a GitHub App instead of a PAT token.  Those instructions can be found [here](https://github.com/actions/actions-runner-controller/blob/master/docs/authenticating-to-the-github-api.md).  Since I was just setting up a test bench in a private repository in my personal organization, I used a PAT token.

There's a lot of other options that can be configured from the helm chart.  Those options can be found [here](https://github.com/actions/actions-runner-controller/blob/master/charts/actions-runner-controller/README.md).  For example, you could configure the controller to watch a single namespace for changes to the CRD objects or set pod disruption budgets on the controller pods for HA.  Since I don't need to do too much customization, I'm going to omit those options from this post.

Now we're ready to actually deploy ARC.  I'm going to use Helm to do this, but it can be done using either Helm or Kubectl commands.  First we need to add the actions-runner-controller repo to our local Helm repos list:

``` sh
helm repo add actions-runner-controller https://actions-runner-controller.github.io/actions-runner-controller
```

And now we can deploy the Helm workload into our cluster.  Be sure to replace the "REPLACE_YOUR_TOKEN_HERE" placeholder with your PAT token!

``` sh
helm upgrade --install --namespace actions-runner-system --create-namespace\
  --set=authSecret.create=true\
  --set=authSecret.github_token="REPLACE_YOUR_TOKEN_HERE"\
  --wait actions-runner-controller actions-runner-controller/actions-runner-controller
```

Once that completes, it's time to create the runners.  There's two main ways to do this: `RunnerDeployment` and `RunnerSet`.  A `RunnerDeployment` is a model based off of a [Kubernetes `Deployment`](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) model, whereas a `RunnerSet` is based off of a [Kubernetes `StatefulSet`](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/) model.  These two APIs have some main differences, but act in similar ways:

* Deployments are used for stateless applications and StatefulSets should be used for stateful applications
* Pods in a deployment are interchangeable whereas pods in a StatefulSet are not
* Deployments require a service to enable interaction with pods, while a headless service handles the pods' network ID in StatefulSets
* In a deployment, the replicas all share a single volume and PVC.  In a StatefulSet, each pod has its own volume and PVC

For this demo, I'm going to use a `RunnerSet`.  A basic `RunnerSet` deployment looks like this:

``` yaml
apiVersion: actions.summerwind.dev/v1alpha1
kind: RunnerSet
metadata:
  name: my-runner
  namespace: github-arc-runner  # As a best practice, it is advised to have this different from where you deployed your operator
spec:
  replicas: 1
  repository: your-repository-here  # Replace this with the name of your repository.
  labels:
    - github-arc
  # Other mandatory fields from StatefulSet
  selector:
    matchLabels:
      app: example
  serviceName: example
  template:
    metadata:
      labels:
        app: example
```

Once this new `RunnerSet` CRD is deployed, the operator that we deployed earlier should see this get added and automatically create some runners.

![GitHub Runner Created!]({{site.baseurl}}/assets/img/github-actions-runner-controller/runners-created-1.png)

Additionally, we can validate that the runner was created successfully by running a couple of kubectl commands:

``` sh
$ kubectl get po -n github-arc-runner
NAME                READY   STATUS    RESTARTS   AGE
my-runner-2jh6q-0   2/2     Running   0          49s

$ kubectl get runnerset -n github-arc-runner
NAME        DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
my-runner   1         1         1            1           34s
```

Great!  We now have a self-hosted runner running in our repository.  Let me emphasize - we have **ONE** runner running.  While that'll work for this example, it probably won't work for most organizations and enterprises.  ARC can be scaled manually by setting the `replicas` in the `RunnerSet` and can also be [automatically scaled](https://github.com/actions/actions-runner-controller/blob/master/docs/automatically-scaling-runners.md).  This includes the ability to [autoscale to/from 0](https://github.com/actions/actions-runner-controller/blob/master/docs/automatically-scaling-runners.md#autoscaling-tofrom-0).  We will use pull-driven scaling for this example, and update our previous `RunnerSet` deployment to now include a `HorizontalRunnerAutoscaler` CRD:

``` yaml
# RunnerSet elided

---
apiVersion: actions.summerwind.dev/v1alpha1
kind: HorizontalRunnerAutoscaler
metadata:
  name: my-runner-autoscaler
spec:
  scaleTargetRef:
    kind: RunnerSet
    name: my-runner # This should match the name of your RunnerSet or RunnerDeployment
  minReplicas: 0
  maxReplicas: 10
  metrics:
  - type: TotalNumberOfQueuedAndInProgressWorkflowRuns
    repositoryNames:
    - your-repository-here  # Replace this with the name of your repository.
```

This will cause auto-scaling between 0 and 10 replicas, depending on the number of queued jobs.  This configuration uses [Pull Driven Scaling](https://github.com/actions/actions-runner-controller/blob/master/docs/automatically-scaling-runners.md#pull-driven-scaling), which has a default of polling every 1 minute.  This can be adjusted, but there are risks around having too fast of a polling rate.  For this example, I'm leaving it as the default and dealing with the "first-hit" penalty that comes along with that.  Ideally, this would be using Webhook-based scaling, where GitHub actually sends a webook event to the operator to control spinning up a new runner.

We now see these deployed resources/CRDs in the runner namespace (note: I have a custom function that pulls resources of all API types.  This is the output of that script.):

``` text
Resource: horizontalrunnerautoscalers.actions.summerwind.dev
NAME                   MIN   MAX   DESIRED   SCHEDULE
my-runner-autoscaler   0     10    0         

Resource: runnersets.actions.summerwind.dev
NAME        DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
my-runner   0         0         0            0           90m
```

Additionally, there are now no longer any running pods in the cluster because we've told the autoscaler to scale down to 0 when nothing is required.

## Using The Self-Hosted Runner

Using a self-hosted runner deployed by GitHub ARC is the same as any other self-hosted runner.  In the workflow's `runs-on` clause, we can specify an array containing "self-hosted" and other labels that point to our self-hosted ARC runners.  For example, the one deployed above has an extra label of "github-arc" so I could specify `runs-on: [self-hosted, Linux, X64, github-arc]` to target the ARC runners.  For example:

``` yml
jobs:
  Explore-GitHub-Actions:
    runs-on: [self-hosted, Linux, X64, github-arc]
    steps:
      # Job steps elided 
```

## Differences I Noticed Between ARC and VM-Based Self-Hosted Runners

There are a couple of key differences I've found between the ARC runner and the VM-based runners:

1. GitHub ARC runners are ephemeral, whereas VM-based runners are not.  This means that every time you run a new workflow using an ARC-based runner, you get a totally new runner instance; the pod exits and restarts after each run.  A VM-based runner **does not** have a totally clean environment every time a workflow run occurs.  If you need proof of that, just navigate to the work folder (this defaults to `_work`) and you can find the contents of whatever repository you ran a workflow against, along with a cache of every action that was used in that workflow.  This sort of behavior is much different than what a GitHub-hosted runner provides - a GitHub-hosted runner *always* starts with a clean instance.
    *ARC-Based Runner*

    ``` text
    Run actions/checkout@v3
      Syncing repository: molson504x/github-arc
      Getting Git version info
      Temporarily overriding HOME='/runner/_work/_temp/c3aa470d-abb7-4d97-9366-bd5a059b0ca8' before making global git config changes
      Adding repository directory to the temporary git global config as a safe directory
      /usr/bin/git config --global --add safe.directory /runner/_work/github-arc/github-arc
      Deleting the contents of '/runner/_work/github-arc/github-arc'
      Initializing the repository
      Disabling automatic garbage collection
      Setting up auth
      Fetching the repository
      Determining the checkout info
      Checking out the ref
      /usr/bin/git log -1 --format='%H'
      '64b0a3f000220aad441a137c43d3b8c65fb08ed2'
    ```

    *VM-Based Runner*

    ``` text
    Run actions/checkout@v3
      Syncing repository: molson504x/github-arc
      Getting Git version info
      Temporarily overriding HOME='/home/azureuser/actions-runner/_work/_temp/4e1ce71b-df29-4f4a-87b5-70596422bb7b' before making global git config changes
      Adding repository directory to the temporary git global config as a safe directory
      /usr/bin/git config --global --add safe.directory /home/azureuser/actions-runner/_work/github-arc/github-arc
      /usr/bin/git config --local --get remote.origin.url
      https://github.com/molson504x/github-arc
      Removing previously created refs, to avoid conflicts
      /usr/bin/git submodule status
      Cleaning the repository
      Disabling automatic garbage collection
      Setting up auth
      Fetching the repository
      Determining the checkout info
      Checking out the ref
      /usr/bin/git log -1 --format='%H'
      '64b0a3f000220aad441a137c43d3b8c65fb08ed2'
    ```
  
    Looking at the logs, in the ARC-based runner we see a line in there, `Deleting the contents of '/runner/_work/github-arc/github-arc'`.  This is due to that directory being automatically created and the default of the `checkout` action's `clean` input being set to `true` which triggers this behavior.  The directory is actually empty before the `checkout` step runs.  This can be verified by adding a step that checks for the existance of the directory and runs a `ls` command:
    *ARC-Based Runner*

    ``` text
    Run if [ -d "/runner/_work/github-arc/github-arc" ]; then
      if [ -d "/runner/_work/github-arc/github-arc" ]; then
        echo "Directory '/runner/_work/github-arc/github-arc' already exists."
        echo $(ls)
        ls
      else
        echo "Directory '/runner/_work/github-arc/github-arc' does not exist."
      fi
      shell: /usr/bin/bash -e {0}
    Directory '/runner/_work/github-arc/github-arc' already exists.
    ```

    *VM-Based Runner*

    ``` text
    Run if [ -d "/home/azureuser/actions-runner/_work/github-arc/github-arc" ]; then
      if [ -d "/home/azureuser/actions-runner/_work/github-arc/github-arc" ]; then
        echo "Directory '/home/azureuser/actions-runner/_work/github-arc/github-arc' already exists."
        echo $(ls)
        ls
      else
        echo "Directory '/home/azureuser/actions-runner/_work/github-arc/github-arc' does not exist."
      fi
      shell: /usr/bin/bash -e {0}
    Directory '/home/azureuser/actions-runner/_work/github-arc/github-arc' already exists.
    README.md
    README.md
    ```

1. Self-hosted runners allow you to manually installing dependencies, whereas GitHub-hosted runners recommend running "setup" actions.  This is because you don't have access to the underlying infrastructure on a GitHub-hosted runner.  GitHub ARC runners work in a similar manner to the GitHub-hosted runners - the recommendation is to use a "setup" action (such as `actions/setup-node`) to install dependencies.  This is due to the ephemeral nature of ARC runners as opposed to the non-ephemeral nature of VM-based runners.  As a test, I set up a VM-based runner and an ARC-based runner and ran a simple workflow twice that installs Node 16.  While it did not fail, there is a clear difference in the output logs:
   *ARC-Based Runner*

    ``` text
    Run actions/setup-node@v3
    Attempting to download 16.x...
    Acquiring 16.20.1 - x64 from https://github.com/actions/node-versions/releases/download/16.20.1-5342959204/node-16.20.1-linux-x64.tar.gz
    Extracting ...
    /usr/bin/tar xz --strip 1 --warning=no-unknown-keyword -C /runner/_work/_temp/bb0680fd-7dd8-4743-ace7-2d67bac95b4d -f /runner/_work/_temp/fecfce4a-261d-461b-b10b-74e8287cdd67
    Adding to the cache ...
    Environment details
      node: v16.20.1
      npm: 8.19.4
      yarn: 
    ```

    *VM-Based Runner*

    ``` text
    Run actions/setup-node@v3
    Found in cache @ /home/azureuser/actions-runner/_work/_tool/node/16.20.1/x64
    Environment details
      node: v16.20.1
      npm: 8.19.4
      yarn: 
    ```

    As you can see, there's 2 major things here: the VM cached the installation for installing Node 16, and it didn't actually have to do it the 2nd time.  I actually found a similar behavior on the GitHub-hosted runner for the setup-node action, but that's because Node comes pre-installed on the GitHub-hosted runners.

    > **NOTE**:
    > You *can* customize the runner image used by ARC, and this customization can be used to manually install dependencies.  For more information on that, see [this documentation](https://github.com/actions/actions-runner-controller/blob/master/docs/about-arc.md#software-installed-in-the-runner-image).

## Conclusions

GitHub ARC is a good solution for hosting GitHub runners within a Kubernetes cluster.  It mimics how GitHub-hosted runners operate normally while allowing for the option of a self-hosted solution.  Additionally, it is easy to set up and configure, and seems to follow the same documentations that GitHub-hosted runners follow.  Its ephemeral nature makes it a close match with GitHub-hosted runners, as opposed to the non-ephemeral nature of VM-based runners.