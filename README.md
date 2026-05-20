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
