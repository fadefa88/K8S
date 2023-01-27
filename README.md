# Create your Kubernetes cluster on premise and integrate it with Cloud One - Container Security

 

For the environment i've used a minimal K8S cluster with two Ubuntu server 20.04 and our Product Cloud infrastructure.

This tutorial is similar to Markus playground, however it requires a longer deployment and on the other hand it reduces the costs of the cloud.

 

 

## Requirements

> ***Note:*** Container Security supports Kubernetes 1.14 or newer.

- 2 Ubuntu 20.04 server, with 2 CPU and 2GB of RAM (otherwise the command `kubeadm init` will fail and you won't be able to initialize the cluster)

You can add your VM directly in your vAPP by using the below configuration:
![1](https://user-images.githubusercontent.com/62143875/215043404-54ca7cc7-ff22-43d6-ba5f-fb330c6b2943.PNG)


I have called the first node `cp-node` which is the master node with IP 192.168.7.10, and the second `worker-node` which is the slave with IP 192.168.7.11.
All the nodes must talk to each other, so either you have a DNS or you can edit the file /etc/hosts of both Ubuntu servers

`cp-node:~$`
```sh
sudo vim /etc/hosts
```
and add the entries:
![image](https://user-images.githubusercontent.com/62143875/215043462-98134e2c-6bcd-4d38-8c56-687e07385cf6.png)


Repeat the same steps on the `worker-node` as well.
 


 

## Install Docker Environment

 

Tu run Kubernetes properly, you need a container runtime environment and in this tutorial I'm using a simple Docker environment. So let's install Docker on both Ubuntu nodes.

`cp-node:~$`
```sh
sudo apt update
```
```sh
sudo apt upgrade
```
```sh
sudo apt install docker.io
```

Now let's enable docker and check if the service is running.
```sh
sudo systemctl enable docker
```
```sh
sudo systemctl status docker   
```
We also need to disable swap memory otherwise Kubernetes wouldn't start
```sh
sudo swapoff -a
```
```sh
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```
 



 

## Install Kubernetes v. 1.23 (not the latest one!)

On both Ubuntu servers type the below commands to install Kubernetes (with the latest versions of Kubernetes, Smart Check is not working anymore. While we wait for the new Cloud replacement, we can still test Smart Check on a previous K8S release).

First, we need to install http, https and curl packets

```sh
sudo apt-get install -y apt-transport-https curl
```
Let's add the public Kubernetes key:
```sh
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add
```
Add the latest Kubernetes repo:
```sh
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
```
Now we can install Kubernetes:
```sh
sudo apt-get update
```
```sh
sudo apt-get install -y kubelet=1.23.0-00 kubeadm=1.23.0-00 kubectl=1.23.0-00
```
```sh
sudo apt-mark hold kubelet kubeadm kubectl
```

On both master and worker nodes, update the cgroupdriver with the following commands: 
```sh
sudo mkdir /etc/docker
```
```sh
cat <<EOF | sudo tee /etc/docker/daemon.json
{ "exec-opts": ["native.cgroupdriver=systemd"],
"log-driver": "json-file",
"log-opts":
{ "max-size": "100m" },
"storage-driver": "overlay2"
}
EOF
```
Then, execute the following commands to restart and enable Docker on system boot-up:  
```sh
sudo systemctl enable docker
```
```sh
sudo systemctl daemon-reload
```
```sh
sudo systemctl restart docker
```
 

 

## Deploy a Pod Network

 
Let's start by initializing the Kubernetes cluster and launch this command on the `cp-node` only: 
```sh
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```
10.244.0.0/16 it's a virtual network and it shouldn't be on the same network of `cp-node` and `worker-node` (for this tutorial my servers are in a 192.168.0.0/20 cidr).

Wait for Kubernetes control-plane initialization and take note of the last kubeadm join commmand. You will need it later for joining the worker node to this one. In my environment this is the command output:
![image (1)](https://user-images.githubusercontent.com/62143875/215048053-ff380893-397b-492b-97c3-fd351de2badb.png)
so I'm pasting it somewhere:
```sh
sudo kubeadm join 192.168.7.10:6443 --token 0svxov.3a7nxiqruo1iz4bj --discovery-token-ca-cert-hash sha256:797d55a78a64ab77c491dea8584b26ad6b93b8fffbda39d1294e1ad25a9ec92e
```

Still on the master node, you need few commands to initialize the cluster properly:
```sh
mkdir -p $HOME/.kube
```
```sh
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
``` 
```sh
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```


## Joining Worker Node to the Kubernetes Cluster

With the kubernetes-master node up and the pod network ready, we can join our worker nodes to the cluster. In this tutorial, we only have one worker node, so we will be working with that.
First, log into your worker node. You will use your kubeadm join command that was shown in your terminal when we initialized the master node:
`worker-node:~$`
```sh
sudo kubeadm join 192.168.7.10:6443 --token 0svxov.3a7nxiqruo1iz4bj --discovery-token-ca-cert-hash sha256:797d55a78a64ab77c491dea8584b26ad6b93b8fffbda39d1294e1ad25a9ec92e
```
You should see similar output like the screenshot below when it completes joining the cluster:
![image (2)](https://user-images.githubusercontent.com/62143875/215049615-f5b1e8a4-993c-484c-bc2c-7bd0a9b1878a.png)

Once the joining process completes, switch the master node terminal and execute the following command to confirm that your worker node has joined the cluster:
`cp-node:~$`
```sh
kubectl get nodes
```
In the screenshot from the output of the command above, we can see that the worker node has joined the cluster:
![unnamed](https://user-images.githubusercontent.com/62143875/215050385-da613b4d-6f4a-432d-96f0-5dd1df031875.png)


### Create Playgrounds built-in Cluster

 

Simply run

 

```sh

# Local built-in Cluster

./up.sh

```

 

Typically, you want to deploy the cluster registry next. Do this by running

 

```sh

./deploy-registry.sh

```

 

You can find the authentication instructions within the file `services`.

 

Now, head over to [Deployments](#deployments).

 

### Create GKE, EKS or AKS Clusters

 

Run one of the following scripts to quickly create a cluster in the clouds.

 

```sh

# GKE

./clusters/rapid-gke.sh

 

# AKS

./clusters/rapid-aks.sh

 

# EKS

./clusters/rapid-eks.sh

```

 

You don't need to create a registry here since you're going to use the cloud provided registries GCR, ACR or ECR.

 

## Deployments

 

The playground provides a couple of scripts which deploy preconfigured versions of several products. This includes currently:

 

- Container Security (`./deploy-container-security.sh`)

- Smart Check (`./deploy-smartcheck.sh`)

- Prometheus & Grafana (`./deploy-prometheus.sh`)

- Starboard (`./deploy-starboard.sh`)

- Falco Runtime Security (`./deploy-falco.sh`)

- Open Policy Agent (`./deploy-opa.sh`)

- Gatekeeper (`./deploy-gatekeeper.sh`)

- Harbor (`./deploy-harbor.sh`)

 

In addition to the above the playground now supports AWS CodePipelines. The pipeline builds a container image based on a sample repo, scans it with Smart Check and deploys it with integrated Cloud One Application Security to the EKS cluster.

 

The pipeline requires an EKS with a deployed Smart Check. If everything has been set up, running the script `./deploy-pipeline-aws.sh` should do the trick :-). When you're done with the pipeline run the generated script `./pipeline-aws-down.sh` to tear it down.

 

## Tear Down

 

### Tear Down Ubuntu Local, MacOS Local or Cloud9 Local Clusters

 

```sh

./down.sh

```

 

### Tear Down Pipelines

 

Run one of the following scripts to quickly tear down a pipeline in the clouds. These scripts are created automatically by the pipeline scripts.

 

```sh

# GCP

# ./pipeline-gcp-down.sh

 

# AWS

./pipeline-aws-down.sh

 

# Azure

# ./pipeline-azure-down.sh

```

 

### Tear Down GKE, EKS or AKS Clusters

 

Run one of the following scripts to quickly tear down a cluster in the clouds. These scripts are created automatically by the cluster scripts.

 

```sh

# GKE

./rapid-gke-down.sh

 

# AKS

./rapid-aks-down.sh

 

# EKS

./rapid-eks-down.sh

```

 

## Add-Ons

 

The documentation for the add-ons are located inside the `./docs` directory.

 

- [Cilium](docs/add-on-cilium.md)

- [Container Security](docs/add-on-container-security.md)

- [Falco](docs/add-on-falco.md)

- [Gatekeeper](docs/add-on-gatekeeper.md)

- [Harbor](docs/add-on-harbor.md)

- [Istio](docs/add-on-istio.md)

- [Krew](docs/add-on-krew.md)

- [Kubescape](docs/add-on-kubescape.md)

- [Open Policy Agent](docs/add-on-opa.md)

- [Prometheus & Grafana](docs/add-on-prometheus-grafana.md)

- [Registry](docs/add-on-registry.md)

- [Starboard](docs/add-on-starboard.md)

 

## Play with the Playground

 

If you wanna play within the playground and you're running it either on Linux or Cloud9, follow the lab guide [Play with the Playground (on Linux & Cloud9)](docs/play-on-linux.md).

 

If you're running the playground on MacOS, follow the lab guide [Play with the Playground (on MacOS)](docs/play-on-macos.md).

 

Both guides are basically identical, but since access to some services is different on Linux and MacOS there are two guides available.

 

If you want to play with pipelines, the Playground now supports CodePipeline on AWS. Follow this [quick documentation](docs/pipelining-on-aws.md) to test it out.

 

Lastly, there is a [guide](docs/play-with-falco.md) to experiment with the runtime rules built into the playground to play with Falco. The rule set of the playground is located [here](falco/playground_rules.yaml).

 

## Demo Scripts

 

The Playground supports automated scripts to demonstrate functionalies of deployments. Currently, there are two scripts available showing some capabilities of Cloud One Container Security.

 

To run them, ensure to have an EKS cluster up and running and have Smart Check and Container Security deployed.

 

After configuring the policy and rule set as shown below, you can run the demos with

 

```sh

# Deployment Control Demo

./demos/demo-c1cs-dc.sh

 

# Runtime Security Demo

./demos/demo-c1cs-rt.sh

```

 

### Deployment Control Demo

 

> ***Storyline:*** A developer wants to try out a new `nginx` image but fails since the image has critical vulnerabilities, he tries to deploy from docker hub etc. Lastly he tries to attach to the pod, which is prevented by Container Security.

 

To prepare for the demo verify that the cluster policy is set as shown below:

 

- Pod properties

  - uncheck - containers that run as root

  - Block - containers that run in the host network namespace

  - Block - containers that run in the host IPC namespace

  - Block - containers that run in the host PID namespace

- Container properties

  - Block - containers that are permitted to run as root

  - Block - privileged containers

  - Block - containers with privilege escalation rights

  - Block - containers that can write to the root filesystem

- Image properties

  - Block - images from registries with names that DO NOT EQUAL REGISTRY:PORT

  - uncheck - images with names that

  - Log - images with tags that EQUAL latest

  - uncheck - images with image paths that

- Scan Results

  - Block - images that are not scanned

  - Block - images with malware

  - Log - images with content findings whose severity is CRITICAL OR HIGHER

  - Log - images with checklists whose severity is CRITICAL OR HIGHER

  - Log - images with vulnerabilities whose severity is CRITICAL OR HIGHER

  - Block - images with vulnerabilities whose CVSS attack vector is NETWORK and whose severity is HIGH OR HIGHER

  - Block - images with vulnerabilities whose CVSS attack complexity is LOW and whose severity is HIGH OR HIGHER

  - Block - images with vulnerabilities whose CVSS availability impact is HIGH and whose severity is HIGH OR HIGHER

  - Log - images with a negative PCI-DSS checklist result with severity CRITICAL OR HIGHER

- Kubectl Access

  - Block - attempts to execute in/attach to a container

  - Log - attempts to establish port-forward on a container

 

Most of it should already configured by the `deploy-container-security.sh` script.

 

Run the demo with

 

```sh

./demos/demo-c1cs-dc.sh

```

 

### Runtime Security Demo

 

> ***Storyline:*** A kubernetes admin newbie executes some information gathering about the kubernetes cluster from within a running pod. Finally, he gets kicked by Container Security because of the `kubectl` usage.

 

To successfully run the runtime demo you need adjust the aboves policy slightly.

 

Change:

 

- Kubectl Access

  - Log - attempts to execute in/attach to a container

 

- Exceptions

  - Allow images with paths that equal `docker.io/mawinkler/ubuntu:latest`

 

Additionally, set the runtime rule `(T1543)Launch Package Management Process in Container` to ***Log***. Normally you'll find that rule in the `*_error` ruleset.

 

Run the demo with

 

```sh

./demos/demo-c1cs-rt.sh

```

 

The demo starts locally on your system, but creates a pod in the `default` namespace of your cluster using a slightly pimped ubuntu image which is pulled from my docker hub account. The main demo runs within that pod on the cluster, not on your local machine.

 

The Dockerfile for this image is in `./demos/pod/Dockerfile` for you to verify, but you do not need to build it yourself.

 

## Experimenting

 

Working with Kubernetes is likely to raise the one or the other challenge.

 

### Migrate

 

This tries to solve the challenge to migrate workload of an existing cluster using public image registries to a trusted, private one (without breaking the services).

 

To try it, being in the playground directory run

 

```sh

migrate/save-cluster.sh

```

 

This scripts dumps the full cluster to json files separated by namespace. The namespaces `kube-system`, `registry` and `default` are currently excluded.

 

Effectively, this is a **backup** of your cluster including ConfigMaps and Secrets etc. which you can deploy on a different cluster easily (`kubectl create -f xxx.json`)

 

To migrate the images currently in use run

 

```sh

migrate/migrate-images.sh

```

 

This second script updates the saved manifests in regards the image location to point them to the private registry. If the image has a digest within it's name it is stripped.

 

The image get's then pulled from the public repo and pushed to the internal one. This is followed by an image scan and the redeployment.

 

> Note: at the time of writing the only supported private registry is the internal one.

 

## Testing the Playground

 

The Playground uses [Bats](https://github.com/sstephenson/bats) for unit testing.

 

Install Bats with

 

```sh

# Linux

npm install -g bats

 

# MacOS

brew install bats

```

 

Unit tests are in `./tests`.

 

To run a full tests for a cluster type simply run

 

```sh

# Local Kind cluster

./test-kind-linux.sh

 

# GKE

./test-gke.sh

 

# AKS

./test-aks.sh

 

# EKS

./test-eks.sh

```

 

while being in the playground directory. Make sure, that you're authenticated on AWS, GCP and / or Azure beforehand.

 

The following playground modules will be executed:

 

```

└── Build Cluster

    ├── (Registry)

    ├── Falco

    ├── Smart Check

    ├── Smart Check Scan

    ├── Container Security

    └── Destroy cluster

```

 

## TODO

 

- ...
