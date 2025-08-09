# Talos Linux Installation

This guide walks you through installing and configuring Talos Linux, including generating secrets, creating configuration files, deploying the control plane, bootstrapping the cluster, exporting kubeconfig, and deploying the Cilium CNI.

## Prerequisites

- `talosctl` command-line tool installed.
- `helm` command-line tool installed.
- DNS setup:
  - `talos-api.whezzel.com` is an A record pointing to the Talos VIP (`10.69.20.10`).
  - Each node has its own A record.

## Download and Boot the SecureBoot ISO

For the initial installation, download the SecureBoot ISO from the Talos image factory. The basic configuration (without any system extensions or other config options) uses the image ID `376567988ad370138ad8b2698212367b8edcb69b5fd68c80be1f2ec7d603b4ba`. Since this is just the boot ISO, it doesn't require special configuration.

> [!NOTE]
> If you need additional system extensions to support specific hardware, generate a new image ID at <https://factory.talos.dev/>.

Download the ISO:

```sh
VERSION=<VERSION>
IMAGE_ID=376567988ad370138ad8b2698212367b8edcb69b5fd68c80be1f2ec7d603b4ba
wget -c https://factory.talos.dev/image/${IMAGE_ID}/v${VERSION}/metal-amd64-secureboot.iso -O metal-amd64-secureboot-v${VERSION}.iso
```

> [!NOTE]
> Replace `<VERSION>` with the version number of Talos you intend to install, e.g., 1.10.6.

Boot each node from this ISO to start Talos in memory mode before applying configurations.

## Step 1: Generate Secrets

Generate secrets for Talos Linux:

```sh
talosctl gen secrets -o secrets.yaml
```

## Step 2: Generate Configuration Files

Create configuration files for the cluster:

```sh
talosctl gen config prod \
    --with-secrets secrets.yaml \
    https://talos-api.whezzel.com:6443 \
    --nodes talos01.whezzel.com \
    --nodes talos02.whezzel.com \
    --nodes talos03.whezzel.com \
    --nodes talos04.whezzel.com \
    --nodes talos05.whezzel.com \
    --nodes talos06.whezzel.com \
    --nodes talos07.whezzel.com \
    --nodes talos08.whezzel.com \
    --force -o ~/repos/talos
```

Move the `talosconfig` file to the correct location:

```sh
mkdir -p ~/.talos
cp talosconfig ~/.talos/config
```

## Step 3: Generate Machine Configuration Files

Patch machine configurations for each node:

```sh
talosctl machineconfig patch controlplane.yaml \
    -p @patches/shared.yaml \
    -p @patches/shared-controlplane.yaml \
    -p @patches/talos01.yaml \
    -o talos01.yaml
talosctl machineconfig patch controlplane.yaml \
    -p @patches/shared.yaml \
    -p @patches/shared-controlplane.yaml \
    -p @patches/talos02.yaml \
    -o talos02.yaml
talosctl machineconfig patch controlplane.yaml \
    -p @patches/shared.yaml \
    -p @patches/shared-controlplane.yaml \
    -p @patches/talos03.yaml \
    -o talos03.yaml
talosctl machineconfig patch worker.yaml \
    -p @patches/shared.yaml \
    -p @patches/shared-worker.yaml \
    -p @patches/talos04.yaml \
    -o talos04.yaml
talosctl machineconfig patch worker.yaml \
    -p @patches/shared.yaml \
    -p @patches/shared-worker.yaml \
    -p @patches/talos05.yaml \
    -o talos05.yaml
talosctl machineconfig patch worker.yaml \
    -p @patches/shared.yaml \
    -p @patches/shared-worker.yaml \
    -p @patches/talos06.yaml \
    -o talos06.yaml
talosctl machineconfig patch worker.yaml \
    -p @patches/shared.yaml \
    -p @patches/shared-worker.yaml \
    -p @patches/shared-worker-nvidia.yaml \
    -p @patches/talos07.yaml \
    -o talos07.yaml
talosctl machineconfig patch worker.yaml \
    -p @patches/shared.yaml \
    -p @patches/shared-worker.yaml \
    -p @patches/shared-worker-nvidia.yaml \
    -p @patches/talos08.yaml \
    -o talos08.yaml
```

## Step 4: Deploy the Control Plane

Since the node lacks a certificate during initial deployment, use the `--insecure` flag to bypass authentication:

```sh
talosctl apply-config \
    -n talos01 \
    --file talos01.yaml \
    --insecure \
    --endpoint 10.69.20.11
```

> [!NOTE]
> The `--insecure` flag is required for the initial deployment because the node does not yet have a certificate, and authentication would otherwise fail. The VIP (`talos-api.whezzel.com`) is not active until the cluster is bootstrapped, so the node's IP (`10.69.20.11`) is used as the endpoint.

## Step 5: Bootstrap the Cluster

Bootstrap the cluster using one of the nodes:

```sh
talosctl bootstrap -n talos01 -e 10.69.20.11
```

## Step 6: Configure the Talos API Endpoint

Now that the cluster has been bootstrapped, we can set the endpoint to the talos-api VIP.

Set the Talos API endpoint:

```sh
talosctl config endpoint talos-api.whezzel.com
```

## Step 7: Apply Remaining Node Configurations

Apply configurations to the remaining nodes (use `--insecure` as certificates are not yet issued):

```sh
talosctl apply-config -n talos02 --file talos02.yaml --insecure
talosctl apply-config -n talos03 --file talos03.yaml --insecure
talosctl apply-config -n talos04 --file talos04.yaml --insecure
talosctl apply-config -n talos05 --file talos05.yaml --insecure
talosctl apply-config -n talos06 --file talos06.yaml --insecure
talosctl apply-config -n talos07 --file talos07.yaml --insecure
talosctl apply-config -n talos08 --file talos08.yaml --insecure
```

## Step 8: Export Kubeconfig

Export the kubeconfig to interact with the cluster:

```sh
talosctl kubeconfig ~/.kube/config
```

## Step 9: Deploy Cilium CNI

With Flannel CNI disabled, deploy Cilium as the CNI using the Helm chart.

Create a `values.yaml` file for Cilium:

```yaml
cni:
  exclusive: false
bgpControlPlane:
  enabled: true
cgroup:
  autoMount:
    enabled: false
  hostRoot: "/sys/fs/cgroup"
ipam:
  mode: "kubernetes"
k8sServiceHost: "localhost"
k8sServicePort: "7445"
kubeProxyReplacement: true
kubeProxyReplacementHealthzBindAddr: "0.0.0.0:10256"
securityContext:
  capabilities:
    ciliumAgent: ["CHOWN","KILL","NET_ADMIN","NET_RAW","IPC_LOCK","SYS_ADMIN","SYS_RESOURCE","DAC_OVERRIDE","FOWNER","SETGID","SETUID"]
    cleanCiliumState: ["NET_ADMIN","SYS_ADMIN","SYS_RESOURCE"]
image:
  pullPolicy: "IfNotPresent"
```

Deploy Cilium:

```bash
helm upgrade --install cilium cilium \
    --repo https://helm.cilium.io \
    --namespace cilium \
    --create-namespace \
    --values ./helm/cilium/values.yaml
```

## Post-Deployment

After deploying Cilium, your cluster is ready for workloads. For load balancer services via BGP, refer to the Cilium documentation: <https://docs.cilium.io/en/stable/installation/k8s-install-helm/>.
