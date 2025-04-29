# CAPI easy via Kubevirt

This repo contains cookbooks for starting Kubernetes via Cluster API using Kubevirt.

Supported platforms:
  - [x] Talos
  - [ ] Ubuntu (tested ok, need to commit)
  - [ ] Flatcar (WIP)

# Plaftorm Requirements

To test this repo, you need a working Kubernetes running on a Linux.

You also need your Kubernetes nodes support, see this [StackOverflow answer](https://stackoverflow.com/questions/11116704/check-if-vt-x-is-activated-without-having-to-reboot-in-linux#answer-51272679) to know how to test this.

You'll also need cUrl and kubectl of course.

## Kubevirt

Kubevirt is the tool that allows you to run virtual machines in Kubernetes. Installation is straight forward:
```shell
export RELEASE=$(curl https://storage.googleapis.com/kubevirt-prow/release/kubevirt/kubevirt/stable.txt)
kubectl apply -f https://github.com/kubevirt/kubevirt/releases/download/${RELEASE}/kubevirt-operator.yaml
kubectl apply -f https://github.com/kubevirt/kubevirt/releases/download/${RELEASE}/kubevirt-cr.yaml

kubectl -n kubevirt wait kv kubevirt --for condition=Available
```

## CDI

CDI stands for Containerized Data Importer. It's a tool made by the Kubevirt team that allows common virtualization images (iso, qcow, ova, ...) to be imported when creating virtual machines. Install it for later use of vendors images:
```shell
export VERSION=$(curl -s https://api.github.com/repos/kubevirt/containerized-data-importer/releases/latest | grep '"tag_name":' | sed -E 's/.*"([^"]+)".*/\1/')
kubectl create -f https://github.com/kubevirt/containerized-data-importer/releases/download/$VERSION/cdi-operator.yaml
kubectl create -f https://github.com/kubevirt/containerized-data-importer/releases/download/$VERSION/cdi-cr.yaml
```

## Longhorn

Longhorn is a distributed storage, usefull when creating VMs you may want to move from one node to another. Installation is also very simple:
```shell
export VERSION=$(curl -s https://api.github.com/repos/longhorn/longhorn/releases/latest | grep '"tag_name":' | sed -E 's/.*"([^"]+)".*/\1/')
kubectl apply -f https://raw.githubusercontent.com/longhorn/longhorn/${VERSION}$/deploy/longhorn.yaml
```

## Clusterctl

Clusterctl is the command line we'll use to generate CAPI clusters. Install via this shell script:
```shell
export VERSION=$(curl -s https://api.github.com/repos/kubernetes-sigs/cluster-api/releases/latest | grep '"tag_name":' | sed -E 's/.*"([^"]+)".*/\1/')

curl -SLO https://github.com/kubernetes-sigs/cluster-api/releases/download/${VERSION}$/clusterctl-linux-amd64
chmod +x clusterctl-linux-amd64
sudo mv clusterctl-linux-amd64 /usr/local/bin/clusterctl

clusterctl version # test if install is ok
```

## System

In order to avoid getting messages like "too many open files" or same with watchers, it is recommended to improve sysctl config: 
```shell
sudo sysctl -w fs.inotify.max_user_watches=2099999999
sudo sysctl -w fs.inotify.max_user_instances=2099999999
sudo sysctl -w fs.inotify.max_queued_events=2099999999
```

# Init CAPI

Before creating a cluster, we need to use clusterctl to init the different providers we'll use:
```shell
clusterctl init --ipam in-cluster --control-plane talos --bootstrap talos  --infrastructure kubevirt --addon helm
```

# Prepare cluster

## Add Talos VirtualMachineClusterPreference

Current list of VMClusterPreference does not contain any Talos definition, let's add it:

```shell
kubectl apply -f talosVMCP.yaml
```

# Create cluster

Generate yamls:
```shell
export INSTANCE_PREFERENCE=talos
export INSTANCE_TYPE=o1.xlarge # see kubectl get VirtualMachineClusterPreference for more instance_types
export ROOT_VOLUME_SIZE=10G
export CONTROL_PLANE_SVC_TYPE=ClusterIP # LoadBalancer, ClusterIP or NodePort
export STORAGE_CLASS_NAME=longhorn # Can not be localpath because of CDI importer
export TALOS_VERSION=v1.9.5
export TALOS_IMAGE_URL=https://factory.talos.dev/image/376567988ad370138ad8b2698212367b8edcb69b5fd68c80be1f2ec7d603b4ba/v1.9.5/openstack-amd64.raw.xz
export TALOS_CODE="t${TALOS_VERSION//[^0-9]/}"
export POD_CIDR_BLOCK=10.243.0.0/16
export SVC_CIDR_BLOCK=10.95.0.0/16
export CAPK_GUEST_K8S_VERSION="v1.32.1"

clusterctl generate cluster talos-cilium --kubernetes-version ${CAPK_GUEST_K8S_VERSION} --control-plane-machine-count=1  --worker-machine-count=1   --from ./cluster-template-cip-talos-cilium-ebpf-cdi.yaml  > talos-cilium.yaml
```

Apply cluster:
```shell
kubectl apply -f talos-cilium.yaml
```

Check cluster status:
```shell
clusterctl describe cluster capi-quickstart
```

## Special tips

### WSL2 for Kubervirt

Kubevirt on WSL2 will probably fail because of a message "/var/lib/kubelet is mounted on / but it is not a shared mount", let's fix it:
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
