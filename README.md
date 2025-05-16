# Talos Linux Installation

> - `talos-api.whezzel.com` is an A record that points to the talos vip (10.69.20.10)
> - each node has an A record as well

## Preparation

Generate the secrets:

```sh
talosctl gen secrets -o ~/repos/talos/secrets.yaml
```

Generate config files:

```sh
talosctl gen config --with-secrets secrets.yaml cluster https://talos-api.whezzel.com:6443
```

Copy talosconfig to appropriate location:

```sh
mkdir ~/.talos
cp ~/repos/talos/talosconfig ~/.config/talos/config
```

Add endpoint to talosconfig

```sh
talosctl config endpoint talos-api.whezzel.com
```

Generate machine config files:

```sh
talosctl machineconfig patch worker.yaml -p @patches/all.yaml -p @patches/longhorn-coral-nvidia.yaml -p @patches/talos01.yaml -o talos01.yaml
talosctl machineconfig patch worker.yaml -p @patches/all.yaml -p @patches/longhorn-coral-nvidia.yaml -p @patches/talos02.yaml -o talos02.yaml
talosctl machineconfig patch controlplane.yaml -p @patches/all.yaml -p @patches/controlplane-worker.yaml -p @patches/longhorn.yaml -p @patches/talos03.yaml -o talos03.yaml
talosctl machineconfig patch controlplane.yaml -p @patches/all.yaml -p @patches/controlplane-worker.yaml -p @patches/longhorn.yaml -p @patches/talos04.yaml -o talos04.yaml
talosctl machineconfig patch controlplane.yaml -p @patches/all.yaml -p @patches/controlplane-worker.yaml -p @patches/longhorn.yaml -p @patches/talos05.yaml -o talos05.yaml
```

## Deploy Talos Linux

Deploy control-plane (you only need `--insecure` for the initial deployment):

```sh
talosctl apply-config -n talos01 --file ~/repos/talos/talos01.yaml --insecure
talosctl apply-config -n talos02 --file ~/repos/talos/talos02.yaml --insecure
talosctl apply-config -n talos03 --file ~/repos/talos/talos03.yaml --insecure
talosctl apply-config -n talos04 --file ~/repos/talos/talos04.yaml --insecure
talosctl apply-config -n talos05 --file ~/repos/talos/talos05.yaml --insecure
```

Bootstrap cluser:

```sh
talosctl bootstrap -n talos05 -e talos05
```

Export kubeconfig:

```sh
talosctl kubeconfig -m ~/.kube/config
```

# Configure Kubernetes

## Deploy Multus

- https://www.talos.dev/v1.7/kubernetes-guides/network/multus/

Download copy of multus thick daemonset:

```sh
wget https://raw.githubusercontent.com/k8snetworkplumbingwg/multus-cni/master/deployments/multus-daemonset-thick.yml
```

Edit `multus-daemonset-thick.yml` and change the volume `host-run-netns` from `/run/netns/` to `/var/run/netns/`:

```diff
        - name: host-run-netns
          hostPath:
-           path: /run/netns/
+           path: /var/run/netns/
```

Deploy multus and the NetworkAttachmentDefinition:

```sh
kubectl apply -f ~/repos/talos/multus/multus-daemonset-thick.yml
kubectl apply -f ~/repos/talos/multus/cni_conf.yaml
```

## Deploy Cert-Manager

 - https://cert-manager.io/docs/installation/helm/

Deploy Cert Manager using helm:

```sh
helm upgrade --install cert-manager cert-manager \
    --repo https://charts.jetstack.io \
    --namespace cert-manager \
    --create-namespace \
    --set crds.enabled=true \
    --set 'extraArgs={--dns01-recursive-nameservers-only,--dns01-recursive-nameservers=8.8.8.8:53\,1.1.1.1:53}'
```

Apply your issuers, and certs from local manifests:

```sh
kubectl apply -f ~/repos/talos/helm/cert-manager
```

## Deploy ingress-nginx

 - https://kubernetes.github.io/ingress-nginx/deploy/

Deploy ingress-nginx using helm:

```sh
helm upgrade --install nginx ingress-nginx \
    --repo https://kubernetes.github.io/ingress-nginx \
    --namespace nginx \
    --create-namespace \
    --values ~/repos/talos/helm/nginx/values.yaml
```

## Deploy keel.sh

- https://keel.sh/docs/#deploying-with-helm

Deploy keel.sh using helm:

```sh
helm upgrade --install keel keel \
    --repo https://charts.keel.sh \
    --namespace=keel \
    --create-namespace \
    --set "nodeSelector\.kubernetes\.io\/arch"=amd64 \
    --values ~/repos/talos/helm/keel/values.yaml
```

## Deploy Longhorn

- https://longhorn.io/docs/1.6.2/advanced-resources/os-distro-specific/talos-linux-support/
- https://longhorn.io/docs/1.6.2/deploy/important-notes/#pod-security-policies-disabled--pod-security-admission-introduction
- https://www.talos.dev/v1.6/kubernetes-guides/configuration/pod-security/

Create namespace with needed labels to satisfy pod security requirements:

```sh
kubectl apply -f ~/repos/talos/namespaces/longhorn-system.yaml
```

Deploy Longhorn using helm:

```sh
helm repo add longhorn https://charts.longhorn.io
helm repo update
helm install longhorn longhorn/longhorn \
    --namespace longhorn-system \
    --values ~/repos/talos/helm/longhorn/values.yaml
```
