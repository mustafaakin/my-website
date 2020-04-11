---
title: Installing a multi-node Kubernetes cluster with Ubuntu Multipass
date: 2020-04-11
--- 

There are tons of tutorials out there for installing Kubernetes. I am just documenting my homelab installation. I like to install Kubernetes without external tools, but relying on _kubeadm_ and using actual Linux installations in VMs, it gives me more control and helps me to understand how it actually works, compared to alternatives like minikube and kind. 

First, I will create 3 VMs with [Ubuntu Multipass](https://multipass.run). The reason I have chosen it is that it makes it really easy to create virtual machines on demand. It's not rocket science, it works with QEMU under the hood, but it makes the management of it easy by configuring the network bridging, disk images and non-ending list of QEMU variables. It has a potential to become the Docker of creating virtual machines, but right now only Ubuntu images are available. As an alternative, one can choose [Weave Ingite](https://github.com/weaveworks/ignite), which is based on Firecracker MicroVM, but boots compatible kernels with OCI images under the hood. It's an interesting project to checkout, I will be working on it soon. 

The following scripts creates the VMs. If the disk images are available, it takes approximately 30 seconds per VM to be available.

```shell
$ multipass launch -c 4 -d 50G -m 8G -n node1
$ multipass launch -c 4 -d 50G -m 8G -n node2
$ multipass launch -c 4 -d 50G -m 8G -n node3

# Verify all worked correctly.
$ multipass info --all
Name:           node1
State:          Running
IPv4:           10.161.139.229
Release:        Ubuntu 18.04.4 LTS
Image hash:     fe3030939822 (Ubuntu 18.04 LTS)
Load:           0.08 0.02 0.01
Disk usage:     1006.5M out of 48.3G
Memory usage:   99.5M out of 7.8G

Name:           node2
State:          Running
IPv4:           10.161.139.12
Release:        Ubuntu 18.04.4 LTS
Image hash:     fe3030939822 (Ubuntu 18.04 LTS)
Load:           0.04 0.03 0.01
Disk usage:     1006.1M out of 48.3G
Memory usage:   102.0M out of 7.8G

Name:           node3
State:          Running
IPv4:           10.161.139.185
Release:        Ubuntu 18.04.4 LTS
Image hash:     fe3030939822 (Ubuntu 18.04 LTS)
Load:           0.09 0.04 0.01
Disk usage:     1007.7M out of 48.3G
Memory usage:   103.2M out of 7.8G
```

At the time of wrtiting this blog, I'm using Ubuntu 20.04 Beta, but multipass defaults to 18.04 for stability reasons. Following the [official documentation](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/) of installing Kubernetes, I have selected Docker for container runtime, the installation can be found [here](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#docker).

After ensuring docker, kubeadm, kubectl is installed in each node, we can go ahead with actually creating the cluster. One caveat though; in order to create proper multi-node networking, we need to select a network add-on before creating the cluster. It's written in the docs, but can be easily overlooked. I have chosen Calico as it's the easiest one. Also my home network is on `192.168.3.1/24` and to ensure it does not conflict with my home network and the network multipass builds, I will create a network in the `172.16.0.0/12`. I probably won't need `2^20` IP, but anways. 

```sh
# Kick start the cluster creation in node1
$ kubeadm init --pod-network-cidr=172.16.0.0/12
.... 
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 10.161.139.229:6443 --token h6r5uv.3vwcmbxtlkzwpwow \
  --discovery-token-ca-cert-hash sha256:bc23f307454945755ffe5ed542e2732d79baedcbefc95a92f856d13456333265 
```

After setting up kubectl, you can join the node2 and node3 to the cluster. run the `kubeadm join` command as specified in the output of the `kubeadm init` in the first node. The nodes will pull the related Docker images and kubelets will join the cluster really quick.

```sh
$ kubectl get node
NAME    STATUS     ROLES    AGE     VERSION
node1   NotReady   master   5m42s   v1.18.1
node2   NotReady   <none>   5m15s   v1.18.1
node3   NotReady   <none>   5m1s    v1.18.1
```

You see the nodes are there, but they are not ready. Because the network add-on is not setup, and they cannot communicate properly. Download the manifest of Calico network and remember to change the CIDR and apply it.

```sh
$ wget https://docs.projectcalico.org/v3.11/manifests/calico.yaml
$ vi calico.yaml # Chagne 192.168.0.0/16 to 172.16.0.0/12
$ kubectl apply -f calico.yaml
# Wait a little
$ kubectl get node
NAME    STATUS   ROLES    AGE     VERSION
node1   Ready    master   8m50s   v1.18.1
node2   Ready    <none>   8m23s   v1.18.1
node3   Ready    <none>   8m9s    v1.18.1
```

As you see the installation was really quick! However, I would like the opportunity to manage my cluster in the host system, not requiring to use multipass to shell into the instance. To do that, we need to copy the kubeconfig file. 

```sh
$ mkdir ~/.kube
$ multipass copy-files node1:.kube/config ~/.kube/
$ kubectl get node
```

Yeah, it's working! It's a pleasant experience. Most people think installing Kubernetes is hard, but it is actually not. For simpler use cases, Minikube works great. Docker for Desktop is also good enough experience for macOS and Windows. For my own experiments, I need raw Kubernetes installation, without any hacks and full access to the kubelet hosts.  Kubeadm works very fast and requires minimal configuration. Of course, running a production ready Kubernetes cluster is a complicated task and you are probably best using managed Kubernetes services. I did install on my machine because I need it for Ph.D experiments, and I have many cores and memory to spare and it is much cheaper. 

Finally, I want to give a shout out to Ubuntu for developing [multipass](https://multipass.run/). It also works on macOS and Windows. It's faster than similar tools. You can close the VMs with `multipass stop --all` and start them again with `multipass start --all`. The nodes will get the same IP address (need to confirm it's always the case) and you can resume working since kubelet and the etcd will be started. 