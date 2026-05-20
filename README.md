# K8s Canary Release Simulation

This repository contains a practical simulation of a automated Canary Release pipeline using **Kubernetes** and **Argo Rollouts**. The project serves as technical documentation and a hands-on proof of concept for my thesis project.

The objective is to demonstrate how to minimize deployment risks by gradually shifting traffic to new application versions and executing automated rollbacks if an issue occurs.

## Architecture & Tools
- **Environment:** Killercoda Kubernetes Playground (K8s v1.35)
- **Deployment Strategy:** Canary Deployment (20% initial traffic split)
- **Traffic Routing & Controller:** Argo Rollouts

---

## Step-by-Step Implementation Guide

### 1. Setup Argo Rollouts Controller
First, create a dedicated namespace and install the official Argo Rollouts controller in the cluster:

    kubectl create namespace argo-rollouts
    kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml
    kubectl get pods -n argo-rollouts


### 2. Install the Argo Rollouts Kubectl Plugin
To view the visual deployment tree and manage rollouts easily, install the CLI plugin:

    curl -LO https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-linux-amd64
    chmod +x ./kubectl-argo-rollouts-linux-amd64
    mv ./kubectl-argo-rollouts-linux-amd64 /usr/local/bin/kubectl-argo-rollouts


### 3. Deploy the Application (Version 1 - Blue)
Deploy the sample application (Service and Rollout configuration) which specifies the canary rules (20% traffic steps):

    kubectl apply -f https://raw.githubusercontent.com/argoproj/argo-rollouts/master/docs/getting-started/basic/service.yaml
    kubectl apply -f https://raw.githubusercontent.com/argoproj/argo-rollouts/master/docs/getting-started/basic/rollout.yaml
    kubectl get pods


### 4. Execute a Successful Canary Release (Version 2 - Yellow)
Trigger a deployment update to the "yellow" version. The controller automatically limits the rollout to 1 pod (20% of the 5 replicas) and pauses.

    kubectl argo rollouts set image rollouts-demo rollouts-demo=argoproj/rollouts-demo:yellow
    kubectl argo rollouts get rollout rollouts-demo --watch

After validating the release, promote it to 100%:

    kubectl argo rollouts promote rollouts-demo


### 5. Simulate a Failed Deployment & Automated Rollback (Version 3 - Red)
Trigger a faulty update using the "red" image to test the safety mechanisms:

    kubectl argo rollouts set image rollouts-demo rollouts-demo=argoproj/rollouts-demo:red
    kubectl argo rollouts get rollout rollouts-demo --watch

Abort the rollout immediately to simulate an automated system rollback due to failed metrics/alerts:

    kubectl argo rollouts abort rollouts-demo

*Result: The controller instantly terminates the faulty red version and reverts 100% of the traffic back to the stable yellow version, ensuring zero downtime for the majority of users.*

<img width="968" height="661" alt="Terminal output showing the failed red deployment being aborted and automatically reverted." src="https://github.com/user-attachments/assets/a25c9d37-6b5b-4254-8983-4d74079708ae" />

*Text-version of image below.*
```
root@controlplane:~$ kubectl create namespace argo-rollouts
namespace/argo-rollouts created
root@controlplane:~$ kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml
customresourcedefinition.apiextensions.k8s.io/analysisruns.argoproj.io created
customresourcedefinition.apiextensions.k8s.io/analysistemplates.argoproj.io created
Name:            rollouts-demo
Namespace:       default
Status:          ✖ Degraded
Message:         RolloutAborted: Rollout aborted update to revision 3
Strategy:        Canary
  Step:          0/8
  SetWeight:     0
  ActualWeight:  0
Images:          argoproj/rollouts-demo:yellow (stable)
Replicas:
  Desired:       5
  Current:       5
  Updated:       0
  Ready:         5
  Available:     5

NAME                                       KIND        STATUS        AGE    INFO
⟳ rollouts-demo                            Rollout     ✖ Degraded    12m    
├──# revision:3                                                             
│  └──⧉ rollouts-demo-64fbcbbd74           ReplicaSet  • ScaledDown  5m43s  canary
├──# revision:2                                                             
│  └──⧉ rollouts-demo-76ddfb4f47           ReplicaSet  ✔ Healthy     9m31s  stable
│     ├──□ rollouts-demo-76ddfb4f47-5hjn8  Pod         ✔ Running     9m31s  ready:1/1
│     ├──□ rollouts-demo-76ddfb4f47-6b5tc  Pod         ✔ Running     7m50s  ready:1/1
│     ├──□ rollouts-demo-76ddfb4f47-4tlk9  Pod         ✔ Running     7m26s  ready:1/1
│     ├──□ rollouts-demo-76ddfb4f47-x68zq  Pod         ✔ Running     7m15s  ready:1/1
│     └──□ rollouts-demo-76ddfb4f47-ldjc8  Pod         ✔ Running     4m44s  ready:1/1
└──# revision:1                                                             
   └──⧉ rollouts-demo-5cd66ff7ff           ReplicaSet  • ScaledDown  12m
