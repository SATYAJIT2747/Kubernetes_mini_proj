# Kubernetes Mini Project --- Complete Revision Guide

> **Goal:** Understand the Kubernetes concepts used in this mini project
> before reading the repository code. This guide uses simple language,
> keeps the important technical terms, and connects each concept to the
> Flask application.

------------------------------------------------------------------------

## Table of Contents

1.  [Project at a Glance](#1-project-at-a-glance)
2.  [What Is Kubernetes?](#2-what-is-kubernetes)
3.  [Why Do We Need Kubernetes?](#3-why-do-we-need-kubernetes)
4.  [Kubernetes Architecture](#4-kubernetes-architecture)
5.  [Kubernetes Objects and Desired
    State](#5-kubernetes-objects-and-desired-state)
6.  [Pods](#6-pods)
7.  [Labels and Selectors](#7-labels-and-selectors)
8.  [ReplicaSets](#8-replicasets)
9.  [Deployments](#9-deployments)
10. [Services and Networking](#10-services-and-networking)
11. [Understanding Kubernetes YAML](#11-understanding-kubernetes-yaml)
12. [Useful `kubectl` Commands](#12-useful-kubectl-commands)
13. [Minikube](#13-minikube)
14. [Docker and Kubernetes Together](#14-docker-and-kubernetes-together)
15. [Understanding the Project
    Files](#15-understanding-the-project-files)
16. [End-to-End Execution Flow](#16-end-to-end-execution-flow)
17. [Troubleshooting Common
    Problems](#17-troubleshooting-common-problems)
18. [Quick Revision Sheet](#18-quick-revision-sheet)
19. [Self-Test Questions](#19-self-test-questions)

------------------------------------------------------------------------

## 1. Project at a Glance

This mini project runs a **Flask web application** inside a container
and uses Kubernetes to manage that application.

At a high level:

``` text
Browser
   |
   | HTTP request
   v
Kubernetes Service
   |
   | forwards traffic to a matching Pod
   v
+------------------+     +------------------+
| Pod 1            |     | Pod 2            |
| Flask container  |     | Flask container  |
+------------------+     +------------------+
          ^                       ^
          |                       |
          +------ ReplicaSet -----+
                     ^
                     |
                 Deployment
```

The objects have different jobs:

  -----------------------------------------------------------------------
  Component               Simple meaning          Main responsibility
  ----------------------- ----------------------- -----------------------
  Flask                   The application         Handles requests and
                                                  creates responses

  Docker image            Application package     Contains the app and
                                                  the environment it
                                                  needs

  Container               Running application     Executes the Flask
                          process                 application

  Pod                     Kubernetes unit that    Provides the
                          runs containers         environment in which a
                                                  container runs

  ReplicaSet              Pod-count manager       Keeps the requested
                                                  number of matching Pods
                                                  running

  Deployment              Application release     Manages ReplicaSets,
                          manager                 updates, scaling, and
                                                  rollbacks

  Service                 Stable network entry    Sends traffic to the
                          point                   appropriate Pods

  Minikube                Local Kubernetes        Lets you practise
                          environment             Kubernetes on your own
                                                  machine

  `kubectl`               Kubernetes command-line Sends commands to the
                          tool                    cluster's API
  -----------------------------------------------------------------------

**Remember the main idea:** Docker packages the application; Kubernetes
manages its deployment and operation.

------------------------------------------------------------------------

## 2. What Is Kubernetes?

**Kubernetes (often shortened to K8s)** is an open-source container
orchestration platform. *Container orchestration* means automatically
managing containers across one or more machines.

Kubernetes can help with:

-   **Deployment:** Running an application in the cluster.
-   **Scaling:** Increasing or decreasing the number of application
    replicas.
-   **Self-healing:** Replacing Pods that fail or disappear.
-   **Service discovery:** Letting applications find one another through
    stable names.
-   **Load balancing:** Distributing traffic across available
    application replicas.
-   **Rolling updates:** Replacing an old application version gradually.
-   **Rollbacks:** Returning to a previous Deployment revision.
-   **Scheduling:** Choosing which worker machine should run each Pod.

### Docker versus Kubernetes

Docker and Kubernetes are related, but they solve different problems.

  -----------------------------------------------------------------------
  Docker / container tools            Kubernetes
  ----------------------------------- -----------------------------------
  Build an image from a Dockerfile    Decide how many application
                                      replicas should run

  Package application code and        Schedule Pods on cluster nodes
  dependencies                        

  Run containers                      Manage Pods and the workloads
                                      inside them

  Useful for running an app on one    Useful for coordinating
  machine                             applications across a cluster

  Does not by itself provide          Continuously works to maintain the
  Kubernetes-style Deployment         declared state
  reconciliation                      
  -----------------------------------------------------------------------

A container runtime such as **containerd** or **CRI-O** actually starts
containers on a Kubernetes node. Docker is commonly used to build
images, but Kubernetes does not require the Docker Engine as its
container runtime.

------------------------------------------------------------------------

## 3. Why Do We Need Kubernetes?

Imagine that the Flask application becomes popular. More users visit it,
one container crashes, or a new version must be released. Managing every
container manually becomes difficult.

### 3.1 Scaling

**Scaling** means changing how many copies of an application run.

For example, the Deployment asks for two replicas:

``` text
Desired replicas: 2
Current Pods:     2
```

If traffic increases, you can request five replicas:

``` text
Desired replicas: 5
Current Pods:     2
```

Kubernetes creates three additional Pods to move toward the requested
state. This is called **horizontal scaling** because more instances run
side by side.

> Kubernetes does not automatically increase replicas just because CPU
> usage rises unless you configure an autoscaling mechanism, such as a
> Horizontal Pod Autoscaler. Changing `replicas` manually is scaling,
> but it is not automatic traffic-based autoscaling.

### 3.2 Self-healing

Suppose a Deployment should have three Pods, but one Pod is deleted or
fails.

``` text
Before:  Pod 1   Pod 2   Pod 3
Failure: Pod 1   Pod 2   [missing]
Repair:  Pod 1   Pod 2   Pod 4
```

A controller notices that the actual number of matching Pods is below
the desired number and works to create a replacement.

**Important distinction:** Kubernetes can replace a failed Pod, but it
cannot guarantee that the application code itself is bug-free. If every
replacement crashes for the same reason, the problem must still be
debugged.

### 3.3 Load balancing

If several Pods run the same application, a Service can distribute
incoming connections across eligible Pods.

``` text
             Service
            /   |   \
         Pod 1 Pod 2 Pod 3
```

This avoids requiring the client to know each Pod's individual IP
address. The exact traffic distribution depends on the networking
implementation and connection behaviour; it is not necessarily an equal
split for every request.

### 3.4 Service discovery

A Pod's IP address can change when that Pod is replaced. A client that
relies on a Pod's old IP may stop working.

A Kubernetes **Service** provides a stable virtual IP and DNS name for a
group of matching Pods, so clients can use the Service rather than
tracking Pod IPs themselves.

### 3.5 Rolling updates

A **rolling update** gradually replaces old application Pods with new
ones instead of deliberately stopping all old Pods at once.

``` text
Old version running
        |
        v
New version Pods are created
        |
        v
Old version Pods are gradually removed
        |
        v
New version running
```

This can reduce downtime, but **zero downtime is not guaranteed
automatically**. It depends on settings such as readiness probes, the
rollout strategy, application compatibility, and sufficient available
capacity.

### 3.6 Rollback

A **rollback** returns a Deployment to an earlier revision when a newer
release is faulty.

``` text
Revision 1 (working)
        |
        v
Revision 2 (problem)
        |
        v
Rollback to Revision 1
```

Kubernetes retains Deployment revision information, subject to its
revision-history settings. A rollback cannot fix a bug that also exists
in the earlier version.

### 3.7 High availability

**High availability** means designing a system to remain usable when
some components fail. Running multiple replicas can help an application
tolerate an individual Pod failure.

However, replicas alone do not guarantee full high availability. Node
failures, storage, network design, capacity, and the application's own
behaviour also matter.

### 3.8 Scheduling

**Scheduling** is the process of choosing a worker node for a newly
created Pod. The scheduler considers resource requests, node
availability, constraints, and other scheduling rules. It does not
normally choose a node simply because that node has the most free CPU.

------------------------------------------------------------------------

## 4. Kubernetes Architecture

A **Kubernetes cluster** is a group of machines that work together to
run applications. These machines may be virtual machines or physical
machines.

A cluster has two broad parts:

``` text
Kubernetes Cluster
|
+-- Control Plane: makes and coordinates decisions
|   |
|   +-- kube-apiserver
|   +-- etcd
|   +-- kube-scheduler
|   +-- kube-controller-manager
|
+-- Worker Node(s): run application workloads
    |
    +-- kubelet
    +-- container runtime
    +-- networking components
    +-- Pods
```

In a small Minikube setup, the control-plane and workload roles may run
on the same node. In a production cluster, they are often separated.

### 4.1 Control Plane

The **Control Plane** is the management layer of Kubernetes. It receives
requests, records cluster information, decides where workloads should
run, and coordinates controllers that work to maintain the requested
state.

#### `kube-apiserver` --- the API entry point

The **API Server** exposes the Kubernetes API. Tools such as `kubectl`
communicate with the cluster through this API.

For example:

``` bash
kubectl get pods
kubectl apply -f deployment.yaml
kubectl delete -f deployment.yaml
```

Conceptually, these commands send requests to the API Server. The API
Server validates and processes them and coordinates access to the
cluster's stored objects.

#### `etcd` --- the cluster's key-value store

**etcd** is a distributed key-value database used by Kubernetes to store
cluster state and configuration data.

It stores information about Kubernetes objects, such as Deployments,
Pods, Services, and other resources. It is not the place where your
Flask application's ordinary user data automatically goes.

Think of it as the cluster's reliable record of its declared and
observed configuration.

#### `kube-scheduler` --- chooses a node

The **Scheduler** finds newly created Pods that do not yet have a node
assigned and selects a suitable worker node.

It considers factors such as:

-   CPU and memory resource requests.
-   Whether a node is available and eligible.
-   Scheduling constraints, affinity rules, and taints or tolerations
    when configured.

The scheduler selects a node; the kubelet on that node is responsible
for making the Pod run.

#### `kube-controller-manager` --- runs controllers

The **Controller Manager** runs controllers that repeatedly compare
desired state with observed state and take action when they differ.

For example, a Deployment and its ReplicaSet work together to keep the
requested number of application replicas running.

### 4.2 Worker Node

A **Worker Node** is a machine that runs application workloads.

#### `kubelet` --- node-level agent

The **kubelet** runs on each node and communicates with the API Server.
It makes sure that the Pods assigned to that node are running as
expected, asks the container runtime to start containers, and reports
status.

The kubelet does not independently decide the overall desired replica
count; controllers manage that higher-level state.

#### Container runtime --- starts containers

The **container runtime** downloads or accesses images and starts and
stops containers. Common Kubernetes-compatible runtimes include:

-   `containerd`
-   `CRI-O`

The runtime works with the kubelet through the Container Runtime
Interface (CRI).

#### `kube-proxy` --- Service networking (where used)

**kube-proxy** helps implement Kubernetes Service networking on many
cluster setups by programming network rules that direct traffic to
Service endpoints.

Some networking implementations replace or do not use kube-proxy, so it
is not guaranteed to be present in every cluster. It is also not the
same thing as the application container or the Service object itself.

### 4.3 The main request path

A simplified explanation of applying a Deployment is:

1.  You run `kubectl apply -f deployment.yaml`.
2.  `kubectl` sends the requested object to the API Server.
3.  Kubernetes validates and stores the relevant object state.
4.  Controllers notice the Deployment and work to create the required
    ReplicaSet and Pods.
5.  The scheduler assigns eligible new Pods to nodes.
6.  The kubelet on each selected node asks the container runtime to
    start the containers.
7.  Kubernetes reports status as the workload progresses.

This is a simplified logical flow, not a claim that every component runs
in one strictly sequential chain.

------------------------------------------------------------------------

## 5. Kubernetes Objects and Desired State

A **Kubernetes object** is a record describing a resource that
Kubernetes should manage. Common objects include:

-   **Pod:** Runs one or more containers.
-   **ReplicaSet:** Maintains a specified number of matching Pod
    replicas.
-   **Deployment:** Manages ReplicaSets and application rollouts.
-   **Service:** Provides stable network access to matching Pods.
-   **ConfigMap:** Stores non-sensitive configuration data.
-   **Secret:** Stores sensitive configuration values, such as
    credentials. Secrets still require appropriate access controls and
    secure cluster configuration.

### 5.1 Desired state versus actual state

**Desired state** is what you ask Kubernetes to maintain.

**Actual state** is what is currently happening in the cluster.

Example:

``` text
Desired state: 3 replicas
Actual state:  2 running Pods
Difference:    1 replica missing
Action:        Kubernetes works to create another Pod
```

### 5.2 Reconciliation loop

A **reconciliation loop** is a repeated process in which a controller
checks the actual state against the desired state and takes corrective
action.

``` text
Read desired state
        |
        v
Observe actual state
        |
        v
Compare the two
        |
        v
Take corrective action if needed
        |
        +------> Repeat
```

This is the foundation of Kubernetes' declarative management and much of
its self-healing behaviour.

**Declarative configuration** means describing the result you want, such
as "run three replicas," rather than writing every individual step
required to create and repair them.

### 5.3 Object hierarchy in this project

``` text
Deployment
    |
    | manages
    v
ReplicaSet
    |
    | creates and maintains
    v
Pods
    |
    | contain
    v
Containers
```

A Service is a separate object. It selects eligible Pods using labels;
it is not a parent of the Pods.

``` text
Service ---- selects Pods using labels ----> Pods
```

------------------------------------------------------------------------

## 6. Pods

A **Pod** is the smallest deployable unit in Kubernetes. Kubernetes
schedules Pods onto nodes; a Pod contains one or more containers.

For this project, a typical Pod contains one Flask container:

``` text
Pod
|
+-- Flask container
    |
    +-- Flask application process
```

### 6.1 Why does Kubernetes use Pods?

A Pod gives its containers a shared execution environment. Containers in
the same Pod can generally:

-   Share the Pod's network namespace and IP address.
-   Communicate through `localhost`.
-   Share configured storage volumes.
-   Be managed as one unit for scheduling.

Most simple web applications use one main application container per Pod.
Multiple containers in one Pod are useful when the containers need to
work closely together, such as an application container and a helper or
sidecar container.

### 6.2 Pod IP addresses

Each Pod normally has its own IP address within the cluster network. If
a Pod is replaced, the new Pod may receive a different IP address. This
is one reason applications usually communicate through Services rather
than hard-coding Pod IPs.

### 6.3 Pod lifecycle and status

You may see these Pod phases or displayed statuses:

-   **Pending:** Kubernetes has accepted the Pod, but it is not yet
    fully running. It may be waiting for scheduling, image download, or
    setup.
-   **Running:** The Pod has been bound to a node and at least one
    container is running or starting/restarting.
-   **Succeeded:** All containers in the Pod have completed successfully
    and will not be restarted.
-   **Failed:** All containers have terminated, and at least one
    terminated unsuccessfully.
-   **Unknown:** Kubernetes cannot reliably determine the Pod's state.

`ContainerCreating` is commonly shown by `kubectl` as a container
state/reason while setup is in progress; it is not one of the official
Pod phases.

### 6.4 Container ports

A Pod specification may declare a container port, such as `5000` for
Flask. This documents the port the application expects to use and can
help other configuration refer to it. Declaring a port does **not**, by
itself, expose the application outside the Pod or cluster.

------------------------------------------------------------------------

## 7. Labels and Selectors

### 7.1 Labels

A **label** is a key-value pair attached to a Kubernetes object. Labels
are used to organize objects and identify groups of resources.

Example:

``` yaml
labels:
  app: flask-demo
  environment: learning
```

Here, `app` and `environment` are label keys, and `flask-demo` and
`learning` are their values.

Labels are metadata; they do not automatically configure networking or
change how the application runs.

### 7.2 Selectors

A **selector** specifies which objects match certain labels.

For example:

``` yaml
selector:
  matchLabels:
    app: flask-demo
```

This selector matches Pods carrying the label `app: flask-demo`.

### 7.3 Why are labels and selectors important?

A Deployment uses a selector to identify the Pods managed by its
ReplicaSets. A Service uses a selector to find the Pods to which it can
send traffic.

``` text
Pod 1: app=flask-demo  <---+
Pod 2: app=flask-demo  <---+--- Service selector: app=flask-demo
Pod 3: app=other-app        (not selected)
```

For a Deployment, its selector must match the labels in its Pod
template. If these labels do not match, the Deployment configuration may
be rejected or fail to manage the intended Pods.

**Easy way to remember:** - **Label:** the tag attached to an object. -
**Selector:** the rule used to find objects with particular tags.

------------------------------------------------------------------------

## 8. ReplicaSets

A **ReplicaSet** makes sure that a specified number of matching Pod
replicas exist.

Suppose its desired replica count is three:

``` text
Desired: 3 Pods
Actual:  2 Pods
```

The ReplicaSet works to create another Pod. If four matching Pods exist
when only three are desired, it works to bring the count back down.

A ReplicaSet provides replica-count maintenance and contributes to
recovery when Pods disappear. It is usually managed by a Deployment
rather than created directly for a typical application.

### Scaling a ReplicaSet versus scaling a Deployment

You can change replica counts directly, but when a Deployment owns the
ReplicaSet, **scale the Deployment** in normal workflows. The Deployment
then coordinates the desired replica count through its managed
ReplicaSet.

------------------------------------------------------------------------

## 9. Deployments

A **Deployment** is a higher-level Kubernetes object used to manage a
stateless application release. It manages ReplicaSets, which in turn
manage Pods.

``` text
Deployment
    |
    +-- ReplicaSet for version A
    |       |
    |       +-- Pod
    |       +-- Pod
    |
    +-- ReplicaSet for version B (after an update)
            |
            +-- Pod
```

During a rollout, more than one ReplicaSet may exist so Kubernetes can
gradually replace old Pods with new ones.

### 9.1 What a Deployment provides

-   **Replica management:** Declares how many Pods should run.
-   **Rolling updates:** Replaces Pods when the Pod template changes.
-   **Rollbacks:** Allows returning to an earlier Deployment revision.
-   **Rollout status:** Lets you inspect whether an update has
    completed.
-   **Version history:** Tracks Deployment revisions, within configured
    history limits.

The Deployment is not itself the container. It describes and manages the
desired application workload.

### 9.2 Scaling

If the Deployment specifies:

``` yaml
spec:
  replicas: 2
```

then it requests two Pod replicas. Changing the value to five asks
Kubernetes to maintain five replicas.

You can also scale from the command line:

``` bash
kubectl scale deployment flask-demo --replicas=5
```

Replace `flask-demo` with the actual Deployment name. If you later
re-apply a YAML file that declares two replicas, that file may change
the desired count back to two.

### 9.3 Rolling updates and rollbacks

A new image or another change to the Pod template can trigger a new
rollout. Kubernetes gradually moves from the previous ReplicaSet to the
new one according to the rollout strategy.

Useful commands:

``` bash
kubectl rollout status deployment/flask-demo
kubectl rollout history deployment/flask-demo
kubectl rollout undo deployment/flask-demo
```

A rollout may stall if the image cannot be downloaded, the container
repeatedly crashes, or the application never becomes ready. Inspect Pod
events and logs to diagnose the cause.

------------------------------------------------------------------------

## 10. Services and Networking

A **Service** provides a stable way to reach a set of Pods. It uses a
selector to identify eligible Pods and exposes a stable virtual IP and
DNS name for clients to use.

Pods may be recreated, but clients can keep using the Service name.

``` text
Browser or client
       |
       v
Service: flask-demo
       |
       +------> Pod 1 (Flask)
       |
       +------> Pod 2 (Flask)
```

The Service does not create Pods. It routes traffic to eligible backends
that match its selector and are available as endpoints.

### 10.1 Service types

#### `ClusterIP`

-   The default Service type.
-   Exposes the Service on an internal cluster IP.
-   Normally reachable from within the cluster, not directly from an
    external browser.

Use it when the application should be reachable internally.

#### `NodePort`

-   Exposes the Service on a port on each eligible node.
-   The usual NodePort range is `30000–32767`, unless the cluster is
    configured differently.
-   In local Minikube setups, it can provide a way to reach an
    application from outside the cluster, depending on the environment
    and networking setup.

#### `LoadBalancer`

-   Requests an external load balancer through an environment that
    supports it, commonly a cloud provider integration.
-   Commonly used in managed cloud Kubernetes environments.
-   On a local cluster, extra support may be required; declaring this
    type does not guarantee that a public load balancer will appear.

### 10.2 `port` versus `targetPort` versus `nodePort`

These fields describe different parts of the traffic path:

``` text
Client
  |
  | connects to Service port
  v
Service: port
  |
  | forwards to a selected Pod's port
  v
Pod / container: targetPort
```

-   **`port`:** The port exposed by the Service.
-   **`targetPort`:** The port on the selected Pod to which traffic is
    forwarded.
-   **`nodePort`:** For a NodePort Service, the port exposed on the
    node.

Example:

``` yaml
ports:
  - port: 80
    targetPort: 5000
```

A client connects to Service port `80`, and traffic is forwarded to port
`5000` on the selected Pod. Flask must actually be listening on that
target port.

### 10.3 How does the Service find Pods?

The Service's selector matches Pod labels. Kubernetes tracks the
matching, eligible backends as endpoints (commonly represented through
EndpointSlices).

For example:

``` yaml
selector:
  app: flask-demo
```

A Pod with `app: flask-demo` can match the selector. A Pod labelled
`app: another-app` will not match it.

### 10.4 How does a browser reach a local Minikube application?

The exact command depends on the Service type and the operating
system/network setup. For a suitable Service, try:

``` bash
minikube service flask-demo
```

This asks Minikube to provide a way to access that Service. For a
`ClusterIP` Service, Minikube may open a tunnel or provide an access URL
depending on the environment. Do not assume that a `ClusterIP` is
directly reachable from every browser or host setup.

------------------------------------------------------------------------

## 11. Understanding Kubernetes YAML

Kubernetes resources are commonly described in **YAML** files. YAML is a
human-readable format for structured configuration. Indentation matters:
use consistent spaces and do not use tabs for indentation.

A typical Kubernetes manifest contains these fields:

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-demo
spec:
  # Desired configuration goes here
```

### 11.1 `apiVersion`

Specifies which Kubernetes API version defines the resource.

Examples: - `apps/v1` for a Deployment or ReplicaSet. - `v1` for a Pod
or Service.

### 11.2 `kind`

Specifies the type of object to create, such as `Deployment`, `Service`,
or `Pod`.

### 11.3 `metadata`

Contains identifying information about the object, commonly its `name`,
`labels`, and optional `namespace`.

### 11.4 `spec`

The **specification** describes the desired configuration of the object.
Its fields depend on the resource type.

For a Deployment, `spec` can include `replicas`, `selector`, and
`template`.

### 11.5 `replicas`

The number of Pod copies the Deployment should maintain.

``` yaml
replicas: 2
```

### 11.6 `selector`

Identifies which Pods the Deployment manages.

``` yaml
selector:
  matchLabels:
    app: flask-demo
```

### 11.7 `template`

The **Pod template** is the blueprint used to create Pods. It contains
Pod metadata and the Pod specification.

``` yaml
template:
  metadata:
    labels:
      app: flask-demo
  spec:
    containers:
      - name: flask-container
        image: flask-demo:latest
```

The template's labels must match the Deployment selector.

### 11.8 `containers`, `image`, and `ports`

-   **`containers`:** Lists the containers inside the Pod.
-   **`name`:** The container's name within the Pod.
-   **`image`:** The container image Kubernetes should run.
-   **`ports`:** Documents container ports that the application uses. It
    does not alone expose those ports outside the Pod.

### 11.9 `resources`

Resource requests and limits tell Kubernetes about CPU and memory needs.

-   **Request:** The amount of a resource used for scheduling decisions
    and as a declared baseline for the container.
-   **Limit:** The maximum amount the container is allowed to use for a
    resource, subject to the resource's behaviour.

Example:

``` yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "256Mi"
```

`100m` CPU means one-tenth of one CPU core. `Mi` means mebibytes, a
binary unit of memory. These are example values, not necessarily the
right settings for every Flask application.

### 11.10 Example Deployment manifest

This is an illustrative example. Adjust the image name, port, and
resource values to match your repository.

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: flask-demo
  template:
    metadata:
      labels:
        app: flask-demo
    spec:
      containers:
        - name: flask-container
          image: flask-demo:latest
          ports:
            - containerPort: 5000
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
```

Notice how the Deployment selector and Pod template both use
`app: flask-demo`. That matching label is essential.

### 11.11 Example Service manifest

This example exposes the Flask application internally on Service port
`80` and forwards traffic to port `5000` on the matching Pods.

``` yaml
apiVersion: v1
kind: Service
metadata:
  name: flask-demo
spec:
  type: NodePort
  selector:
    app: flask-demo
  ports:
    - port: 80
      targetPort: 5000
```

Kubernetes can assign a NodePort if one is not specified. For local
access, use the instructions appropriate to your Minikube setup.

------------------------------------------------------------------------

## 12. Useful `kubectl` Commands

`kubectl` (pronounced "cube control" or "cube cuddle") is the
command-line tool used to communicate with a Kubernetes cluster.

### 12.1 Apply and inspect resources

``` bash
# Create or update resources described in a YAML file
kubectl apply -f deployment.yaml

# List Pods
kubectl get pods

# List Deployments
kubectl get deployments

# List Services
kubectl get services

# List common resources together
kubectl get all
```

`kubectl get all` is a convenient summary, not literally every resource
type in the cluster.

### 12.2 Inspect a resource

``` bash
kubectl describe pod POD_NAME
kubectl describe deployment flask-demo
kubectl describe service flask-demo
```

`describe` provides detailed information such as labels, state, node
assignment, events, and configuration. Events are especially useful when
something fails.

### 12.3 Read logs

``` bash
kubectl logs POD_NAME
```

For a Deployment, you can also use:

``` bash
kubectl logs deployment/flask-demo
```

If a Pod contains multiple containers, specify the container with
`-c CONTAINER_NAME`.

Logs help you see application output and errors. They do not always
explain scheduling or image-pull failures; use `describe` and events for
those.

### 12.4 Execute a command inside a container

``` bash
kubectl exec -it POD_NAME -- sh
```

This opens a shell if `sh` exists in the container image. Some minimal
images do not include a shell or debugging tools. Use `bash` only if it
is installed.

### 12.5 Scale a Deployment

``` bash
kubectl scale deployment flask-demo --replicas=3
```

This changes the desired replica count to three.

### 12.6 Check rollout status and undo a rollout

``` bash
kubectl rollout status deployment/flask-demo
kubectl rollout history deployment/flask-demo
kubectl rollout undo deployment/flask-demo
```

### 12.7 Delete resources

``` bash
kubectl delete -f deployment.yaml
```

This deletes resources defined in that file in the current Kubernetes
context and namespace, subject to the objects in the manifest.

### 12.8 A useful debugging sequence

``` bash
kubectl get pods
kubectl describe pod POD_NAME
kubectl logs POD_NAME
kubectl get services
kubectl describe service flask-demo
```

Replace placeholder names with the real names in your cluster. Also
check `kubectl get events --sort-by=.metadata.creationTimestamp` when
recent cluster events may explain the issue.

------------------------------------------------------------------------

## 13. Minikube

**Minikube** is a tool for running a local Kubernetes cluster on a
development machine. It is useful for learning and testing without
needing a cloud cluster.

Conceptually:

``` text
Your laptop
   |
   v
Minikube local cluster
   |
   +-- Kubernetes components
   +-- Pods running your application
   +-- Services for accessing the application
```

Depending on its driver and configuration, Minikube may run the cluster
inside a virtual machine or container. It is not necessarily a separate
physical computer.

### Useful commands

``` bash
# Start a local cluster
minikube start

# Check cluster status
minikube status

# Open the Kubernetes dashboard, if supported
minikube dashboard

# Access a Service
minikube service flask-demo

# Stop the cluster
minikube stop

# Delete the local cluster
minikube delete
```

`minikube delete` removes the Minikube cluster and its local cluster
data. Use it carefully.

### Image availability in Minikube

A common beginner problem is building an image locally and then asking
Kubernetes to run it, while the Minikube node cannot see that image.

The image must be available to the cluster. Depending on your setup, you
can: - Push the image to a registry and refer to it by its image name. -
Load the image into Minikube using a command such as
`minikube image load flask-demo:latest`. - Build the image in Minikube's
image environment if your setup supports that workflow.

Also check `imagePullPolicy`. For a local learning image, an image name
and pull policy that make sense for your setup can prevent Kubernetes
from unnecessarily trying to download an image that exists only locally.

------------------------------------------------------------------------

## 14. Docker and Kubernetes Together

A common workflow for this project is:

``` text
1. Write the Flask application
          |
          v
2. Define dependencies and a Dockerfile
          |
          v
3. Build a container image
          |
          v
4. Make the image available to Kubernetes
   (registry or local Minikube image)
          |
          v
5. Apply Kubernetes manifests
          |
          v
6. Deployment creates/manages ReplicaSet and Pods
          |
          v
7. Service provides a stable way to reach the Pods
          |
          v
8. Open the application and test it
```

### What each step means

1.  **Write code:** Create the Flask routes and application logic in
    `app.py`.
2.  **Dockerfile:** Describe how to build an image containing the
    application and its dependencies.
3.  **Build image:** Docker builds the image from the Dockerfile.
4.  **Make the image available:** Kubernetes nodes need access to the
    image, either through a registry or an image loaded into the local
    cluster.
5.  **Apply manifests:** `kubectl apply` sends the desired Deployment
    and Service configuration to Kubernetes.
6.  **Start Pods:** Controllers and the scheduler coordinate Pod
    creation and placement; kubelet and the container runtime start
    containers.
7.  **Expose the application:** The Service selects matching Pods and
    provides stable networking.
8.  **Test:** Use the URL or access method appropriate to your Service
    type and Minikube setup.

**Important:** A Docker image is a packaged artifact; a running
container is an instance of that image. A Pod is the Kubernetes unit
that contains the container. These terms are related, but they are not
interchangeable.

------------------------------------------------------------------------

## 15. Understanding the Project Files

The exact filenames and contents depend on the repository. The following
describes the roles of the files in this mini project.

### 15.1 `app.py` --- the Flask application

This is the application code.

It may: - Create the Flask application object. - Define routes for HTTP
requests, such as `GET` or `POST`. - Read request data or form input. -
Return HTML, JSON, or another response. - Listen on the port configured
for the project.

**HTTP request:** A message sent by a client to a web server.\
**Route:** A URL pattern connected to a Python function.\
**GET:** Commonly used to retrieve a resource.\
**POST:** Commonly used to submit data or trigger an operation.

If Flask runs inside the container on port `5000`, the Kubernetes
`targetPort` should point to the port where the application is actually
listening. The Flask development server is suitable for learning;
production deployments usually use a production-ready WSGI server.

### 15.2 `Dockerfile` --- image build instructions

A Dockerfile contains instructions for building the application image.

Common instructions include:

-   `FROM`: Select a base image.
-   `WORKDIR`: Set the working directory inside the image.
-   `COPY`: Copy application files into the image.
-   `RUN`: Execute a command during image build, often to install
    dependencies.
-   `EXPOSE`: Document a port the application is expected to use. This
    does not publish the port by itself.
-   `CMD` or `ENTRYPOINT`: Define the command that runs when the
    container starts.

A Dockerfile might install Python packages, copy `app.py`, and configure
the command that launches the Flask application. The actual steps depend
on the repository's Dockerfile.

### 15.3 `deployment.yaml` --- desired application workload

A Deployment manifest commonly defines:

-   The Deployment's name.
-   The number of Pod replicas.
-   The selector used to identify managed Pods.
-   The Pod template and its labels.
-   The container image and command.
-   Container ports and optional resource requests/limits.

Applying this file asks Kubernetes to create or update the declared
Deployment. It does not mean that every Pod is instantly ready; inspect
the rollout and Pod status.

### 15.4 Service manifest --- stable network access

The Service manifest commonly defines:

-   The Service name and type.
-   The Pod-label selector.
-   The Service port.
-   The target port inside the Pod.

Its selector must match the labels on the intended Pods. Otherwise, the
Service may have no usable backends.

### 15.5 How to connect the files mentally

``` text
app.py
  | application source code
  v
Dockerfile
  | build instructions
  v
Container image
  | referenced by
  v
deployment.yaml
  | creates/manages
  v
Deployment -> ReplicaSet -> Pods -> Container running Flask

Service manifest -> Service -> matching Pods
```

The Service is a separate networking object; it is not created
automatically just because a Deployment exists unless you separately
define or create it.

------------------------------------------------------------------------

## 16. End-to-End Execution Flow

Here is the complete story of what happens when you deploy the Flask
mini project.

### Phase A: Build the application image

1.  You write the Flask code in `app.py`.
2.  You define the environment and startup command in the Dockerfile.
3.  You run a Docker build command to create an image.
4.  You make that image available to the Minikube cluster or a registry
    accessible to its nodes.

### Phase B: Ask Kubernetes to deploy it

5.  You run `kubectl apply -f deployment.yaml`.
6.  `kubectl` sends the manifest to the API Server.
7.  Kubernetes validates and records the desired object state.
8.  The Deployment controller creates or updates a ReplicaSet.
9.  The ReplicaSet works to create the required Pods.
10. The scheduler assigns new Pods to suitable nodes.
11. The kubelet on each selected node works with the container runtime
    to start the containers.
12. Flask starts inside each container, assuming the image,
    configuration, and startup command are correct.

### Phase C: Make the application reachable

13. You apply the Service manifest.
14. The Service selector identifies matching Pods.
15. Kubernetes tracks eligible Service backends.
16. You access the Service using a suitable method, such as
    `minikube service flask-demo`.
17. The request reaches the Service and is forwarded to an eligible Pod.
18. The Flask application handles the request and returns a response.
19. The response travels back to the client.

### Full flow diagram

``` text
Developer writes app.py
        |
        v
Dockerfile defines image build
        |
        v
Build container image
        |
        v
Make image available to cluster
        |
        v
kubectl apply -f deployment.yaml
        |
        v
API Server accepts the request
        |
        v
Deployment controller manages ReplicaSet
        |
        v
ReplicaSet maintains Pod replicas
        |
        v
Scheduler selects node(s)
        |
        v
kubelet + container runtime start containers
        |
        v
Flask runs inside Pod(s)
        |
        v
Service selects matching, eligible Pods
        |
        v
Client request -> Service -> Pod -> Flask
        |
        v
Flask response -> client
```

------------------------------------------------------------------------

## 17. Troubleshooting Common Problems

When the application does not work, do not guess. Check each layer.

### Problem: Pod is stuck in `Pending`

Possible causes: - No suitable node is available. - Requested CPU or
memory cannot be allocated. - A scheduling constraint prevents
placement.

Try:

``` bash
kubectl describe pod POD_NAME
kubectl get events --sort-by=.metadata.creationTimestamp
```

### Problem: `ImagePullBackOff` or `ErrImagePull`

Kubernetes cannot successfully download or access the image.

Possible causes: - Incorrect image name or tag. - Private registry
credentials are missing or incorrect. - The image is only on your laptop
and is not available to Minikube. - Registry access is unavailable.

Check the Pod events, image name, registry access, and Minikube image
availability.

### Problem: `CrashLoopBackOff`

A container repeatedly starts and exits, and Kubernetes is delaying
another restart.

Possible causes: - Python exception or application bug. - Incorrect
startup command. - Missing dependency or environment variable. - The app
exits immediately.

Try:

``` bash
kubectl logs POD_NAME
kubectl describe pod POD_NAME
```

If the container restarted, previous-container logs may help:

``` bash
kubectl logs POD_NAME --previous
```

### Problem: Service exists, but the application is unreachable

Check these items:

1.  Does the Service selector match the Pod labels?
2.  Are the Pods running and ready?
3.  Is `targetPort` correct?
4.  Is Flask listening on the expected port and on an interface
    reachable from outside the process (commonly `0.0.0.0` inside a
    container)?
5.  Is the Service type and Minikube access method correct?

Useful commands:

``` bash
kubectl get pods --show-labels
kubectl get services
kubectl describe service flask-demo
kubectl get endpointslices
```

An empty set of endpoints or EndpointSlices for the Service can indicate
a selector mismatch or no eligible Pods.

### Problem: `kubectl` cannot connect to the cluster

Check:

``` bash
minikube status
kubectl config current-context
kubectl get nodes
```

Start Minikube if it is stopped, and verify that `kubectl` is using the
intended cluster context.

------------------------------------------------------------------------

## 18. Quick Revision Sheet

  -----------------------------------------------------------------------
  Term                                Remember it as
  ----------------------------------- -----------------------------------
  Kubernetes (K8s)                    Platform that orchestrates
                                      containerized workloads

  Cluster                             The machines and Kubernetes
                                      components working together

  Control Plane                       The management and decision-making
                                      layer

  Worker Node                         A machine where workload Pods run

  API Server                          Entry point for Kubernetes API
                                      requests

  etcd                                Key-value store for Kubernetes
                                      cluster state

  Scheduler                           Chooses a suitable node for a Pod

  Controller                          Repeatedly works to maintain a
                                      resource's desired state

  kubelet                             Node agent that makes assigned Pods
                                      run

  Container runtime                   Software that starts and stops
                                      containers

  kube-proxy                          Common implementation component for
                                      Service networking

  Object                              A Kubernetes resource record

  Desired state                       The configuration you ask
                                      Kubernetes to maintain

  Actual state                        What currently exists or is
                                      happening

  Reconciliation loop                 Compare desired and actual state,
                                      then act

  Pod                                 Smallest deployable Kubernetes unit

  Container                           Running instance of a packaged
                                      application image

  Image                               Packaged application filesystem and
                                      metadata used to start containers

  Label                               Key-value metadata attached to an
                                      object

  Selector                            Rule for matching objects by labels

  ReplicaSet                          Maintains the desired number of
                                      matching Pods

  Deployment                          Manages ReplicaSets, releases,
                                      rollouts, and rollbacks

  Service                             Stable network endpoint for
                                      matching Pods

  ClusterIP                           Service reachable internally in the
                                      cluster

  NodePort                            Service exposed through a port on
                                      cluster nodes

  LoadBalancer                        Service type that requests an
                                      external load balancer where
                                      supported

  `port`                              Port exposed by the Service

  `targetPort`                        Port on the selected Pod receiving
                                      forwarded traffic

  `nodePort`                          Node-level port used by a NodePort
                                      Service

  YAML manifest                       File describing the desired
                                      configuration of Kubernetes
                                      resources

  `kubectl`                           Command-line tool for talking to
                                      Kubernetes

  Minikube                            Tool for running a local Kubernetes
                                      cluster

  Dockerfile                          Instructions for building a
                                      container image

  Flask                               Python web framework used by the
                                      application
  -----------------------------------------------------------------------

### The five relationships to memorise

1.  **Dockerfile → Image → Container:** instructions build the package;
    the package is used to start a running container.
2.  **Deployment → ReplicaSet → Pod → Container:** the Deployment
    manages the release, the ReplicaSet maintains replica count, the Pod
    hosts the container.
3.  **Service → selector → matching Pods:** the Service uses labels to
    find its backends.
4.  **Desired state vs actual state → reconciliation:** controllers keep
    working toward the requested configuration.
5.  **`kubectl` → API Server → cluster resources:** `kubectl` sends
    commands through the Kubernetes API.

------------------------------------------------------------------------

## 19. Self-Test Questions

Try to answer these without looking at the revision sheet.

1.  What is the difference between Docker and Kubernetes?
2.  Why does Kubernetes schedule Pods rather than schedule containers
    directly?
3.  What happens if a Deployment asks for three replicas but only two
    Pods exist?
4.  What is the difference between a Deployment and a ReplicaSet?
5.  Why should a client usually use a Service rather than a Pod IP
    address?
6.  What is the difference between a label and a selector?
7.  What is the difference between Service `port` and `targetPort`?
8.  What does the API Server do?
9.  What is the responsibility of the scheduler, and how is it different
    from the kubelet?
10. Why might a Pod show `ImagePullBackOff`?
11. Why might a Service exist but have no working backends?
12. What is the purpose of Minikube?
13. What does `kubectl rollout undo` do?
14. Does declaring `containerPort: 5000` automatically expose Flask to
    the internet?
15. Describe the path of an HTTP request from the browser to Flask and
    back.

### Suggested answers

1.  Docker tools build and run containers; Kubernetes coordinates and
    manages workloads made of containers.
2.  A Pod is Kubernetes' smallest deployable unit and provides the
    shared environment for its container or containers.
3.  The ReplicaSet works to create another Pod so the actual state moves
    toward three replicas.
4.  A Deployment manages application rollouts and ReplicaSets; a
    ReplicaSet maintains the number of matching Pods.
5.  Pod IPs can change; a Service gives clients a stable virtual IP and
    DNS name.
6.  A label is a key-value tag; a selector is a rule that matches tags.
7.  `port` is the Service-facing port; `targetPort` is the port on the
    selected Pod to which traffic is forwarded.
8.  The API Server exposes the Kubernetes API and processes requests
    from tools and components.
9.  The scheduler chooses a node for an unscheduled Pod; the kubelet on
    that node works to make the assigned Pod run.
10. Kubernetes cannot successfully retrieve or access the specified
    image.
11. Its selector may not match the Pod labels, or no matching Pods may
    be ready and eligible.
12. Minikube provides a local Kubernetes cluster for learning and
    testing.
13. It asks Kubernetes to return the Deployment to a previous revision,
    when that revision is available.
14. No. It documents the container port; a Service and an appropriate
    access path are separate concerns.
15. The client connects to the Service; the Service forwards traffic to
    an eligible Pod; Flask handles it and returns a response through the
    network path.

------------------------------------------------------------------------

## Final mental model

If you remember only one picture, remember this:

``` text
Dockerfile -> Image -> Container
                         ^
                         |
Deployment -> ReplicaSet -> Pod
                            ^
                            |
Browser -> Service ---------+
```

-   **Flask** is the application.
-   **Docker** packages the application into an image.
-   **Kubernetes** manages where and how the application runs.
-   **Deployment** describes and manages the application rollout.
-   **ReplicaSet** keeps the requested number of Pods.
-   **Pod** is the unit that contains the running container.
-   **Service** gives clients a stable way to reach the matching Pods.
-   **Minikube** lets you practise this workflow locally.
-   **`kubectl`** is the tool you use to tell Kubernetes what you want
    and inspect what is happening.

Read the guide once for the big picture, then revisit the architecture,
YAML, and end-to-end flow while looking at the actual repository files.
The exact image names, ports, labels, and commands should always be
checked against your project's files.
