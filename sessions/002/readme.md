# **002:** Create a Highly Available Kubernetes Cluster with Talos & Proxmox — *Intermediate*

## Explain like I'm 5

**What is Kubernetes?**
Think of Kubernetes like a hospital's central monitoring system that manages all the critical care equipment across multiple operating rooms and ICUs. It automatically decides which ventilators and anesthesia machines should run which patient monitoring software, ensures all devices are functioning properly and communicating with each other, and if one piece of equipment fails, it instantly redirects the workload to backup systems. If an entire room goes down, it seamlessly moves all critical functions to other rooms so patient care never stops.

**What is Talos?**
Talos is like having specialized, medical-grade computer systems that are purpose-built exclusively for running this hospital management system. Unlike general-purpose computers that might have games, web browsers, or other software that could interfere with critical operations, these systems contain only the essential components needed for medical device management. They're tamper-proof, can't be modified by unauthorized personnel, run from certified firmware that can't be corrupted, and are configured entirely through validated medical protocols - no technician needs to manually log in and potentially introduce human error into life-critical systems.

## Overview

 Talos Linux is a Kubernetes optimized Linux distro. It does one thing and does it well: Run Kubernetes. It is designed to:

- **Distributed** - Built for high-availability with no single points of failure, operating as a coordinated cluster of machines.
- **Immutable** - Runs from signed SquashFS images that are never modified, ensuring no configuration drift and system integrity.
- **Minimal** - Contains only necessary components with no shell, SSH, or GNU utilities, resulting in an OS under 80 MB.
- **Ephemeral** - All disk writes are replicated or reconstructable, allowing nodes to be wiped and rebuilt without data loss.
- **Secure** - Features encrypted communication, automatic certificate rotation, and follows kernel security recommendations with no passwords.
- **Declarative** - Entire system configuration managed through a single YAML manifest with no scripting required.
- **Purpose-built** - Ground-up rewrite designed exclusively for running Kubernetes clusters, not based on any existing distribution.

## CVE comparison as of 03.09.2025

| Operating System | Critical CVEs | High CVEs | Total not-fixed CVEs |
|------------------|---------------|-----------|---------------------|
| Talos Linux 1.11.0 | 0 | 29 | 6 |
| Flatcar 4230.2.2 | 27 | 75 | 2 |
| Ubuntu 22.04.05 | 280 | 1943 | 5653 |
| Rocky Linux 10 | 0 | 381 | 10808 |

## Setup Talos the CLI way

For trying this without ProxMox you can also start a Talos Cluster in Docker, see <https://docs.siderolabs.com/talos/v1.11/getting-started/quickstart#create-the-cluster>

For our more realistic setup we need:

- Proxmox VE
- Talosctl
- kubectl

In a real-world setup we would use Omni to onboard, provision and manage Talos clusters, but for simplicity and to show you what happens under the hood, we’ll do it manually with talosctl.

### Setup machines in Proxmox

1. Create a customer Talos image in their factory at <https://factory.talos.dev/>. Images can be declared and linked, [for example a bare-metal image inclusing QEMU guest agent](https://factory.talos.dev/?arch=amd64&cmdline-set=true&extensions=-&extensions=siderolabs%2Fqemu-guest-agent&platform=metal&target=metal&version=1.11.2)
2. Upload to ProxMox
3. Create at least one VM in Talos

### Start the cluster

1. Create a Talos secret `talosctl gen secrets -o ./demo-secrets.yaml`
2. Create a declarative config for our Controlplane.
    This config deactives the standard CNI, so we can use Cilium later. It also mounts an additional disk to `/var/mnt/storage` for our Persistent Volumes.

    ```yaml
    # controlplane.yaml
    machine:
      install:
        image: ghcr.io/siderolabs/installer:v1.11.2
      env:
        https_proxy: someProxy
        http_proxy: someProxy
        no_proxy: '0.0.0.0/32,10.0.0.0/8,192.168.0.0/16,172.17.0.0/16,172.18.0.0/16,172.19.0.0/16,127.0.0.0,.domain.local,.domain2.local,localhost,.cluster.local'
      disks:
      - device: /dev/sdb
        partitions:
          - mountpoint: /var/mnt/storage
      kubelet:
        extraMounts:
        - destination: /var/mnt/storage
          type: bind
          source: /var/mnt/storage
          options:
            - bind
            - rshared
            - rw
    cluster:
      network:
        cni:
          name: none
      proxy:
        disabled: true
      allowSchedulingOnControlPlanes: true
    ````

3. And for our workers

    ```yaml
    worker.yaml

    machine:
      install:
        image: ghcr.io/siderolabs/installer:v1.11.2
      env:
        https_proxy: someProxy
        http_proxy: someProxy
        no_proxy: '0.0.0.0/32,10.0.0.0/8,192.168.0.0/16,172.17.0.0/16,172.18.0.0/16,172.19.0.0/16,127.0.0.0,.domain.local,.domain2.local,localhost,.cluster.local'
      disks:
      - device: /dev/sdb
        partitions:
          - mountpoint: /var/mnt/storage
      kubelet:
        extraMounts:
        - destination: /var/mnt/storage
          type: bind
          source: /var/mnt/storage
          options:
            - bind
            - rshared
            - rw
    ```

4. Optional: Create patches, to include specific manifests in the machine config, for example to install Cilium as CNI. In a default Talos installation you would have a default CNI, we just like Cilium more. It would also be possible to install it later via `kubectl apply -f ...`, or `helm install ...`, but this way we have it right from the start, delivered via our Infrastructure as Code.

    ```bash
    VERSION=1.18.2 && \
    (echo -e "cluster:\n  inlineManifests:\n    - name: cilium\n      contents: |" && \
    helm template cilium cilium/cilium --version $VERSION \
      --namespace kube-system \
      --set ipam.mode=kubernetes \
      --set cni.exclusive=false \
      --set kubeProxyReplacement=true \
      --set securityContext.capabilities.ciliumAgent="{CHOWN,KILL,NET_ADMIN,NET_RAW,IPC_LOCK,SYS_ADMIN,SYS_RESOURCE,DAC_OVERRIDE,FOWNER,SETGID,SETUID}" \
      --set securityContext.capabilities.cleanCiliumState="{NET_ADMIN,SYS_ADMIN,SYS_RESOURCE}" \
      --set cgroup.autoMount.enabled=false \
      --set cgroup.hostRoot=/sys/fs/cgroup \
      --set k8sServiceHost=localhost \
      --set k8sServicePort=7445 | sed 's/^/        /') > patches/cni.cilium.yaml
    ```

5. Then we can generate the machine configs

    ```bash
    export CONTROL_PLANE_IP="x.x.x.x"
    export TALOSCONFIG="_out/talosconfig"

    talosctl gen config seilerre-talos-demo https://$CONTROL_PLANE_IP:6443 \
    --with-secrets ./demo-secrets.yaml \
    --config-patch-control-plane @controlplane.yaml \
    --config-patch-control-plane @patches/cni.cilium.yaml \
    --config-patch-worker @worker.yaml \
    --force \
    --output-dir _out
    ```

6. Apply the config to the machines

    ```bash
    talosctl apply-config --insecure \
    --nodes $CONTROL_PLANE_IP \
    --file _out/controlplane.yaml
    ```

7. Configure talosctl to use the new cluster and bootstrap the cluster, as this was the first Control Plane node

    ```bash
    talosctl config endpoint $CONTROL_PLANE_IP
    talosctl config node $CONTROL_PLANE_IP

    talosctl bootstrap
    ```

    We can merge the kubeconfig using `talosctl kubeconfig`.

    or `talosctl help`

    Some useful commands

    ```bash
    talosctl get disks
    talosctl ls /
    talosctl dmesg
    talosctl get sboms
    ```

8. Lets add a worker node

    ```bash
    export WORKER_IP="x.x.x.x"

    talosctl apply-config --insecure \
    --nodes $WORKER_IP \
    --file _out/worker.yaml
    ```

    We can see it connecting via `kubectl get nodes` after a few moments.
