# CAPI easy via Kubevirt

This repo contains cookbooks for starting Kubernetes via Cluster API using Kubevirt.

Supported platforms:
  - [x] Talos
  - [ ] Ubuntu (tested ok, need to commit)
  - [ ] Flatcar (WIP)

TODOs:
  - support non Longhorn version (WIP)

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

## virtctl

virtctl is a command line tool that can interacts with Kubevirt virtual machines:
```shell
export VERSION=$(curl https://storage.googleapis.com/kubevirt-prow/release/kubevirt/kubevirt/stable.txt)\ncurl -SLO  https://github.com/kubevirt/kubevirt/releases/download/${VERSION}/virtctl-${VERSION}-linux-amd64
sudo chmod +x virtctl-v1.5.0-linux-amd64
sudo mv virtctl-v1.5.0-linux-amd64 /usr/local/bin/virtctl
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

# Talos + Cilium

## Init CAPI

Before creating the cluster, we need to use clusterctl to init the different providers we'll use:
```shell
clusterctl init --ipam in-cluster --control-plane talos --bootstrap talos  --infrastructure kubevirt --addon helm
```

## Prepare cluster

Current list of VMClusterPreference does not contain any Talos definition, let's add it:

```shell
kubectl apply -f talosVMCP.yaml
```

## Choose your instance type

Kubevirt installs instance types you can then use to avoid repeating yourself when defining resources for a VM. "o1.xlarge" is recommended for easy test, but you can get a list of existing types via `kubectl get VirtualMachineClusterInstancetype` for more instance types and then describe them to check their settings. Please not some of those types requires cpu-manager to be configured on your hosts kubelet (for example any of the "cx" serie)

## Prepare image

Go to https://factory.talos.dev/ and prepare your image by choosing:
  - Hardware type: Cloud Server
  - Talos Linux Version: 1.9.5 or higher
  - Cloud: Openstack
  - Machine Architecture: amd64 or arm64 depending on your hardware
  - Extensions: add any you want
  - Customization: leave as is

You'll get some links and one of them is the link to a raw.xyz image (the one ending with openstack-amd64.raw.xz). Copy this link and remind it for later use.

## Create cluster

Creating a cluster with Cluster API is very easy. It uses a template mechanism with a few variables replacements.

First let's export the variables that will be injected in the template. Every var can be tuned (please notice only the image variable is not preset and you MUST provide one):
```shell
export INSTANCE_PREFERENCE=talos
export INSTANCE_TYPE=o1.xlarge # see kubectl get VirtualMachineClusterInstancetype for more instance_types
export ROOT_VOLUME_SIZE=10G
export CONTROL_PLANE_SVC_TYPE=ClusterIP # LoadBalancer, ClusterIP or NodePort
export STORAGE_CLASS_NAME=longhorn # Can not be localpath because of CDI importer
export TALOS_VERSION=v1.9.5
export TALOS_IMAGE_URL= # add here your image https://factory.talos.dev/image/xxx/v1.9.5/openstack-amd64.raw.xz
export TALOS_CODE="t${TALOS_VERSION//[^0-9]/}"
export POD_CIDR_BLOCK=10.243.0.0/16
export SVC_CIDR_BLOCK=10.95.0.0/16
export CAPK_GUEST_K8S_VERSION="v1.32.1"
```

Now let's generate the cluster manifests:
```shell
clusterctl generate cluster talos-cilium --kubernetes-version ${CAPK_GUEST_K8S_VERSION} --control-plane-machine-count=1  --worker-machine-count=1   --from ./cluster-template-talos-cilium-ebpf-cdi.yaml  > talos-cilium.yaml
```

You can check the generated file `talos-cilium.yaml` to understand what objects will be created.

Next you can apply cluster:
```shell
kubectl apply -f talos-cilium.yaml
```

## Check cluster

clusterctl can give you overall status of the cluste:
```shell
clusterctl describe cluster capi-quickstart
```

Booting the whole cluster with one control plane and one node shoot take from 3 to 5 minutes depending one your hardware and Internet connexion.

You can also check the status of a few CRDs:
  - VirtualMachine (created by Kubevirt) - kubectl get vms, kubectl describe vms
  - Machine (created by CAPI)  - kubectl get machines, kubectl describe machines

You can also check for logs in 'guest-console-log' of the VMs Pods. 

You even can connect to graphic interface of Talos: get the name of the vm with `kubectl get vms` then `virtctl vnc`

# Special tips

## WSL2 for Kubervirt

Kubevirt on WSL2 will probably fail because of a message "/var/lib/kubelet is mounted on / but it is not a shared mount", let's fix it:
```
sudo mount --make-shared /
```

## k3s longhorn

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

# Credits

This work is mostly inspired by the excellent work of the kubernetes-sigs/cluster-api-provider-kubevirt team, thanks to them for their incredible work.
