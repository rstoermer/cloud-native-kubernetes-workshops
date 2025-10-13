# **001:** Run Kubernetes (Local, Edge, Cloud) and deploy your first App — *Basic*

## Overview

Kubernetes is everywhere, but how do you actually run it? In this session, we’ll explore different ways to get a Kubernetes cluster up and running—from local setups for development to edge deployments and cloud-managed services.

The following lists are by far not exhaustive, but should give you a good starting point.

### Cloud Providers

Every Cloud provider has its own managed Kubernetes service, like:

* **Amazon EKS** (Elastic Kubernetes Service)
* **Google GKE** (Google Kubernetes Engine)
* **Azure AKS** (Azure Kubernetes Service)
* **OVHCloud** Managed Kubernetes Service

Of course, you can also set up Kubernetes manually on VMs or bare-metal servers in the cloud, but managed services take care of the control plane and simplify operations. Tools like the following can help with provisioning:

* **kops** (Kubernetes Operations): <https://github.com/kubernetes/kops>
* **kubeadm**: <https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/>
* **kubespray**: <https://github.com/kubernetes-sigs/kubespray>
* **hetzner kubernetes**: <https://github.com/vitobotta/hetzner-k3s> or <https://github.com/kube-hetzner/terraform-hcloud-kube-hetzner>

and many more...

### Edge

For edge deployments, lightweight Kubernetes distributions are often preferred due to resource constraints. Some popular options include:

* **k3s**: A lightweight Kubernetes distribution by Rancher, designed for IoT and edge computing. <https://k3s.io/>. Underlying OS needed.
* **k0s**: A zero-friction Kubernetes distribution that is easy to install and manage. <https://k0sproject.io/> . Underlying OS needed.
* **Talos**: A modern OS designed for running Kubernetes clusters securely and efficiently. <https://talos.dev/>. No underlying OS needed.
* **KubeSolo**: Single-node Kubernetes cluster for edge use cases. <https://github.com/portainer/kubesolo>. Underlying OS needed.

Talos offers a big advantage, as it is a complete OS (purpose built for Kubernetes) by itself, so you don’t need to install and manage an underlying operating system. It can be run on bare-metal servers or virtual machines.

### Local

All Edge distros can also be used locally on your laptop or workstation. However, there are also other options specifically designed for local development, as they are much quicker to start and destroy:

* **Minikube**: Run Kubernetes inside a VM. <https://minikube.sigs.k8s.io/docs/>
* **Kind** (Kubernetes IN Docker): Run Kubernetes clusters in Docker containers. <https://kind.sigs.k8s.io/>
* **k3d**: Run k3s in Docker containers. <https://k3d.io/>
* **Docker Desktop**: Includes a built-in Kubernetes cluster. <https://docs.docker.com/desktop/features/kubernetes/>

Minikube is a great choice for beginners, as is is closer to a classic Kubernetes setup with a VM. As kind is more lightweight, it is often preferred for CI/CD pipelines or when clusters need to be created and destroyed frequently.

## Let's get started

### kubeconfig

To interact with any Kubernetes cluster, you need a `kubeconfig` file. This file contains the necessary information to connect to the cluster, including the API server address, user credentials, and context.

When you create a cluster using any of the methods mentioned above, a `kubeconfig` file is usually generated for you. You can specify its location using the `KUBECONFIG` environment variable or place it in the default location (`~/.kube/config`).

```yaml
apiVersion: v1
kind: Config
preferences: {}

clusters:
- cluster:
  name: development
- cluster:
  name: test

users:
- name: developer
- name: experimenter

contexts:
- context:
  name: dev-frontend
- context:
  name: dev-storage
- context:
  name: exp-test
```

### Create a Cluster via kind

Let's create a local cluster using `kind` as an example. Make sure you have Docker installed and running.

```sh
CLUSTER_NAME="cloud-native-kubernetes-workshop"
kind create cluster --name $CLUSTER_NAME --config kind-cluster.yaml
```

### Verify the Cluster

```sh
alias k="kubectl"
k get nodes
```

Or show the labels we included to simulate availability zones:

```sh
k get nodes --show-labels
```

### Deploy a Container

Let's run a container imperatively:

```sh
k run nginx --image=nginx:1.29.1-alpine --port=80 -n default
```

Alternatively we can declare it via YAML. The YAML can be taken from the official documentation: <https://kubernetes.io/docs/concepts/workloads/pods/>, or easier, generated via kubectl without actually creating the resource:

```sh
k create deployment nginx --image=nginx:1.29.1-alpine -o yaml --dry-run=client > nginx-deployment.yaml
```

Then apply it:

```sh
k apply -f nginx-deployment.yaml
```

### Expose the Container

So, what can we do with this container? How can we access it? Well, luckily nothing is exposed by default in Kubernetes.

Let's start a throwaway Pod to test connectivity from within the cluster:

```sh
kubectl run tmp-shell --rm -i --tty --image nicolaka/netshoot -n default
```

Trying to reach the container is usually possible via DNS, Kubernetes handles that for us, however `curl nginx.default.svc.cluster.local` will not work, as the Pod is not exposed via a Service yet. There are different ways to find out the Pod IP, however this is not recommended, as Pods are ephemeral and can be restarted or rescheduled at any time.

```sh
k expose pod nginx --type=ClusterIP --port=80
```

So even if we restart our nginx container, scale it up or down, the Service will always point to the right Pod(s) and will be load-balanced by Kubernetes.

If we want to reach it from outside of the cluster, we have 3 options:

#### Port Forwarding

```sh
k port-forward svc/nginx 8080:80
```

Great for testing, but not for production.

#### NodePort

```sh
kubectl create service nodeport nginx --tcp=80:80 --node-port=30080
```

Well, this doesn't work. Why? Nothing is static in Kubernetes, everything is selected via labels. Our `kubectl run` command. created a Pod with the label `run=nginx`, but our Service is looking for Pods with the label `app=nginx`. We can fix that by adding the correct selector, or labeling the Pod correctly.

```sh
k label pod nginx app=nginx
```

or edit the Service via `k edit svc/nginx` and change the selector to `run: nginx`.

#### LoadBalancer

Kubernetes provides an API to provision a LoadBalancer, which works the same irrespective of the Cloud Provider. However, this only works if the cluster is running in an environment that supports LoadBalancers, like a Cloud Provider. When creating a LoadBalancer service, Kubernetes will communicate with the Cloud Provider's API to create the necessary resources and provision an external IP address.

This can also be done in kind using <https://github.com/kubernetes-sigs/cloud-provider-kind>.

```sh
k expose pod nginx --type=LoadBalancer --port=80
```

Get the provisioned IP address:

```sh
k get svc nginx -ojsonpath='{.status.loadBalancer.ingress[0].ip}'
```

or look it up manually via `k get svc`

Using this for an nginx Pod is not something you would do in real life, but instead create an Ingress or Gateway resource, which then routes traffic to the right Service based on hostname or path. But this is out of scope for this session.

### Cleanup

After we are finished, we can delete the cluster again.

```sh
kind delete cluster --name $CLUSTER_NAME
```
