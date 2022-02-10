# k8s-cluster

Deployment instructions for creating a Kubernetes cluster using Rocky Linux 8. See other branches for other OSes.

## Architecture

This will expand as I make the cluster bigger and better:

- The entire cluster will be just **one** node right now
- IP: 10.1.20.81
- Hostname: k01
- OS: Rocky Linux 8

I previously built a cluster with three nodes where two of the nodes were LXC containers and one node was a physical host with Rocky Linux 8 [here](https://github.com/willquill/kube-lxc).

Note: Since I'll probably have more nodes in the future and don't want to rewrite this README, I'll use terms like "all of the nodes" when, obviously, there is only one node right now.

## Install OS

Nothing much to say here. I downloaded the Rocky Linux ISO and installed it onto the destination machines. I just use a 128 GB USB flash drive with a bunch of different ISOs on it and use [Ventoy](https://www.ventoy.net/en/index.html) to choose an ISO from a menu on destination machines.

## Prepare Bastion Host

Presumably, you're going to run your `kubectl` commands from a machine not in the cluster. That's your bastion host. In my case, it's an openSUSE Tumbleweed machine using the Alacritty terminal.

### Prepare SSH Keys and Config

First, you'll want to prepare your SSH keys.

I didn't want to use the default key names of `id_rsa` and `id_rsa.pub` so I ran the following command:

`ssh-keygen -t rsa -b 4096 -N '' -f ~/.ssh/k8s_rsa`

Since every one of my kubernetes nodes will have a hostname that starts with `k0`, I'm making the SSH config apply to only the `k0*` hosts.

```sh
cat >> ~/.ssh/config<< EOF
Host k0*
    UserKnownHostsFile /dev/null
    StrictHostKeyChecking no
    IdentitiesOnly yes
    ConnectTimeout 0
    ServerAliveInterval 30
    IdentityFile ~/.ssh/k8s_rsa
EOF
```

Set up the destination for remote authentication. In my case, I'm using the `nova` user on host `k01`.

`ssh-copy-id -i ~/.ssh/k8s_rsa.pub nova@k01`

May as well do root too, just in case:

`ssh-copy-id -i ~/.ssh/k8s_rsa.pub root@k01`

### Install your terminfo (optional)

Since I'll be SSHing from Alacritty, I'll need the Alacritty terminfo on all machines.

*If you do not install the terminfo, you may sometimes see messages like this:*

```sh
E558: Terminal entry not found in terminfo
'alacritty' not known. Available builtin terminals are:
    builtin_amiga
    builtin_beos-ansi
    builtin_ansi
    builtin_pcansi
    builtin_win32
    builtin_vt320
    builtin_vt52
    builtin_xterm
    builtin_iris-ansi
    builtin_debug
    builtin_dumb
defaulting to 'ansi'
```

First, download the Alacritty repo locally.

`git clone https://github.com/alacritty/alacritty.git`

`cd alacritty`

Now copy `extra/alacritty.info` to remote machines.

`scp extra/alacritty.info nova@k01:/home/nova`

And install the term info on remote machines.

`ssh nova@k01 'sudo tic -xe alacritty,alacritty-direct alacritty.info'`

The next few commands will be run on your destination hosts, where the commands differ based on linux distribution.

### Prepare hosts file

My two hosts:

Host Purpose  |     IP     |       FQDN      | hostname
--------------|------------|-----------------|---------
Bastion       | 10.1.20.40 | lizard.paw.blue | lizard
K8s Node 01   | 10.1.20.81 | k01.paw.blue    | k01.paw.blue

So I want to update the /etc/hosts file on my bastion machine like this:

```sh
sudo tee -a /etc/hosts > /dev/null <<EOT
10.1.20.40 lizard.paw.blue lizard
10.1.20.81 k01.paw.blue k01

# K8s Endpoints
10.1.20.81 k8ep.paw.blue k8ep
EOT
```

Notice that I use `tee` to append because I need root privileges.

### Install Python 3.9

I used [this](https://computingforgeeks.com/install-kubernetes-cluster-on-rocky-linux-with-kubeadm-crio/) guide.

I won't reiterate all the steps from that guide, but here are some of them, as there were some occasional deviations. For some of the steps, I included instructions for both openSUSE and Rocky Linux bastion hosts.

openSUSE only: `sudo zypper in python39`

Rocky Linux only: `sudo dnf -y install python39`

Both OSes:

`sudo pip3 install setuptools-rust wheel`

`sudo pip3 install --upgrade pip`

### Install Ansible

`sudo python3.9 -m pip install ansible`

`ansible --version`

You should see something like this in openSUSE:

```sh
ansible [core 2.12.1]
  config file = None
  configured module search path = ['/home/will/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/local/lib/python3.9/site-packages/ansible
  ansible collection location = /home/will/.ansible/collections:/usr/share/ansible/collections
  executable location = /usr/local/bin/ansible
  python version = 3.9.9 (main, Nov 17 2021, 09:50:59) [GCC]
  jinja version = 3.0.3
  libyaml = True
```

You should see something like this in Rocky Linux:

```sh
ansible [core 2.12.1]
  config file = None
  configured module search path = ['/home/nova/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/local/lib/python3.9/site-packages/ansible
  ansible collection location = /home/nova/.ansible/collections:/usr/share/ansible/collections
  executable location = /usr/local/bin/ansible
  python version = 3.9.6 (default, Nov  9 2021, 13:31:27) [GCC 8.5.0 20210514 (Red Hat 8.5.0-3)]
  jinja version = 3.0.3
  libyaml = True
```

## Install Kubernetes

The `k8s-prep.yml` Ansible playbook will do a lot, including installing kubelet, kubeadm, and kubectl.

`ansible-playbook -i hosts -l k8snodes k8s-prep.yml --ask-become-pass`

## Mount NFS

`ansible-playbook -i hosts -l k8sworkers mount-nfs.yml --ask-become-pass`

## Create the cluster

From my bastion host, I'm running a remote SSH command on my node. I couldn't figure out how to do this as my regular user, so I'm doing it as root.

Remember to do this first if you didn't do it earlier:

`ssh-copy-id -i ~/.ssh/k8s_rsa.pub root@k01`

Do a dry run and then look at your `dryrun.txt` file.

`ssh root@k01 'kubeadm init --dry-run --cri-socket=/var/run/crio/crio.sock --control-plane-endpoint=k8ep.paw.blue --pod-network-cidr=10.244.0.0/16 --kubernetes-version=1.23.3' > dryrun.txt`

Do it for real.

`ssh root@k01 'kubeadm init --cri-socket=/var/run/crio/crio.sock --control-plane-endpoint=k8ep.paw.blue --pod-network-cidr=10.244.0.0/16 --kubernetes-version=1.23.3'`

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

kubeadm join 10.1.20.81:6443 --token <redacted> \
  --discovery-token-ca-cert-hash sha256:<redacted>
```

## Setup Bastion Host

Presumably, you don't want to run your `kubectl` commands on the server all the time.

From your bastion host, install `kubectl` and then do the following with your own node IP to allow your local `kubectl` client to controller the cluster you just created:

`mkdir -p $HOME/.kube`

`scp root@k01:/etc/kubernetes/admin.conf $HOME/.kube/config`

`chown $(id -u):$(id -g) $HOME/.kube/config`

## Verify Installation

Make sure everything started.

`kubectl get all -A`

## License

Distributed under the MIT License. See LICENSE for more information.

## Credit

- [Official documentation for Kubic](https://en.opensuse.org/Kubic:kubeadm)
- [This blog entry by Josphat Mutai](https://blog.sdmoko.net/create-k8s-cluster-with-kubic.html)
