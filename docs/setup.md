## Requirements

* One or more machines running a deb/rpm-compatible OS (e.g Ubuntu / CentOS).
* 2 GB or more of RAM per machine. Any less leaves little room for your apps.
* 2 CPUs or more on the master.
* Full network connectivity among all machines in the cluster.

## Install prerequisites

_The following setup instructions assume Ubuntu OS with root access_

### Install docker

```bash
apt update
apt upgrade
apt install docker.io
```

### Install kubelet kubeadm kubectl
_[source](https://kubernetes.io/docs/setup/independent/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl)_

```bash
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
apt install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl
```

?> You don’t need to install kubectl on worker nodes

### Config autocomplete

```bash
kubeadm completion bash > /etc/bash_completion.d/kubeadm
kubectl completion bash > /etc/bash_completion.d/kubectl
source .profile
```

## Master

### Initialize cluster
_[source](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/#initializing-your-master)_

On your master node run this, which conveniently stores the output in a `README` file so you can access it when you want to join worker nodes. 

```bash
kubeadm init --pod-network-cidr=10.244.0.0/16 | tee README
```

?> notice the `pod-network-cidr` option, which is needed for the [CANAL Network Provider](#canal-network-provider)

### Setup admin config

Copy the admin configuration so that you can run `kubectl` commands to your cluster.

```bash
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```

?> copy that file over `scp` to your local machine to use `kubectl` remotely

### CANAL Network Provider

Install a `Network Provider` _[source](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/#pod-network)_

_Make sure to use the latest available version_

```bash
kubectl apply -f https://docs.projectcalico.org/v3.3/getting-started/kubernetes/installation/hosted/canal/rbac.yaml
kubectl apply -f https://docs.projectcalico.org/v3.3/getting-started/kubernetes/installation/hosted/canal/canal.yaml
```

### Control Plane Isolation

If you only have one machine (i.e. single machine cluster) you need to remove the taint from the master node, so that kubernetes can schedule pods to be created on the master node

Remove taint _(this allows for pods to be created in the master node)_
   
```bash
kubectl taint nodes <node> node-role.kubernetes.io/master-
```

Add taint _(this prevents for pods to be created in the master node)_

```bash
kubectl taint nodes <node> node-role.kubernetes.io/master=:NoSchedule
```

?> Master node is `tainted` by default

## Nodes

### Join cluster
On each of your other machines (i.e. `workers`) run this as root

```bash
kubeadm join <master-ip>:<master-port> --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

?> See the `README` file that was created while initializing the cluster for more details

### Set worker role

From your local machine or your master node run this for every worker `node`.
This will help distinguishing `master` and `worker` nodes.

```bash
kubectl label node <node-name> node-role.kubernetes.io/worker=
```
