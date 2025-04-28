# Requirements

## Kubevirt 

```shell
export RELEASE=$(curl https://storage.googleapis.com/kubevirt-prow/release/kubevirt/kubevirt/stable.txt)
kubectl apply -f https://github.com/kubevirt/kubevirt/releases/download/${RELEASE}/kubevirt-operator.yaml
kubectl apply -f https://github.com/kubevirt/kubevirt/releases/download/${RELEASE}/kubevirt-cr.yaml

kubectl -n kubevirt wait kv kubevirt --for condition=Available
```

## CDI

```shell
export VERSION=$(curl -s https://api.github.com/repos/kubevirt/containerized-data-importer/releases/latest | grep '"tag_name":' | sed -E 's/.*"([^"]+)".*/\1/')
kubectl create -f https://github.com/kubevirt/containerized-data-importer/releases/download/$VERSION/cdi-operator.yaml
kubectl create -f https://github.com/kubevirt/containerized-data-importer/releases/download/$VERSION/cdi-cr.yaml
```

## Longhorn

```shell
kubectl apply -f https://raw.githubusercontent.com/longhorn/longhorn/v1.8.1/deploy/longhorn.yaml
```

## Clusterctl

```shell
export VERSION=$(curl -s https://api.github.com/repos/kubernetes-sigs/cluster-api/releases/latest | grep '"tag_name":' | sed -E 's/.*"([^"]+)".*/\1/')

curl -SLO https://github.com/kubernetes-sigs/cluster-api/releases/download/${VERSION}$/clusterctl-linux-amd64
chmod +x clusterctl-linux-amd64
sudo mv clusterctl-linux-amd64 /usr/local/bin/clusterctl

clusterctl version # test if install is ok
```

# Init CAPI

```shell
clusterctl init --ipam in-cluster --control-plane talos --bootstrap talos  --infrastructure kubevirt --addon helm
```

# Add Talos VirtualMachineClusterPreference

```shell
kubectl apply -f talosVMCP.yaml     
```

# Set up cluster

## Generate yamls

```shell
export INSTANCE_PREFERENCE=talos
export INSTANCE_TYPE=o1.xlarge # see kubectl get VirtualMachineClusterPreference for more instance_types
export ROOT_VOLUME_SIZE=10G
export STORAGE_CLASS_NAME=longhorn # Can not be localpath because of CDI importer
export TALOS_VERSION=v1.9.5
export TALOS_IMAGE_URL=https://factory.talos.dev/image/376567988ad370138ad8b2698212367b8edcb69b5fd68c80be1f2ec7d603b4ba/v1.9.5/openstack-amd64.raw.xz
export TALOS_CODE="t${TALOS_VERSION//[^0-9]/}"
export POD_CIDR_BLOCK=10.243.0.0/16
export SVC_CIDR_BLOCK=10.95.0.0/16
export CAPK_GUEST_K8S_VERSION="v1.32.1"

clusterctl generate cluster talos-cilium --kubernetes-version ${CAPK_GUEST_K8S_VERSION} --control-plane-machine-count=1  --worker-machine-count=1   --from ./cluster-template-cip-talos-cilium-ebpf-cdi.yaml  > talos-cilium.yaml
```

```shell
kubectl apply -f talos-cilium.yaml
```

```shell
clusterctl describe cluster capi-quickstart
```

## special tips

### wsl2 for Kubervirt
```
sudo mount --make-shared /
```

### k3s longhorn

```
sudo cp /var/lib/rancher/k3s/agent/etc/containerd/config.toml /var/lib/rancher/k3s/agent/etc/containerd/config-v3.toml.tmpl 
```

Edit `device_ownership_from_security_context` in /var/lib/rancher/k3s/agent/etc/containerd/config-v3.toml.tmpl and set to true

```toml
[plugins.'io.containerd.cri.v1.runtime']
...
  device_ownership_from_security_context = true
```

Then restart k3s via `sudo systemctl restart  k3s`
