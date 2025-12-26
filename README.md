# Talos Linux Kubernetes Cluster Bootstrap

```bash
# Generate Config
talosctl gen config talos-cluster https://10.0.203.100:6443 --output-dir _out

# Apply Config
for node in 10.0.203.11 10.0.203.12 10.0.203.13; do
  talosctl apply-config --insecure --nodes $node \
    --file _out/controlplane.yaml --config-patch @bootstrap/talos/controlplane.yaml
done

for node in 10.0.203.21 10.0.203.22 10.0.203.23; do
  talosctl apply-config --insecure --nodes $node \
    --file _out/worker.yaml --config-patch @bootstrap/talos/worker.yaml
done

# Bootstrap (once)
export TALOSCONFIG="_out/talosconfig"
talosctl config endpoint 10.0.203.11
talosctl bootstrap
talosctl kubeconfig _out/kubeconfig
```
