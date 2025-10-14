# **003:** Understanding Kubernetes Core Objects: Pods, Services, Deployments, ConfigMaps, and Secrets — *Basic*

## Overview

In session 001, we learned how to run Kubernetes and deploy our first application. Now it's time to dive deeper into the fundamental building blocks that make Kubernetes applications work: **Pods**, **Services**, **Deployments**, **ConfigMaps**, and **Secrets**.

Think of these as the foundation of any Kubernetes application:

- **Pods** are the smallest unit you can deploy—like individual containers with shared resources
- **Services** provide stable networking to reach your Pods—like a phone number that always works even if you move houses
- **Deployments** manage the lifecycle of your Pods—like a manager ensuring the right number of workers are always available
- **ConfigMaps** store configuration data—like a settings file that can be shared between applications
- **Secrets** store sensitive data securely—like a locked safe for passwords and certificates

We'll start simple and build up complexity step by step, exploring what happens under the hood with **ReplicaSets** and **Endpoints**. Everything will be done declaratively using YAML manifests to follow best practices.

## Prerequisites

Make sure you have a Kubernetes cluster running. If you followed session 001, you can use the same kind cluster:

```sh
CLUSTER_NAME="cloud-native-kubernetes-workshop"
kind create cluster --name $CLUSTER_NAME --config kind-cluster.yaml
```

Or use any other cluster you have available.

```sh
alias k="kubectl"
k get nodes
```

## Testing Connectivity

Throughout this session, we'll test our services using temporary pods. Here's the pattern we'll use:

```sh
# Quick one-liner for testing (pod is automatically cleaned up)
k run tmp-shell --rm -i --image nicolaka/netshoot --restart=Never -- curl http://service-name
```

## Files in this Session

## Step 1: Creating a ConfigMap

ConfigMaps allow you to separate configuration from your application code. Let's start by creating a ConfigMap that contains some basic web server configuration.

The ConfigMap is defined in `manifests/web-config.yaml` and contains:

- Simple key-value pairs for server configuration
- An nginx configuration file
- A custom HTML page that will be served

Apply the ConfigMap:

```sh
k apply -f manifests/web-config.yaml
k get configmaps
k describe configmap web-config
```

## Step 2: Creating a Secret

Secrets are similar to ConfigMaps but are designed for sensitive data. Let's create a Secret for database credentials.

The Secret is defined in `manifests/db-secret.yaml` and contains:

- Base64 encoded username and password
- Plain text host and port (using stringData)

Apply the Secret:

```sh
k apply -f manifests/db-secret.yaml
k get secrets
k describe secret db-secret
```

Notice that when you describe the secret, the actual values are hidden for security.

## Step 3: Creating a Pod that Uses ConfigMap and Secret

Now let's create a Pod that uses both the ConfigMap and Secret.

The Pod is defined in `manifests/nginx-pod.yaml` and demonstrates:

- Environment variables from ConfigMap and Secret
- Volume mounts for configuration files
- Proper labeling for service discovery

Apply the Pod:

```sh
k apply -f manifests/nginx-pod.yaml
k get pods
k describe pod nginx-pod
```

Let's verify the configuration is working:

```sh
# Check environment variables
k exec nginx-pod -- env | grep -E "(SERVER_NAME|MAX_CONNECTIONS|DB_)"

# Check mounted files
k exec nginx-pod -- cat /etc/nginx/conf.d/default.conf
k exec nginx-pod -- cat /usr/share/nginx/html/index.html
```

## Step 4: Creating a Service

Now let's create a Service to expose our Pod.

The Service is defined in `manifests/nginx-service.yaml` and provides:

- Stable networking using label selectors
- Load balancing across matching Pods
- Internal cluster DNS name

Apply the Service:

```sh
k apply -f manifests/nginx-service.yaml
k get services
k describe service nginx-service
```

Test the Service:

```sh
# Create a temporary pod to test connectivity
k run tmp-shell --rm -i --image nicolaka/netshoot --restart=Never -- curl http://nginx-service

# Test the info endpoint to see which Pod is responding
k run tmp-shell --rm -i --image nicolaka/netshoot --restart=Never -- curl http://nginx-service/info
```

You should see the custom HTML page from our ConfigMap! The `/info` endpoint will show you which Pod (hostname) is serving the request.

## Step 5: Understanding Endpoints

Let's examine how the Service connects to our Pod through Endpoints:

```sh
k get endpoints nginx-service
k describe endpoints nginx-service
```

The Endpoints object shows which Pod IPs the Service will route traffic to based on the label selector.

## Step 6: Creating a Deployment

While our single Pod works, let's create a Deployment for better management and scaling.

The Deployment is defined in `manifests/nginx-deployment.yaml` and includes:

- Multiple replicas for high availability
- Resource requests and limits
- Health checks (liveness and readiness probes)
- Same ConfigMap and Secret integration as the Pod

First, let's delete our single Pod and then apply the Deployment:

```sh
k delete -f manifests/nginx-pod.yaml
k apply -f manifests/nginx-deployment.yaml
```

Check what was created:

```sh
k get deployments
k get replicasets
k get pods -l app=nginx
```

## Step 7: Understanding the Full Picture

Now let's see how all these objects work together:

### The Object Hierarchy

```text
ConfigMap ────┐
              ├─→ Deployment → ReplicaSet → Pods
Secret ───────┘                              ↑
                                             │
                              Service → Endpoints
```

### Verify Everything is Working

Check that our Service now load balances across all Pods:

```sh
# Check endpoints - should show 3 Pod IPs
k get endpoints nginx-service
k describe endpoints nginx-service

# Test load balancing - see which Pod serves each request
for i in {1..6}; do
  echo "=== Request $i ==="
  k run tmp-shell-$i --rm -i --image nicolaka/netshoot --restart=Never -- curl -s http://nginx-service/info
done
```

### Examining the ReplicaSet

A Deployment creates a ReplicaSet, which manages the Pods:

```sh
k get replicasets -l app=nginx
k describe replicaset $(k get rs -l app=nginx -o jsonpath='{.items[0].metadata.name}')
```

### Test Self-Healing

Let's see Kubernetes self-healing by deleting a Pod:

```sh
# Watch Pods in one terminal
k get pods -l app=nginx -w

# In another terminal, delete a Pod
k delete pod $(k get pods -l app=nginx -o jsonpath='{.items[0].metadata.name}')
```

The ReplicaSet will immediately create a new Pod to maintain the desired replica count of 3.

### Scale the Deployment

```sh
# Scale up
k scale deployment nginx-deployment --replicas=5
k get pods -l app=nginx

# Check that Service endpoints are updated
k get endpoints nginx-service

# Scale back down
k scale deployment nginx-deployment --replicas=2
k get pods -l app=nginx
```

## Step 8: Configuration Management in Practice

Let's update our configuration and see how it affects running Pods:

### Update the ConfigMap

We have an updated version of the ConfigMap in `manifests/web-config-updated.yaml` with different HTML content.

Apply the updated ConfigMap:

```sh
k apply -f manifests/web-config-updated.yaml
```

The ConfigMap is updated, but the running Pods won't see the change until they restart. Let's restart the Deployment:

```sh
k rollout restart deployment nginx-deployment
k rollout status deployment nginx-deployment
```

Test the updated content:

```sh
k run tmp-shell --rm -i --image nicolaka/netshoot --restart=Never -- curl http://nginx-service
```

## Key Takeaways

1. **ConfigMaps** store non-sensitive configuration data that can be consumed as environment variables or mounted files
2. **Secrets** store sensitive data with base64 encoding and additional security measures
3. **Pods** are the smallest deployable units that can reference ConfigMaps and Secrets
4. **Services** provide stable networking and load balancing using label selectors
5. **Deployments** manage the lifecycle of Pods through ReplicaSets
6. **ReplicaSets** ensure the desired number of Pod replicas are running
7. **Endpoints** automatically track which Pods a Service routes to
8. **Labels and selectors** connect all objects together
9. Configuration updates require Pod restarts to take effect
10. Kubernetes provides **self-healing**, **scaling**, and **rolling updates** automatically

## Cleanup

Let's clean up our resources:

```sh
k delete -f manifests/nginx-deployment.yaml
k delete -f manifests/nginx-service.yaml
k delete -f manifests/web-config.yaml
k delete -f manifests/db-secret.yaml

# Verify cleanup
k get all
k get configmaps
k get secrets
```

or just delete the cluster

```sh
kind delete cluster --name $CLUSTER_NAME
```