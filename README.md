# Talos Linux Kubernetes Cluster Bootstrap

## 1. Bootstrap Cluster

```bash
VIP="10.0.203.100"
CONTROLPLANE_NODES=(
  10.0.203.11
  10.0.203.12
  10.0.203.13
)
WORKER_NODES=(
  10.0.203.21
  10.0.203.22
  10.0.203.23
)

talosctl gen config talos-cluster https://${VIP}:6443 --output-dir _out

for node in "${CONTROLPLANE_NODES[@]}"; do
  talosctl apply-config --insecure --nodes $node \
    --file _out/controlplane.yaml --config-patch @bootstrap/talos/controlplane.yaml
done

for node in "${WORKER_NODES[@]}"; do
  talosctl apply-config --insecure --nodes $node \
    --file _out/worker.yaml --config-patch @bootstrap/talos/worker.yaml
done

export TALOSCONFIG="_out/talosconfig"
talosctl config endpoint ${CONTROLPLANE_NODES[0]}
talosctl bootstrap
talosctl kubeconfig _out/kubeconfig
```

## 2. Install CNI & Applications

```bash
export KUBECONFIG="_out/kubeconfig"

# Cilium CNI
helm repo add cilium https://helm.cilium.io/
helm install cilium cilium/cilium --version 1.18.5 \
  --namespace kube-system \
  --set ipam.mode=kubernetes \
  --set kubeProxyReplacement=true \
  --set l2announcements.enabled=true \
  --set securityContext.capabilities.ciliumAgent="{CHOWN,KILL,NET_ADMIN,NET_RAW,IPC_LOCK,SYS_ADMIN,SYS_RESOURCE,DAC_OVERRIDE,FOWNER,SETGID,SETUID}" \
  --set securityContext.capabilities.cleanCiliumState="{NET_ADMIN,SYS_ADMIN,SYS_RESOURCE}" \
  --set cgroup.autoMount.enabled=false \
  --set cgroup.hostRoot=/sys/fs/cgroup \
  --set k8sServiceHost=localhost \
  --set k8sServicePort=7445 \
  --set gatewayAPI.enabled=true

# Longhorn Storage
helm repo add longhorn https://charts.longhorn.io
helm install longhorn longhorn/longhorn --version 1.7.2 \
  --namespace longhorn-system --create-namespace \
  --set defaultSettings.defaultDataPath=/var/mnt/longhorn

# ArgoCD
helm repo add argo https://argoproj.github.io/argo-helm
helm install argocd argo/argo-cd --version 2.13.2 \
  --namespace argocd --create-namespace \
  --set configs.params."server\.insecure"=true
```
