# Talos Linux Installation

This guide walks you through installing and configuring Talos Linux, including generating secrets, creating configuration files, deploying the control plane, bootstrapping the cluster, exporting kubeconfig, deploying the Cilium CNI, and upgrading the cluster.

## Prerequisites

- `talosctl` command-line tool installed.
- `helm` command-line tool installed.
- DNS setup:
  - `talos-api.whezzel.com` is an A record pointing to the Talos VIP (`10.69.20.10`).
  - Each node has its own A record.

## Download and Boot the SecureBoot ISO

For the initial installation, download the SecureBoot ISO from the Talos image factory. The basic secureboot+uefi-only configuration (without any system extensions) uses the image ID `80469ef645425bc7cb27830a7143999e422fd7b206c9d9726c47de47800e956d`. Since this is just the boot ISO, it doesn't require special configuration.

> [!NOTE]
> If you need additional system extensions to support specific hardware, generate a new image ID at <https://factory.talos.dev/>.

Download the ISO:

```sh
VERSION=<VERSION>
IMAGE_ID=80469ef645425bc7cb27830a7143999e422fd7b206c9d9726c47de47800e956d
wget -c "https://factory.talos.dev/image/${IMAGE_ID}/v${VERSION}/metal-amd64-secureboot.iso" -O "metal-amd64-secureboot-v${VERSION}.iso"
```

> [!NOTE]
> Replace `<VERSION>` with the version number of Talos you intend to install, e.g., 1.10.6.

Boot each node from this ISO to start Talos in memory mode before applying configurations.

## Step 1: Generate Secrets

Generate secrets for Talos Linux:

```sh
talosctl gen secrets --output-file secrets.yaml
```

## Step 2: Generate Configuration Files

Create configuration files for the cluster:

```sh
talosctl gen config prod \
    --with-secrets secrets.yaml \
    https://talos-api.whezzel.com:6443 \
    --force --output ~/repos/talos-configs
```

Move the `talosconfig` file to the correct location:

```sh
mkdir --parent ~/.talos
cp talosconfig ~/.talos/config
```

## Step 3: Generate Machine Configuration Files

Patch machine configurations for each node:

```sh
t machineconfig patch controlplane.yaml --patch @patches/shared.yaml --patch @patches/shared-controlplane.yaml --patch @patches/talos01.yaml --output talos01.yaml
t machineconfig patch controlplane.yaml --patch @patches/shared.yaml --patch @patches/shared-controlplane.yaml --patch @patches/talos02.yaml --output talos02.yaml
t machineconfig patch controlplane.yaml --patch @patches/shared.yaml --patch @patches/shared-controlplane.yaml --patch @patches/talos03.yaml --output talos03.yaml
t machineconfig patch worker.yaml --patch @patches/shared.yaml --patch @patches/shared-worker.yaml --patch @patches/talos04.yaml --output talos04.yaml
t machineconfig patch worker.yaml --patch @patches/shared.yaml --patch @patches/shared-worker.yaml --patch @patches/talos05.yaml --output talos05.yaml
t machineconfig patch worker.yaml --patch @patches/shared.yaml --patch @patches/shared-worker.yaml --patch @patches/talos06.yaml --output talos06.yaml
t machineconfig patch worker.yaml --patch @patches/shared.yaml --patch @patches/shared-worker.yaml --patch @patches/shared-worker-nvidia.yaml --patch @patches/talos07.yaml --output talos07.yaml
t machineconfig patch worker.yaml --patch @patches/shared.yaml --patch @patches/shared-worker.yaml --patch @patches/shared-worker-nvidia.yaml --patch @patches/talos08.yaml --output talos08.yaml
```

## Step 4: Deploy the First Control Plane Node

Since the node lacks a certificate during initial deployment, use the `--insecure` flag to bypass authentication:

```sh
talosctl apply-config \
    --nodes talos01 \
    --file talos01.yaml \
    --insecure \
    --endpoints 10.69.20.11
```

> [!NOTE]
> The `--insecure` flag is required for the initial deployment because the node does not yet have a certificate, and authentication would otherwise fail. The VIP (`talos-api.whezzel.com`) is not active until the cluster is bootstrapped, so the node's IP (`10.69.20.11`) is used as the endpoint.

## Step 5: Bootstrap the Cluster

Bootstrap the cluster using the first control plane node:

```sh
talosctl bootstrap \
    --nodes talos01 \
    --endpoints 10.69.20.11
```

## Step 6: Configure the Talos API Endpoint

Now that the cluster has been bootstrapped, set the endpoint to the Talos API VIP:

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

```sh
helm upgrade --install cilium cilium \
    --repo https://helm.cilium.io \
    --namespace cilium \
    --create-namespace \
    --values ./helm/cilium/values.yaml
```

## Post-Deployment

After deploying Cilium, your cluster is ready for workloads. For load balancer services via BGP, refer to the Cilium documentation: <https://docs.cilium.io/en/stable/installation/k8s-install-helm/>.

# Upgrading Talos Linux

To upgrade Talos Linux, follow these steps to ensure a smooth and safe process. Perform upgrades one node at a time to maintain cluster availability.

### Step 1: Verify Image IDs

Check the image IDs for your nodes to ensure they match the desired version and configuration.

For ARM-based control plane nodes (talos01, talos02, talos03):

```sh
curl -s -X POST --data-binary @customization/arm.yaml https://factory.talos.dev/schematics | jq -r '.id'
```

For AMD64-based worker nodes (talos04, talos05, talos06):

```sh
curl -s -X POST --data-binary @customization/longhorn.yaml https://factory.talos.dev/schematics | jq -r '.id'
```

For AMD64-based worker nodes with Nvidia GPU and Coral TPU (talos07, talos08):

```sh
curl -s -X POST --data-binary @customization/longhorn-coral-nvidia.yaml https://factory.talos.dev/schematics | jq -r '.id'
```

> [!NOTE]
> Ensure the `customization/*.yaml` files exist and contain the correct system extensions for your nodes. Update them if necessary to match the target Talos version.

### Step 2: Update Patch Files

Review and update the patch files (`./patches/shared-controlplane.yaml`, `./patches/shared-worker.yaml`, `./patches/shared-worker-nvidia.yaml`) to include the correct image IDs and any other required changes for the new Talos version.

### Step 3: Regenerate Configuration Files

Regenerate the configuration files for the cluster:

```sh
talosctl gen config prod \
    --with-secrets secrets.yaml \
    https://talos-api.whezzel.com:6443 \
    --force --output ~/repos/talos-configs
```

Edit the generated `controlplane.yaml` and `worker.yaml` files to comment out the `install` section, as custom installation details are defined in the patch files:

```diff
--- controlplane.yaml
+++ controlplane.yaml
@@ -187,10 +187,10 @@
     #     enabled: true # Enable the KubeSpan feature.
 
     # Used to provide instructions for installations.
-    install:
-        disk: /dev/sda # The disk used for installations.
-        image: ghcr.io/siderolabs/installer:<version> # Allows for supplying the image used to perform the installation.
-        wipe: false # Indicates if the installation disk should be wiped at installation time.
+#    install:
+#        disk: /dev/sda # The disk used for installations.
+#        image: ghcr.io/siderolabs/installer:<version> # Allows for supplying the image used to perform the installation.
+#        wipe: false # Indicates if the installation disk should be wiped at installation time.
```

```diff
--- worker.yaml
+++ worker.yaml
@@ -187,10 +187,10 @@
     #     enabled: true # Enable the KubeSpan feature.
 
     # Used to provide instructions for installations.
-    install:
-        disk: /dev/sda # The disk used for installations.
-        image: ghcr.io/siderolabs/installer:<version> # Allows for supplying the image used to perform the installation.
-        wipe: false # Indicates if the installation disk should be wiped at installation time.
+#    install:
+#        disk: /dev/sda # The disk used for installations.
+#        image: ghcr.io/siderolabs/installer:<version> # Allows for supplying the image used to perform the installation.
+#        wipe: false # Indicates if the installation disk should be wiped at installation time.
```

### Step 4: Regenerate Machine Configuration Files

Reapply patches to generate updated machine configuration files for each node, as described in [Step 3: Generate Machine Configuration Files](#step-3-generate-machine-configuration-files).

### Step 5: Upgrade Nodes

Upgrade each node one at a time, replacing `{image_factory_id}` with the appropriate image ID from Step 1 and `{version}` with the target Talos version (e.g., `1.11.1`):

```sh
t upgrade --drain=false -n talos01 -i factory.talos.dev/metal-installer/$(curl -s -X POST --data-binary @customization/arm.yaml https://factory.talos.dev/schematics | jq -r '.id'):v{version}
t upgrade --drain=false -n talos02 -i factory.talos.dev/metal-installer/$(curl -s -X POST --data-binary @customization/arm.yaml https://factory.talos.dev/schematics | jq -r '.id'):v{version}
t upgrade --drain=false -n talos03 -i factory.talos.dev/metal-installer/$(curl -s -X POST --data-binary @customization/arm.yaml https://factory.talos.dev/schematics | jq -r '.id'):v{version}
t upgrade --drain=false -n talos04 -i factory.talos.dev/metal-installer-secureboot/$(curl -s -X POST --data-binary @customization/longhorn.yaml https://factory.talos.dev/schematics | jq -r '.id'):v{version}
t upgrade --drain=false -n talos05 -i factory.talos.dev/metal-installer-secureboot/$(curl -s -X POST --data-binary @customization/longhorn.yaml https://factory.talos.dev/schematics | jq -r '.id'):v{version}
t upgrade --drain=false -n talos06 -i factory.talos.dev/metal-installer-secureboot/$(curl -s -X POST --data-binary @customization/longhorn.yaml https://factory.talos.dev/schematics | jq -r '.id'):v{version}
t upgrade --drain=false -n talos07 -i factory.talos.dev/metal-installer-secureboot/$(curl -s -X POST --data-binary @customization/longhorn-coral-nvidia.yaml https://factory.talos.dev/schematics | jq -r '.id'):v{version}
t upgrade --drain=false -n talos08 -i factory.talos.dev/metal-installer-secureboot/$(curl -s -X POST --data-binary @customization/longhorn-coral-nvidia.yaml https://factory.talos.dev/schematics | jq -r '.id'):v{version}
```

> [!NOTE]
> Monitor each node's upgrade process using `talosctl get members` to ensure the node rejoins the cluster successfully before proceeding to the next node.

### Step 6: Upgrade Kubernetes

Perform a dry run to preview the Kubernetes upgrade changes:

```sh
talosctl upgrade-k8s --nodes talos01 --dry-run
```

If the dry run output is satisfactory, proceed with the Kubernetes upgrade:

```sh
talosctl upgrade-k8s --nodes talos01
```

> [!NOTE]
> The Kubernetes upgrade should be performed on a control plane node (e.g., talos01). Ensure all nodes are upgraded before running the Kubernetes upgrade.

# Troubleshooting

## Reset Node

To reset a node:

```sh
talosctl --nodes <node> reset --system-labels-to-wipe EPHEMERAL,STATE --reboot --graceful=false
```

## Prune Images

### Step 1: Create debug container

```sh
kubectl debug -n kube-system -it --image ghcr.io/whezzel/alpine-tools --profile=sysadmin node/<node>
```

### Step 2: Install tools
#### Not necessary if using my custom "alpine-tools" images since this package is already installed.

```sh
apk add cri-tools
```

### Step 3: Set runtime endpoint
#### Not necessary if using my custom "alpine-tools" images since this env is already set.

```sh
export CONTAINER_RUNTIME_ENDPOINT=unix:///host/run/containerd/containerd.sock
```

### Step 4: Prune Images

```sh
crictl rmi --prune
```
