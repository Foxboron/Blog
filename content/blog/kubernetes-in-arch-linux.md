---
title: "Kubernetes in Arch Linux"
date: "2020-12-23T20:00:00+02:00"
tags:
  - english
  - archlinux
---

Arch Linux got [kubernetes](https://www.archlinux.org/packages/community/x86_64/kubernetes/) packaged into the `[community]` repository the past
week with the hard work of David Runge. I contribute to testing the packages so
I thought it would be interesting to write up quickly the testing that was done.
Originally I did the testing with `docker` but with the `dockershim` [deprecation](https://kubernetes.io/blog/2020/12/02/dockershim-faq/)
I rewrote the blog to utilize `containerd` instead.

David has reworked the kubernetes [archwiki](https://wiki.archlinux.org/index.php/Kubernetes) article as well. It currently doesn't
cover all use cases and contributions welcome. I will try cover the `containerd`
parts of this page to the wiki.

Our goal in this post is to simply setup a one node "cluster" and test some
basic functionality. `kubeadm` is going to be used for bootstrapping the cluster
as it is an easy way to get started.

For the testing portion of kubernetes I utilized [lxd](https://linuxcontainers.org/) as it's a neat interface
for managing virtual machines. They also provide Arch Linux images which makes
things fairly easy to setup.

{{< highlight text >}}
# lxc launch images:archlinux/current arch-vm-kubernetes-test --vm
# lxc exec arch-vm-kubernetes-test bash
{{< /highlight >}}

This sets up an Arch Linux virtual machine we will setup kubernetes with. Next
up we need to install the packges we will be using to setup kubernetes.

{{< highlight text >}}
# pacman -S kubernetes-control-plane 
# pacman -S containerd kubeadm kubectl
{{< /highlight >}}

We have split up the kubernetes package in different components and then
construced 3 package groups for users. `kubernetes-control-plane` for the
control planes/master nodes. `kubernetes-node` for worker nodes, and then
`kubernetes-tools` for the tools like `kubectl`. We are also fetching
`containerd` for the container runtime.

Setup of the different container runtimes is explained at the [container runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)
webpage. However, there are a few distro specifics which could be nice to
document on the archwiki. Next part of the setup is to ensure the modules and
kernel parameters are correct.

{{< highlight text >}}
# cat <<EOF | tee /etc/modules-load.d/containerd.conf
br_netfilter
EOF
# modprobe br_netfilter
# cat <<EOF | tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
# sysctl --system
# systemctl start containerd
{{< /highlight >}}

I have no clue why they `br_netfilter` are needed, but I guess `docker` solves
this itself where as `containerd` does not. It should be noted we don't setup
the overlay module as the `containerd` service inserts it on its own. Rest of it
is fairly straight forward and verbatim from the kubernetes documentation.

{{< highlight text >}}
# kubeadm init --pod-network-cidr=192.168.0.0/16 \
	--upload-certs --ignore-preflight-errors=NumCPU,Mem
[init] Using Kubernetes version: v1.19.4
[preflight] Running pre-flight checks
[[...SNIP...]]
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!
[[...SNIP...]]
{{< /highlight >}}

`kubeadm` sets up the kubernetes cluster. The pod network CIRD defines the IP
range the pods (read containers) are going to be using inside the cluster.
Because we are using the default `lxd` containers we need to ignore a few limits
kubernetes imposes on us. It expects more then 1 CPU code, and more then 1GB
RAM. However this doesn't matter for the fairly small workload we have for the
testing.

In a real cluster you would hopefully have larger nodes!

Once the cluster has initialized we can copy the `admin.conf` and query for the
cluster nodes.

{{< highlight text >}}
# mkdir ~/.kube
# cp -i /etc/kubernetes/admin.conf ~/.kube/config
# kubectl get nodes
NAME        STATUS      ROLES    AGE   VERSION
archlinux   NotReady    master   18m   v1.20.0
{{< /highlight >}}

Currently it says `NotReady` which is because there is no network configured on
the cluster. We will untaint the master node, which allows us to deploy pods on
the master, and then setup some network.


{{< highlight text >}}
# kubectl taint nodes --all node-role.kubernetes.io/master-
node/archlinux untainted
{{< /highlight >}}

We will be using `kube-router` for the networking portion as it is what I have
been using personally, and it has been fairly straight forward to setup.

It can be fetched from their github repository which has the RBAC support
`kubeadm` setup. We can just apply this deployment fairly easily.
https://raw.githubusercontent.com/cloudnativelabs/kube-router/v1.1.0/daemonset/kubeadm-kuberouter.yaml

{{< highlight text >}}
# kubectl apply -f kubeadm-kuberouter.yaml
# kubectl get pods -A
NAMESPACE     NAME                                READY   STATUS    RESTARTS
kube-system   coredns-f9fd979d6-2wp2x             1/1     Running   0
kube-system   coredns-f9fd979d6-522nz             1/1     Running   0
kube-system   etcd-archlinux                      1/1     Running   0
kube-system   kube-apiserver-archlinux            1/1     Running   0
kube-system   kube-controller-manager-archlinux   1/1     Running   0
kube-system   kube-proxy-sjxvw                    1/1     Running   0
kube-system   kube-router-8fq2m                   1/1     Running   0
kube-system   kube-scheduler-archlinux            1/1     Running   0
# kubectl get nodes
NAME        STATUS  ROLES    AGE   VERSION
archlinux   Ready   master   20m   v1.20.0
{{< /highlight >}}

Now our cluster is ready for a workload! Personally I have sometimes queried the
`docker` for the running containerd, but we are currently using `containerd`
which provides `ctr` for interacting with the containers. It organizes all pods
related to kubernetes inside a `k8s.io` namespace, where you can list the pods.

{{< highlight text >}}
# ctr -n k8s.io c list
CONTAINER  IMAGE                                         RUNTIME             
0cf8a9c17  docker.io/cloudnativelabs/kube-router:latest  io.containerd.runc.v2
2bda5427a  k8s.gcr.io/pause:3.2                          io.containerd.runc.v2
4dbdaea0d  k8s.gcr.io/pause:3.2                          io.containerd.runc.v2
5430728fc  k8s.gcr.io/pause:3.2                          io.containerd.runc.v2
546633029  k8s.gcr.io/pause:3.2                          io.containerd.runc.v2
5593c7f09  k8s.gcr.io/kube-proxy:v1.20.0                 io.containerd.runc.v2
7f4dd2ff4  k8s.gcr.io/pause:3.2                          io.containerd.runc.v2
96dd370ec  k8s.gcr.io/pause:3.2                          io.containerd.runc.v2
aab09f1e1  docker.io/cloudnativelabs/kube-router:latest  io.containerd.runc.v2
aba203048  k8s.gcr.io/pause:3.2                          io.containerd.runc.v2
b1bd5082a  sha256:0369cf4303ffdb467dc219990960a9baa8...  io.containerd.runc.v2
dc02359d9  k8s.gcr.io/coredns:1.7.0                      io.containerd.runc.v2
e42de4f65  k8s.gcr.io/kube-controller-manager:v1.20.0    io.containerd.runc.v2
fc943a990  k8s.gcr.io/kube-apiserver:v1.20.0             io.containerd.runc.v2
fe49fbf3f  k8s.gcr.io/kube-scheduler:v1.20.0             io.containerd.runc.v2
{{< /highlight >}}

Now we have the basics cluster working. We will add an deployment on this
cluster and expose it through a load balancer. It's

The steps are taken from the "create a deployment" page in the [kubernetes](https://kubernetes.io/docs/tutorials/hello-minikube/#create-a-deployment)
documentation.

{{< highlight text >}}
# kubectl create deployment hello-node --image=k8s.gcr.io/echoserver:1.10
# kubectl expose deployment hello-node --type=LoadBalancer --port=8080
service/hello-node exposed
# kubectl get services
NAME         TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
hello-node   LoadBalancer   10.96.177.87   <pending>     8080:31569/TCP   5s
kubernetes   ClusterIP      10.96.0.1      <none>        443/TCP          22m
# curl 10.96.177.87:8080

Hostname: hello-node-59bffcc9fd-hs7mk

Pod Information:
	-no pod information available-

Server values:
	server_version=nginx: 1.13.3 - lua: 10008

Request Information:
	client_address=192.168.0.1
	method=GET
	real path=/
	query=
	request_version=1.1
	request_scheme=http
	request_uri=http://10.96.177.87:8080/

Request Headers:
	accept=*/*
	host=10.96.177.87:8080
	user-agent=curl/7.73.0

Request Body:
	-no body in request-
{{< /highlight >}}

This shows an fairly simple setup with one kubernetes node running a service and
a pod. It can be expanded upon, but for the sake of keeping this short this
works as an introduction I reckon.

Hopefully this was as a simple introduction to kubernetes on Arch Linux!
