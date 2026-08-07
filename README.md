Kubernetes Mini Project - Complete Revision README

«Goal: Learn the Kubernetes concepts used in the mini project before understanding the repository code.»

---

Table of Contents

1. Introduction
2. Why Kubernetes?
3. Kubernetes Architecture
4. Kubernetes Objects
5. Pods
6. ReplicaSet
7. Deployment
8. Services
9. Kubernetes YAML
10. kubectl Commands
11. Minikube
12. Docker + Kubernetes
13. Project Files Overview
14. End-to-End Execution Flow

---

1. Introduction

What is Kubernetes?

Kubernetes (K8s) is an open-source container orchestration platform that automates deploying, scaling, networking, and managing containerized applications.

Docker packages and runs containers.

Kubernetes manages those containers.

---

Why do we need Kubernetes?

Docker solves:

- Application packaging
- Dependency management
- Portability

Kubernetes solves:

- Deploying containers
- Scaling applications
- Self-healing
- Load balancing
- Service discovery
- Rolling updates
- Rollbacks
- High availability

---

Project Architecture

Browser
   │
   ▼
Kubernetes Service
   │
   ▼
Pod 1        Pod 2
   │            │
Flask      Flask
Container  Container
      ▲
      │
 ReplicaSet
      ▲
      │
 Deployment
      ▲
      │
 Kubernetes Cluster

---

2. Why Kubernetes?

Problems with Docker Alone

1. Scaling

One container cannot handle heavy traffic.

Instead of manually starting

Container1
Container2
Container3

Kubernetes automatically creates more Pods.

---

2. Self-Healing

If one container crashes,

Docker does nothing.

Kubernetes automatically creates a replacement Pod.

---

3. Load Balancing

If there are multiple Pods,

Kubernetes distributes requests among them.

User

↓

Service

↓

Pod1

Pod2

Pod3

---

4. Service Discovery

Pods have temporary IP addresses.

Kubernetes Services provide:

- Stable IP
- Stable DNS Name

---

5. Rolling Updates

Instead of

Stop Everything

↓

Deploy New Version

Kubernetes updates gradually.

v1

↓

v2

↓

v2

↓

v2

No downtime.

---

6. Rollback

If a deployment fails,

Kubernetes restores the previous version.

---

7. High Availability

Multiple Pods ensure the application remains available even if one Pod fails.

---

8. Scheduling

Kubernetes automatically decides which Worker Node should run each Pod based on available resources.

---

3. Kubernetes Architecture

A Kubernetes Cluster consists of:

Cluster

├── Control Plane
└── Worker Nodes

---

Cluster

A Cluster is a collection of machines working together.

---

Control Plane

Acts as the brain of Kubernetes.

Responsibilities:

- Accept requests
- Store cluster state
- Schedule Pods
- Monitor cluster
- Maintain desired state

---

Worker Node

Runs the actual application Pods.

Contains:

- kubelet
- kube-proxy
- container runtime
- Pods

---

kube-apiserver

The entry point of Kubernetes.

Every command like

kubectl apply
kubectl get pods
kubectl delete

first reaches the API Server.

---

etcd

Kubernetes database.

Stores:

- Deployments
- Pods
- Services
- Secrets
- ConfigMaps
- Cluster state

---

Scheduler

Chooses the best Worker Node for new Pods.

Considers:

- CPU
- Memory
- Node availability

---

Controller Manager

Maintains Desired State.

Checks continuously:

Desired State

vs

Actual State

If different,

it fixes the cluster.

---

kubelet

Runs on every Worker Node.

Responsibilities:

- Create Pods
- Monitor Pods
- Report Pod status
- Communicate with Control Plane

---

kube-proxy

Handles networking.

Responsible for:

- Service networking
- Load balancing
- Routing requests

---

Container Runtime

Actually starts containers.

Examples:

- containerd
- CRI-O

---

4. Kubernetes Objects

Everything in Kubernetes is an Object.

Examples:

- Pod
- ReplicaSet
- Deployment
- Service
- ConfigMap
- Secret

---

Desired State

Represents what we want.

Example

2 Pods

---

Actual State

Represents what currently exists.

Example

1 Pod

---

Reconciliation Loop

Controller Manager repeatedly checks

Desired State

↓

Actual State

↓

Different?

↓

Fix It

This is why Kubernetes is self-healing.

---

Relationship between Objects

Deployment

↓

ReplicaSet

↓

Pods

↓

Containers

Service is separate.

Service

↓

Pods

---

5. Pods

A Pod is the smallest deployable unit in Kubernetes.

Kubernetes never deploys containers directly.

Instead

Pod

↓

Container

---

Why Pods?

Pods allow multiple containers to:

- Share IP
- Share localhost
- Share storage
- Share lifecycle

Most applications use

1 Pod

↓

1 Container

---

Pod Lifecycle

Pending

↓

ContainerCreating

↓

Running

↓

Succeeded / Failed

---

Labels

Labels are key-value pairs.

Example

app=frontend

env=prod

Used to identify Pods.

---

Selectors

Selectors search for Pods using Labels.

Example

Selector

↓

app=frontend

↓

Matching Pods

Deployment and Service both use selectors.

---

6. ReplicaSet

Purpose:

Maintain the required number of Pods.

Example

Desired

3 Pods

Current

2 Pods

ReplicaSet creates

1 New Pod

---

ReplicaSet provides:

- Self-Healing
- Scaling

Usually managed by Deployment.

---

7. Deployment

Deployment manages ReplicaSets.

Hierarchy

Deployment

↓

ReplicaSet

↓

Pods

Deployment provides:

- Rolling Updates
- Rollback
- Scaling
- Version History
- Self-Healing

---

Scaling

replicas = 2

↓

replicas = 5

↓

Three more Pods created

---

Rolling Updates

Old version replaced gradually.

No downtime.

---

Rollback

Restore previous working Deployment.

---

8. Services

Pods change.

Services don't.

Service provides:

- Stable IP
- Stable DNS
- Load Balancing

Browser

↓

Service

↓

Pods

---

ClusterIP

Default Service.

Accessible only inside cluster.

---

NodePort

Exposes Service using a Node Port.

Useful in Minikube.

---

LoadBalancer

Creates cloud load balancer.

Used in AWS, Azure and GCP.

---

Port vs TargetPort

Browser

↓

Service Port

↓

Target Port

↓

Container

Service Port

- Port exposed by Service.

Target Port

- Port where application runs inside Pod.

---

9. Kubernetes YAML

Kubernetes configuration is written in YAML.

Common structure

apiVersion:
kind:
metadata:
spec:

---

apiVersion

Specifies API version.

Examples

apps/v1

v1

---

kind

Specifies object type.

Examples

Deployment

Service

Pod

---

metadata

Contains object information.

Examples

- name
- labels
- namespace

---

spec

Defines Desired State.

Contains

- replicas
- image
- ports
- resources
- selector

---

selector

Used to identify Pods.

---

template

Blueprint used to create Pods.

---

labels

Identity tags for Pods.

---

containers

Defines container configuration.

---

image

Docker image used by Pod.

---

ports

Container listening ports.

---

resources

Defines CPU and Memory limits.

---

10. kubectl Commands

Create Resources

kubectl apply -f deployment.yaml

---

View Pods

kubectl get pods

---

View Deployments

kubectl get deployment

---

View Services

kubectl get svc

---

Describe Resource

kubectl describe pod POD_NAME

Shows:

- Events
- IP
- Status
- Node
- Labels

---

View Logs

kubectl logs POD_NAME

Useful for debugging.

---

Execute inside Pod

kubectl exec -it POD_NAME -- bash

---

Scale Deployment

kubectl scale deployment app --replicas=5

---

Delete Resources

kubectl delete -f deployment.yaml

---

Rollout Status

kubectl rollout status deployment app

---

11. Minikube

Minikube creates a local Kubernetes Cluster.

Perfect for learning.

Architecture

Laptop

↓

Minikube

↓

Single Node Cluster

---

Useful Commands

Start

minikube start

Stop

minikube stop

Delete

minikube delete

Dashboard

minikube dashboard

Open Service

minikube service SERVICE_NAME

---

12. Docker + Kubernetes

Workflow

Write Code

↓

Dockerfile

↓

Docker Build

↓

Docker Image

↓

Docker Hub / Local Image

↓

Deployment

↓

Pods

↓

Service

↓

Browser

Docker packages the application.

Kubernetes runs and manages it.

---

13. Project Files Overview

app.py

Purpose:

Runs the Flask application.

Responsibilities

- Creates Flask app
- Handles GET and POST requests
- Reads user input
- Renders HTML
- Runs on Port 5000

---

Dockerfile

Purpose:

Build Docker Image.

Responsibilities

- Select base image
- Install dependencies
- Copy source code
- Expose application port
- Start Flask application

---

deployment.yaml

Purpose:

Deploy application to Kubernetes.

Responsibilities

- Create Deployment
- Maintain required Pods
- Define Docker Image
- Configure resource limits
- Configure container port
- Define Pod template

---

Service

Purpose:

Expose Pods through stable networking.

Responsibilities

- Find Pods using labels
- Load balance traffic
- Route requests
- Expose Service Port

---

14. End-to-End Execution Flow

Complete Flow

Developer

↓

Write Flask Code

↓

Dockerfile

↓

docker build

↓

Docker Image

↓

(Local Image / Docker Hub)

↓

kubectl apply

↓

API Server

↓

etcd

↓

Controller Manager

↓

Scheduler

↓

Worker Node

↓

kubelet

↓

containerd

↓

Pod Created

↓

Flask Starts

↓

Service Finds Pods

↓

Browser Request

↓

Service

↓

Pod

↓

Container

↓

Flask

↓

HTML Response

---

Final Revision Summary

Flask → Application

Docker → Package Application

Image → Runnable Artifact

Deployment → Manages Pods

ReplicaSet → Maintains Pod Count

Pod → Runs Containers

Service → Stable Networking

Labels → Pod Identity

Selectors → Find Pods

API Server → Entry Point

etcd → Cluster Database

Scheduler → Chooses Worker Node

kubelet → Creates Pods

containerd → Runs Containers

Controller Manager → Maintains Desired State
