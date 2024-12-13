# demo-kubernetes
K8s - Kubernetes demo project.


## Overview
### Kubernetes Distro:<br/>
[K3s Docs](https://docs.k3s.io/) - K3s is a lightweight, easy-to-install Kubernetes distribution that is ideal for a 3-node cluster because it simplifies deployment and management, while providing the full functionality of Kubernetes with a smaller resource footprint.

### Nodes:
```bash
# demo-kubernetes nodes
10.61.20.81 demo-k8s-ctr01  # Control-Plane
10.61.20.82 demo-k8s-wrk01  # Worker
10.61.20.83 demo-k8s-wrk02  # Worker
```
### Networks:
```bash
# Network Ports and CIDRs
6443/tcp        # Kubernetes apiserver port (used by kubectl)
10.42.0.0/16    # Pod Network used by Kubernetes
10.43.0.0/16    # Service Network used byKubernetes
```
<br/>

## Prep Config (optional)
These tasks should be performed on __each node__ prior to Kubernetes installation/configuration.

### Add records to the Hosts file
Adding an entry in the hosts file on each node in the Kubernetes cluster ensures that the nodes can always resolve each other's hostnames, without relying on DNS.
```bash 
$ vi /etc/hosts
```
```bash
# demo-kubernetes nodes
10.61.20.81 demo-k8s-ctr01  # Control-Plane
10.61.20.82 demo-k8s-wrk01  # Worker
10.61.20.83 demo-k8s-wrk02  # Worker
```

### Create Firewall rules
We need to open the firewall ports & networks that will be used 
```bash
# ubuntu (or debian)
sudo ufw allow 6443/tcp #apiserver
sudo ufw allow from 10.42.0.0/16 to any #pods
sudo ufw allow from 10.43.0.0/16 to any #services
```
```bash
# RHEL Based (Oracle Linux)
sudo firewall-cmd --permanent --add-port=6443/tcp #apiserver
sudo firewall-cmd --permanent --zone=trusted --add-source=10.42.0.0/16 #pods
sudo firewall-cmd --permanent --zone=trusted --add-source=10.43.0.0/16 #services
sudo firewall-cmd --reload
```

### Update the OS and packages
Its a good idea before installing additional packages and making system config changes that we upgrade our Operating System and currently installed packages:

```bash
# ubuntu (or debian) - Update
$ sudo apt update -y && sudo apt upgrade -y
```
```bash
# RHEL Based (Oracle Linux) - Update
$ dnf update -y
```

### Bash Prompt directory Length (_ubuntu_)
In Ubuntu the default bash prompt will show the full path of your current working directory. *My prefernce is to show only the current directory.* <br/>
To show the current directory, only, in your bash prompt:

```bash
$ export PROMPT_DIRTRIM=1
```

To make this persistent, set the variable in the `~/.bashrc` file:
```bash
# Sets PROMPT_DIRTRIM for every shell login
PROMPT_DIRTRIM=1
```

<br/>


## Install Kubernetes (K3s) - Server / Control-Plane

| __Node:__ | __`demo-k8s-ctr01`__ |
| --- | --- |
| Type:   | Server |
| Roles:  | control-plane, etcd, master |


### Create Configuration File(s):
By default, values present in a YAML file located at `/etc/rancher/k3s/config.yaml` will be used on install 
(this allows overrides on the default K3s config). <br/>
```sh
$ sudo mkdir -p /etc/rancher/k3s
$ sudo vi /etc/rancher/k3s/config.yaml
```

[K3s Server Options](https://docs.k3s.io/cli/server) - create a config file for the server node (Control-Plane). <br/>
On the server node we will be adding a taint, to make this node not available for pods to be scheduled on.
```yaml
tls-san:
  - "demo-k8s-ctr01"
  - "10.61.20.81"
node-taint: 
  - "node-role.kubernetes.io/master=true:NoSchedule"
disable:
  - "traefik"
  - "servicelb"
cluster-cidr: "10.42.0.0/16"
service-cidr: "10.43.0.0/16"
cluster-init: true
```

### Install K3s on the Control-Plane:
Install K3s, using the convinience script. (by default this will install the latest "Stable" Release)
```bash
$ curl -sfL https://get.k3s.io | sudo sh -
```
If all went well then you should get an output like:
```
[INFO]  Finding release for channel stable
[INFO]  Using v1.31.3+k3s1 as release
[INFO]  Downloading hash https://github.com/k3s-io/k3s/releases/download/v1.31.3+k3s1/sha256sum-amd64.txt
[INFO]  Downloading binary https://github.com/k3s-io/k3s/releases/download/v1.31.3+k3s1/k3s
[INFO]  Verifying binary download
[INFO]  Installing k3s to /usr/local/bin/k3s
[INFO]  Skipping installation of SELinux RPM
[INFO]  Creating /usr/local/bin/kubectl symlink to k3s
[INFO]  Creating /usr/local/bin/crictl symlink to k3s
[INFO]  Creating /usr/local/bin/ctr symlink to k3s
[INFO]  Creating killall script /usr/local/bin/k3s-killall.sh
[INFO]  Creating uninstall script /usr/local/bin/k3s-uninstall.sh
[INFO]  env: Creating environment file /etc/systemd/system/k3s.service.env
[INFO]  systemd: Creating service file /etc/systemd/system/k3s.service
[INFO]  systemd: Enabling k3s unit
Created symlink /etc/systemd/system/multi-user.target.wants/k3s.service → /etc/systemd/system/k3s.service.
[INFO]  systemd: Starting k3s
```

### Perform initial checks:
Is the cluster running?
```bash
$ sudo kubectl get nodes

...
NAME             STATUS   ROLES                       AGE     VERSION
demo-k8s-ctr01   Ready    control-plane,etcd,master   3m43s   v1.31.3+k3s1
```

Are containers/pods running?
```bash
# Check Pods are running using kubectl
$ sudo kubctl get pods -A

# Check pods/containers are running using crictl
$ sudo crictl ps
```

Check the control-plane node labels:
```bash
$ sudo kubectl get nodes demo-k8s-ctr01 --show-labels

...
NAME             STATUS   ROLES                       AGE   VERSION        LABELS
demo-k8s-ctr01   Ready    control-plane,etcd,master   41m   v1.31.3+k3s1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/instance-type=k3s,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=demo-k8s-ctr01,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=true,node-role.kubernetes.io/etcd=true,node-role.kubernetes.io/master=true,node.kubernetes.io/instance-type=k3s
```

Check the control-plane node taints, this should match what we set in the `/etc/rancher/k3s/config.yaml` config:
```bash
$ sudo kubectl describe node demo-k8s-ctr01 | grep -i 'Taints:'

...
Taints:             node-role.kubernetes.io/master=true:NoSchedule
```

<br/>

### Copy Node Join Token to Agents:
Before we install K3s on the Agents we should copy the join token so this can be called during the Agent install process.
```bash
# copy the node-token to home, and chown
$ sudo cp /var/lib/rancher/k3s/server/node-token ~/k3s-node-token && sudo chown $(whoami):$(whoami) ~/k3s-node-token

# now copy this file over to each node using scp
$ scp ~/k3s-node-token demo-k8s-wrk01:~/k3s-node-token
$ scp ~/k3s-node-token demo-k8s-wrk02:~/k3s-node-token

# finally remove the join token from the home directory
$ rm -f ~/k3s-node-token
```
<br/>

## Install Kubernetes (K3s) - Agent / Workers

| __Node:__ | __`demo-k8s-wrk01` / `demo-k8s-wrk02`__ |
| --- | --- |
| Type:   | Agent/Worker |
| Roles:  | n/a |

The following tasks need to be ran on each of the worker nodes.

Create an SSH session on each to `demo-k8s-wrk01` and `demo-k8s-wrk02`

### Install K3s on the Agent:
_run on the __agent__ nodes_ <br/>
[K3s Agent CLI Options](https://docs.k3s.io/cli/agent) - Additional options are available to be passed over as part of the install. <br/>
To install as an agent/worker node we need to specify the `K3S_URL` to connect to and the `K3S_TOKEN_FILE` we copied over earlier.

```bash
# Elevate to root
$ sudo su

# Install K3s as an Agent, connected to our demo-k8s-ctr01 cluster
$ curl -sfL https://get.k3s.io | K3S_URL=https://demo-k8s-ctr01:6443 K3S_TOKEN_FILE=/tmp/k3s-node-token sh -s -

...
[INFO]  Finding release for channel stable
[INFO]  Using v1.31.3+k3s1 as release
[INFO]  Downloading hash https://github.com/k3s-io/k3s/releases/download/v1.31.3+k3s1/sha256sum-amd64.txt
[INFO]  Downloading binary https://github.com/k3s-io/k3s/releases/download/v1.31.3+k3s1/k3s
[INFO]  Verifying binary download
[INFO]  Installing k3s to /usr/local/bin/k3s
[INFO]  Skipping installation of SELinux RPM
[INFO]  Creating /usr/local/bin/kubectl symlink to k3s
[INFO]  Creating /usr/local/bin/crictl symlink to k3s
[INFO]  Creating /usr/local/bin/ctr symlink to k3s
[INFO]  Creating killall script /usr/local/bin/k3s-killall.sh
[INFO]  Creating uninstall script /usr/local/bin/k3s-agent-uninstall.sh
[INFO]  env: Creating environment file /etc/systemd/system/k3s-agent.service.env
[INFO]  systemd: Creating service file /etc/systemd/system/k3s-agent.service
[INFO]  systemd: Enabling k3s-agent unit
Created symlink /etc/systemd/system/multi-user.target.wants/k3s-agent.service → /etc/systemd/system/k3s-agent.service.
[INFO]  systemd: Starting k3s-agent

# Exit from elevated session
$ exit
```

### Check Nodes in the Cluster:
_run on the __server__ node_ <br/>
Now connect back to your server node `demo-k8s-ctr01` and let check to see if the cluster has the newly added nodes. 
```bash
# Get the nodes, but lets output wide (showing more information).
$ sudo kubectl get nodes -o wide

...
NAME             STATUS   ROLES                       AGE     VERSION        INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
demo-k8s-ctr01   Ready    control-plane,etcd,master   142m    v1.31.3+k3s1   10.61.20.81   <none>        Ubuntu 24.04.1 LTS   6.8.0-50-generic   containerd://1.7.23-k3s2
demo-k8s-wrk01   Ready    <none>                      13m     v1.31.3+k3s1   10.61.20.82   <none>        Ubuntu 24.04.1 LTS   6.8.0-31-generic   containerd://1.7.23-k3s2
demo-k8s-wrk02   Ready    <none>                      6m52s   v1.31.3+k3s1   10.61.20.83   <none>        Ubuntu 24.04.1 LTS   6.8.0-31-generic   containerd://1.7.23-k3s2
```

### Remove the join token:
_run on the __agent__ nodes_ <br/>
Now that the Agent nodes have joined the cluster we should remove the k3s-node-token we copied over earlier.
```bash
rm /tmp/k3s-node-token
```
<br/>

## Play Time 
That's it we now have a 3 node kubernetes cluster (1 Server, and 2 Agents).<br/>
The Agents (worker) nodes will run the containers, and the servers role is to manage the Kubernetes API, and control the scheduling of the pods/containers to the appropriate agent.<br/>
We can now have a little play before any "production" workload is released to the cluster.

### Create Pods and perform checks
_run on the __server__ node_ <br/>
In this section we will create two webservers, and a debugging pod: 
* web01 - nginx webserver
* web02 - httpd webserver
* debug - alpine linux (really small linux distro).

```bash
# create pod named nginx01 using the nginx container image
$ sudo kubectl run web01 --image=nginx

# create pod named nginx01 using the nginx container image
$ sudo kubectl run web02 --image=httpd
```
OK lets check the status of these pods and check they are running, and make a note of the IP Addresses they've been assigned.
```bash
$ sudo kubectl get pods -o wide

...
NAME    READY   STATUS    RESTARTS   AGE   IP          NODE             NOMINATED NODE   READINESS GATES
web01   1/1     Running   0          32s   10.42.3.4   demo-k8s-wrk01   <none>           <none>
web02   1/1     Running   0          26s   10.42.4.6   demo-k8s-wrk02   <none>           <none>
```

Great the pods are running and we have the IP Addresses.<br/>
__note: these addresses are in the CIDR range we defined for the pod-network_ <br/>
Lets run a pod that we will connect into interactive shell, and we'll set it to remove when we exit.
```bash
# Create the debugging pod, and lets curl the webservers we just created.
$ sudo kubectl run -i -t --rm debug --image=alpine -- /bin/sh

...
If you don't see a command prompt, try pressing enter.
/ #
```
Now we have a terminal shell session directly into the pod, lets do some debugging of the webservers.
```sh
# Lets ping the pods IP Addresses and see if we get a response
$ ping 10.42.3.4
$ ping 10.42.3.4

# Now lets curl the webserver pods, to do this we need to install curl (as this is a minimal pod).
$ apk add curl

# web01 website test
$ curl -v http://10.42.3.4
...
< Server: nginx/1.27.3
<title>Welcome to nginx!</title>


# web02 website test
$ curl -v http://10.42.4.6
...
< Server: Apache/2.4.62 (Unix)
<html><body><h1>It works!</h1></body></html>

# Everything looks good lets exit from the debugging pod
exi
```

Lets check the pod is removed (or removing)
```bash
$ sudo kubectl get pods
```
