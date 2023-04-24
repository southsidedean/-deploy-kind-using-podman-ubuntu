# Deploying a Kubernetes In Docker (KIND) Cluster Using Podman on Ubuntu Linux

***Tom Dean - 3/9/23***

###### tags: `Ubuntu`, `Docker`, `Podman`, `KIND`, `Linux`, `Debian`, `Intel`, `apt-get`, `Kubernetes`, `K8s`

## Introduction

Sometimes you just need a lightweight Kubernetes cluster.  You might want to brush up on your Kubernetes skills in preparation for a CNCF exam, prototype and test some code, or one of a million other possible scenarios.  Standing up a Kubernetes cluster, even with automation and the Cloud, can be slow, cumbersome and overkill for many of your K8s needs.

One great way to practice Kubernetes, without the need to create a full-footprint, multi-node cluster, is using a Kubernetes-IN-Docker, or KIND cluster.  A KIND cluster is a minimal Kubernetes cluster, running as a Docker, or Podman, container.  KIND even has Rootless support, using Podman, allowing you to run an entire KIND cluster without requiring elevated permissions.

In this tutorial, we're going to use Rootless Podman to deploy a KIND cluster on Ubuntu Linux 20.04.

***Let's see how we set it up!***


## References

[KIND: Kubernetes in Docker](https://kind.sigs.k8s.io)

[KIND: Quick Start](https://kind.sigs.k8s.io/docs/user/quick-start/)

[Rootless KIND](https://kind.sigs.k8s.io/docs/user/rootless/)


## Checking For and Removing Docker *(If Necessary)*

If you have Docker installed, you will need to remove it.  Let's check.

Check and see if Docker is installed:
```bash
docker version
```

If Docker is installed, remove Docker with:
```bash
sudo apt-get update
sudo apt-get remove docker* -y
```

Check for Docker again:
```bash
docker version
```

You should see something like the following:
```bash
-bash: /usr/bin/docker: No such file or directory
```

***Docker should now be uninstalled.  We're ready to proceed with installing Podman.***


## Installing Podman

You will need ***Podman*** installed.  Let's check and see if it is.

Check for Podman:
```bash
podman version
```

If Podman is installed, we should see output similar to the following:
```bash
Version:      3.4.2
API Version:  3.4.2
Go Version:   go1.15.2
Built:        Thu Jan  1 00:00:00 1970
OS/Arch:      linux/amd64
```

If you don't already have Podman installed, you can install it using `apt-get`.

First, we'll need to add the Podman repositories:
```bash
sudo apt-get install -y curl wget gnupg2
source /etc/os-release ; sudo sh -c "echo 'deb http://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/xUbuntu_${VERSION_ID}/ /' > /etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list"
```

Add the GPG key for the Podman repositories:
```bash
source /etc/os-release ; wget -nv https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable/xUbuntu_${VERSION_ID}/Release.key -O- | sudo apt-key add -
```

Now, install Podman using `apt-get`:
```bash
sudo apt-get update
sudo apt-get install -y podman
```

Checking our work:
```bash
podman version
```

We should see output similar to the following:
```bash
Version:      3.4.2
API Version:  3.4.2
Go Version:   go1.15.2
Built:        Thu Jan  1 00:00:00 1970
OS/Arch:      linux/amd64
```

***Now that Podman is installed, we can give it a try!***

### Quick Test: Podman

Let's make sure Podman is working:
```bash
podman run hello-world
```

We should see something like the following:
```bash title=Output
Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```

## Install `go`

In order to install and use KIND, you will need to have `go` installed.  If you don't already have `go` installed, the most direct way to install it is using `snap`:
```bash
sudo snap install go --classic
```

### Quick Test: `go`

Let's verify our `go` installation:
```bash
go version
```

We should see output like the following:
```bash
go version go1.19.5 linux/amd64
```

Your version of `go` may be different, just make sure it's `1.17+`.

## Install `kubectl`

We're going to need a way to talk to the API of our KIND cluster, and `kubectl` is it.  Let's check and see if it's installed:
```bash
kubectl version --output=yaml
```

If you don't already have `kubectl` installed, you can install it using `apt-get`:
```bash
sudo apt-get install kubectl -y
```

Checking our work:
```bash
kubectl version --output=yaml
```

We should see something similar to the following:
```bash
clientVersion:
  buildDate: "2022-09-21T13:19:24Z"
  compiler: gc
  gitCommit: b39bf148cd654599a52e867485c02c4f9d28b312
  gitTreeState: clean
  gitVersion: v1.24.6
  goVersion: go1.18.6
  major: "1"
  minor: "24"
  platform: linux/amd64
kustomizeVersion: v4.5.4

The connection to the server localhost:8080 was refused - did you specify the right host or port?
```

*Remember, versions and architectures can vary based on when you install and what you install it on!*

Our `kubectl` command is installed and ready to go.  Don't worry about the connection error, we don't have an API to talk to yet.

***Let's do some final configuration, and then we can install KIND!***

## Configure Your Shell

Before we start working with KIND, we're going to want to set a couple of things in our shell environment to make working with KIND easier.  If you're using BASH, use the `.bashrc` file.

Add some environment variables to your `.bashrc` file:
```bash
cat << EOF >> ~/.bashrc
alias docker=podman
PATH=$PATH:~/go/bin
KIND_EXPERIMENTAL_PROVIDER=podman
EOF
```

Next, we'll source the `~/.bashrc` file to pick up the changes.

For BASH:
```bash
source .bashrc
```

Checking our work:
```bash
docker version
```

We should see output like the following:
```bash
Version:      3.4.2
API Version:  3.4.2
Go Version:   go1.15.2
Built:        Thu Jan  1 00:00:00 1970
OS/Arch:      linux/amd64
```

***Now we're ready to roll up our sleeves and get started with KIND!***

## Installing KIND

Now that our prerequisites are satisfied and we have working Podman and `go` installations, let's install KIND.  KIND is installed using `go`, and will be installed in the `~/go/bin` directory.  We already added that directory to our path, so we can just type `kind` when we want to use KIND.

Install KIND:
```bash
go install sigs.k8s.io/kind@v0.17.0
```

We should see output like the following:
```bash
go: downloading sigs.k8s.io/kind v0.17.0
go: downloading github.com/spf13/pflag v1.0.5
go: downloading github.com/spf13/cobra v1.4.0
go: downloading github.com/pkg/errors v0.9.1
go: downloading github.com/alessio/shellescape v1.4.1
go: downloading github.com/mattn/go-isatty v0.0.14
go: downloading golang.org/x/sys v0.0.0-20210630005230-0f9fa26af87c
go: downloading github.com/pelletier/go-toml v1.9.4
go: downloading github.com/BurntSushi/toml v1.0.0
go: downloading github.com/evanphx/json-patch/v5 v5.6.0
go: downloading gopkg.in/yaml.v3 v3.0.1
go: downloading sigs.k8s.io/yaml v1.3.0
go: downloading gopkg.in/yaml.v2 v2.4.0
go: downloading github.com/google/safetext v0.0.0-20220905092116-b49f7bc46da2
```

If we check the `~/go/bin` directory, we should see `kind`:
```bash
ls -al ~/go/bin
```

Our output should be similar to this:
```bash
total 9196
drwxrwxr-x 2 ubuntu ubuntu    4096 Feb  2 22:35 .
drwxrwxr-x 4 ubuntu ubuntu    4096 Feb  2 22:35 ..
-rwxrwxr-x 1 ubuntu ubuntu 9408040 Feb  2 22:35 kind
```

Checking KIND:
```bash
kind version
```

We should see something similar to the following:
```bash
kind v0.17.0 go1.19.5 linux/amd64
```

*Again, versions and architectures can vary based on when you install and what you install it on!*

***Now that we have a working KIND installation, let's use it to deploy a KIND cluster!***

## Deploying a KIND Cluster

Before we dive right in and deploy our KIND cluster, let's take a quick look at how we use the `kind` command.  We have already added `~/go/bin` to our `PATH`, so we don't need to specify the full path to the `kind` executable.

We can use the `--help` switch to get information on the `kind` command:
```bash
kind --help
```

Help Output:
```bash
kind creates and manages local Kubernetes clusters using Docker container 'nodes'

Usage:
  kind [command]

Available Commands:
  build       Build one of [node-image]
  completion  Output shell completion code for the specified shell (bash, zsh or fish)
  create      Creates one of [cluster]
  delete      Deletes one of [cluster]
  export      Exports one of [kubeconfig, logs]
  get         Gets one of [clusters, nodes, kubeconfig]
  help        Help about any command
  load        Loads images into nodes
  version     Prints the kind CLI version

Flags:
  -h, --help              help for kind
      --loglevel string   DEPRECATED: see -v instead
  -q, --quiet             silence all stderr output
  -v, --verbosity int32   info log verbosity, higher value produces more output
      --version           version for kind

Use "kind [command] --help" for more information about a command.
```

The three verbs we're going to use in this tutorial are `get`, `create` and `delete`.

Let's check for existing KIND clusters before we proceed:
```bash
kind get clusters
```

We shouldn't see any clusters:
```bash
enabling experimental podman provider
No kind clusters found.
```

Let's create a cluster:
```bash
kind create cluster
```

We should see output similar to this:
```bash
enabling experimental podman provider
Cgroup controller detection is not implemented for Podman. If you see cgroup-related errors, you might need to set systemd property "Delegate=yes", see https://kind.sigs.k8s.io/docs/user/rootless/
Creating cluster "kind" ...
 ✓ Ensuring node image (kindest/node:v1.25.3) 🖼 
 ✓ Preparing nodes 📦  
 ✓ Writing configuration 📜 
 ✓ Starting control-plane 🕹️ 
 ✓ Installing CNI 🔌 
 ✓ Installing StorageClass 💾 
Set kubectl context to "kind-kind"
You can now use your cluster with:

kubectl cluster-info --context kind-kind

Have a question, bug, or feature request? Let us know! https://kind.sigs.k8s.io/#community 🙂
```

***If this fails with an error telling us we need to configure Linux to use `cgroups v2`.  If this is the case, do the steps in the next section, "Configure Host To Use `cgroup v2`".  If not, skip the next section and proceed with "More KIND Cluster Operations".***

### Configure Host To Use `cgroup v2`

In order to get KIND working, we need to be using `cgroups v2` on our host.

First, become `root`:
```bash
sudo su
```

Add a line to `/etc/default/grub` and update `grub`:
```bash
cat << EOF >> /etc/default/grub
GRUB_CMDLINE_LINUX="systemd.unified_cgroup_hierarchy=1"
EOF
update-grub
```

Enable delegation of the CPU controller:
```bash
mkdir /etc/systemd/system/user@.service.d
cat << EOF >> /etc/systemd/system/user@.service.d/delegate.conf
[Service]
Delegate=yes
EOF
systemctl daemon-reload
```

Configure some iptables modules to load at boot:
```bash
cat << EOF >> /etc/modules-load.d/iptables.conf
ip6_tables
ip6table_nat
ip_tables
iptable_nat
EOF
```

Reboot the host:
```bash
systemctl reboot
```

***Let the system reboot, then log back in after a minute.***

Let's try to create a KIND cluster again:
```bash
kind create cluster
```

We should see output similar to this:
```bash
enabling experimental podman provider
Cgroup controller detection is not implemented for Podman. If you see cgroup-related errors, you might need to set systemd property "Delegate=yes", see https://kind.sigs.k8s.io/docs/user/rootless/
Creating cluster "kind" ...
 ✓ Ensuring node image (kindest/node:v1.25.3) 🖼 
 ✓ Preparing nodes 📦  
 ✓ Writing configuration 📜 
 ✓ Starting control-plane 🕹️ 
 ✓ Installing CNI 🔌 
 ✓ Installing StorageClass 💾 
Set kubectl context to "kind-kind"
You can now use your cluster with:

kubectl cluster-info --context kind-kind

Have a question, bug, or feature request? Let us know! https://kind.sigs.k8s.io/#community 🙂
```

### More KIND Cluster Operations

Checking for clusters:
```bash
kind get clusters
```

Checking for our cluster:
```bash
enabling experimental podman provider
kind
```

We can see we have a KIND cluster named `kind`.  This cluster is actually a container, running under Podman:
```bash
podman ps -a
```

We can see our KIND container (`docker.io/kindest/node@...`):
```bash
CONTAINER ID  IMAGE                                                                                           COMMAND     CREATED        STATUS                  PORTS                      NAMES
96ec7ce8d0b9  docker.io/library/hello-world:latest                                                            /hello      2 hours ago    Exited (0) 2 hours ago                             blissful_volhard
636ec84eb358  docker.io/kindest/node@sha256:f52781bc0d7a19fb6c405c2af83abfeb311f130707a0e219175677e366cc45d1              4 minutes ago  Up 4 minutes ago        127.0.0.1:46251->6443/tcp  kind-control-plane
```

Let's take a look at our cluster using `kubectl`:
```bash
kubectl get nodes
```

We should see something like the following:
```bash
NAME                 STATUS   ROLES           AGE     VERSION
kind-control-plane   Ready    control-plane   6m39s   v1.25.3
```

Taking a look at all the resources in our KIND cluster:
```bash
kubectl get all -A | more
```

We should see something like the following:
```bash
NAMESPACE            NAME                                             READY   STATUS    RESTARTS   AGE
kube-system          pod/coredns-565d847f94-m9tbm                     1/1     Running   0          6m29s
kube-system          pod/coredns-565d847f94-nc7ch                     1/1     Running   0          6m29s
kube-system          pod/etcd-kind-control-plane                      1/1     Running   0          6m40s
kube-system          pod/kindnet-bp4m8                                1/1     Running   0          6m29s
kube-system          pod/kube-apiserver-kind-control-plane            1/1     Running   0          6m40s
kube-system          pod/kube-controller-manager-kind-control-plane   1/1     Running   0          6m40s
kube-system          pod/kube-proxy-d2drf                             1/1     Running   0          6m29s
kube-system          pod/kube-scheduler-kind-control-plane            1/1     Running   0          6m40s
local-path-storage   pod/local-path-provisioner-684f458cdd-md4m7      1/1     Running   0          6m29s

NAMESPACE     NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
default       service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP                  6m43s
kube-system   service/kube-dns     ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   6m41s

NAMESPACE     NAME                        DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
kube-system   daemonset.apps/kindnet      1         1         1       1            1           <none>                   6m38s
kube-system   daemonset.apps/kube-proxy   1         1         1       1            1           kubernetes.io/os=linux   6m41s

NAMESPACE            NAME                                     READY   UP-TO-DATE   AVAILABLE   AGE
kube-system          deployment.apps/coredns                  2/2     2            2           6m41s
local-path-storage   deployment.apps/local-path-provisioner   1/1     1            1           6m37s

NAMESPACE            NAME                                                DESIRED   CURRENT   READY   AGE
kube-system          replicaset.apps/coredns-565d847f94                  2         2         2       6m29s
local-path-storage   replicaset.apps/local-path-provisioner-684f458cdd   1         1         1       6m29s
```

***So, we have a KIND cluster.  What can we do with it?***

### Running a Pod in Our KIND Cluster

While having a running KIND cluster is amazing, using it to do Kubernetes things is even better.  Let's give it a shot!

Let's see if we have any pods running in the `default` namespace:
```bash
kubectl get pods -o wide
```

As we can see, there are no pods running in the `default` namespace:
```bash
No resources found in default namespace.
```

Let's create an `nginx` pod, named `nginx-kind-test`:
```bash
kubectl run nginx-kind-test --image=nginx
```

Checking for pods in the `default` namespace:
```bash
kubectl get pods -o wide
```

We should see something like the following:
```bash
NAME              READY   STATUS    RESTARTS   AGE    IP           NODE                 NOMINATED NODE   READINESS GATES
nginx-kind-test   1/1     Running   0          2m8s   10.244.0.5   kind-control-plane   <none>           <none>
```

Let's delete the `nginx-kind-test` pod:
```bash
kubectl delete pod nginx-kind-test
```

Checking for pods in the `default` namespace:
```bash
kubectl get pods -o wide
```

As we can see, there are no pods running in the `default` namespace:
```bash
No resources found in default namespace.
```

***Pretty cool!  We have our own little KIND cluster to work with, running rootlessly under Podman as a container!***

## Deleting Our KIND Cluster

Let's try deleting our cluster:
```bash
kind delete cluster
```

We should see output similar to this:
```bash
enabling experimental podman provider
Deleting cluster "kind" ...
```

Let's check for existing KIND clusters before we proceed:
```bash
kind get clusters
```

We shouldn't see any clusters:
```bash
enabling experimental podman provider
No kind clusters found.
```

## Summary

Using the power of Podman and KIND, we can build a nice, lightweight K8s environment on Ubuntu (or other distributions!) to learn and experiment with Kubernetes.

If you'd like to learn some more advanced KIND concepts, check out the **KIND: Quick Start** link in the **References** section.

Enjoy!

*Tom Dean*
