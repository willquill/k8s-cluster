# k8s-cluster

Deployment instructions for creating a Kubernetes cluster using Kubic

## Architecture

This will expand as I make the cluster bigger and better:

- The entire cluster will be just **one** node right now
- IP: 10.1.20.81
- Hostname: k01
- OS: openSUSE Kubic

I previously built a cluster with three nodes where two of the nodes were LXC containers and one node was a physical host with Rocky Linux 8 [here](https://github.com/willquill/kube-plex).

Note: Since I'll probably have more nodes in the future and don't want to rewrite this README, I'll use terms like "all of the nodes" when, obviously, there is only one node right now.

## Install Kubic

Nothing much to say here. I downloaded the Kubic ISO and installed it onto the destination machines.

## Create Kubernetes Cluster

If you did not install Docker and want to use CRI-O, then do:

`kubeadm init --pod-network-cidr=10.244.0.0/16`

If you installed Docker but want to use CRI-O, then do:

`kubeadm init --pod-network-cidr=10.244.0.0/16 --cri-socket=/var/run/crio/crio.sock`

^ *This is the method I used because I installed Docker on the host for other purposes.*

If you installed Docker and want to use dockershim, then do:

`kubeadm init --pod-network-cidr=10.244.0.0/16 --cri-socket=/var/run/dockershim.sock`

If it's successful, you'll see something like this:

```sh
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 10.1.20.81:6443 --token 8lxrg1.t3ly4eoijwejfcfu \
	--discovery-token-ca-cert-hash sha256:cf6fe4e63b971ff8869233c8265e432fb2319eef3c87ec64006c73e21a0962b6
```

## Setup Bastion Host

Presumably, you don't want to run your `kubectl` commands on the server all the time.

From your bastion host, install `kubectl` and then do the following with your own node IP to allow your local `kubectl` client to controller the cluster you just created:

`mkdir -p $HOME/.kube`

`scp root@10.1.20.81:/etc/kubernetes/admin.conf $HOME/.kube/config`

`chown $(id -u):$(id -g) $HOME/.kube/config`

## Setup Network Plugin

Weave is the recommended CNI, so install it with this:

`kubectl apply -f /usr/share/k8s-yaml/weave/weave.yaml`

Note: openSUSE used to recommend Flannel as the CNI, but now they recommend Weave. If you accidentally installed Flannel at any point, you can undo with the following:

`kubectl delete -f /usr/share/k8s-yaml/flannel/kube-flannel.yaml`

## Installing Packages in Kubic

A note about `transactional-update`:

- You can install a package like docker with `transactional-update pkg install docker`, but the changes will not take place until you reboot.

## Install Rancher (optional)

Personally, Rancher is not working at all for me. The output of `docker logs rancher 2>&1` shows `[FATAL] k3s exited with: exit status 1` at the end of the log. There are many GitHub issues about this with no fix, so I'm forgetting about Rancher for now. But I already wrote all these instructions below, so maybe they'll work for you.

For now, just skip to the next section, [Create Kubernetes Cluster](#create-kubernetes-cluster).

### Simple Rancher Install

Follow the [Installing Packages](#installing-packages) section to install docker (remember to reboot) and then run:

`docker run -d --name=rancher --restart=unless-stopped -p 80:80 -p 443:443 --privileged rancher/rancher`

### SSL Rancher Install

Or do it the fancy way if you want trusted certificates. In my case, I have a private CA on my OPNsense box, so I created a server certificate in OPNsense named `rancher.paw.blue` and downloaded the server cert, server key, and the full chain as `rancher.paw.blue.crt`, `rancher.paw.blue.key`, and `rancher.paw.blue.p12`, respectively.

Then on my local machine, I converted the three files as follows:

`mv rancher.paw.blue.crt ranchercert.pem` (only needs an extension change)

`openssl rsa -in rancher.paw.blue.key -text > rancherkey.pem`

`openssl pkcs12 -clcerts -in rancher.paw.blue.p12 -out ranchercacerts.pem -nokeys`

You are left with `ranchercert.pem`, `rancherkey.pem`, and `ranchercacerts.pem`

Then I copied them all to the Kubic node using `zsh` shell as follows:

`\scp rancher*.pem root@10.1.20.81:/root`

Verify on my Kubic node:

```text
k01:~ # ls -l *.pem
-rw------- 1 root root  2769 Feb  4 03:21 ranchercacerts.pem
-rw-r--r-- 1 root root  2427 Feb  4 03:21 ranchercert.pem
-rw-r--r-- 1 root root 11107 Feb  4 03:21 rancherkey.pem
```

Then run this to install Rancher via Docker:

```docker
docker run -d --name=rancher --restart=unless-stopped \
  -p 80:80 -p 443:443 \
  -v /root/ranchercert.pem:/etc/rancher/ssl/cert.pem \
  -v /root/rancherkey.pem:/etc/rancher/ssl/key.pem \
  -v /root/ranchercacerts.pem:/etc/rancher/ssl/cacerts.pem \
  --privileged \
  rancher/rancher:latest
```

## License

Distributed under the MIT License. See LICENSE for more information.

## Credit

- [Official documentation for Kubic](https://en.opensuse.org/Kubic:kubeadm)
- [This blog entry](https://blog.sdmoko.net/create-k8s-cluster-with-kubic.html) helped me
