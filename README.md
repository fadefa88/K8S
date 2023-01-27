# Create your Kubernetes cluster on premise and integrate it with Cloud One - Container Security

 

For the environment i've used a minimal K8S cluster with two Ubuntu server 20.04 and our Product Cloud infrastructure.

This tutorial is similar to Markus playground, however it requires a longer deployment and on the other hand it reduces the costs of the cloud.

 <br><br>

 

## Requirements

> ***Note:*** Container Security supports Kubernetes 1.14 or newer.
<br>

- 2 Ubuntu 20.04 server, with 2 CPU and 2GB of RAM minimum.
<br>

- You can add your VM directly in your Product Cloud vAPP by using the below configuration:
![image](https://user-images.githubusercontent.com/62143875/215054353-1397aa14-ff2e-48ab-839c-fe7918237235.png)
<br>
<br>


- I have called the first node `cp-node` which is the master node with IP 192.168.7.10, and the second `worker-node` which is the slave with IP 192.168.7.11.
All the nodes must talk to each others, so if you don't have a DNS in place, you have to edit the file /etc/hosts of both Ubuntu servers:

```sh
cp-node:~$ sudo vim /etc/hosts
```
- Then add the entries of `cp-node` and `worker-node`:
![image](https://user-images.githubusercontent.com/62143875/215043462-98134e2c-6bcd-4d38-8c56-687e07385cf6.png)


- Repeat the same steps on the `worker-node` as well.
<br>
 


 

## Install Docker Environment

 

- Tu run Kubernetes properly, you need a container runtime environment and in this tutorial I'm using a simple Docker environment. So let's install Docker on both Ubuntu nodes.


```sh
cp-node:~$ sudo apt update
```
```sh
sudo apt upgrade
```
```sh
sudo apt install docker.io
```

- Now let's enable docker and check if the service is running.
```sh
sudo systemctl enable docker
```
```sh
sudo systemctl status docker   
```
- We also need to disable swap memory otherwise Kubernetes wouldn't start
```sh
sudo swapoff -a
```
```sh
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```
<br>
 



 

## Install Kubernetes v. 1.23 (not the latest version!)

On both Ubuntu servers type the below commands to install Kubernetes (with the latest versions of Kubernetes, Smart Check is deprecated. While we wait for the new Cloud replacement, we can still test Smart Check on a previous K8S release).

- First, we need to install http, https and curl packets

```sh
cp-node:~$ sudo apt-get install apt-transport-https curl
```
- Let's add the public Kubernetes key:
```sh
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add
```
- Add the latest Kubernetes repo:
```sh
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
```
- Now we can install Kubernetes v. 1.23:
```sh
sudo apt-get update
```
```sh
sudo apt-get install kubelet=1.23.0-00 kubeadm=1.23.0-00 kubectl=1.23.0-00
```
```sh
sudo apt-mark hold kubelet kubeadm kubectl
```

- On both master and worker nodes, update the cgroupdriver with the following commands: 

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
- Then, execute the following commands to restart and enable Docker on system boot-up:  
```sh
sudo systemctl enable docker
```
```sh
sudo systemctl daemon-reload
```
```sh
sudo systemctl restart docker
```
<br>
 

 

## Deploy a Pod Network

 
- Let's start by initializing the Kubernetes cluster and launch this command on the `cp-node` only: 
```sh
cp-node:~$ sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```
> ***Note:*** 10.244.0.0/16 it's a virtual network and it shouldn't be on the same network of `cp-node` and `worker-node` (for this tutorial my servers are in a 192.168.0.0/20 cidr).

- Wait for Kubernetes control-plane initialization and take note of the last kubeadm join commmand. You will need it later for joining the worker node to this one. In my environment this is the command output:
![image (1)](https://user-images.githubusercontent.com/62143875/215048053-ff380893-397b-492b-97c3-fd351de2badb.png)
- so I'm pasting that command somewhere:
```sh
sudo kubeadm join 192.168.7.10:6443 --token 0svxov.3a7nxiqruo1iz4bj --discovery-token-ca-cert-hash sha256:797d55a78a64ab77c491dea8584b26ad6b93b8fffbda39d1294e1ad25a9ec92e
```

- Still on the master node, you need few commands to initialize the cluster properly:
```sh
mkdir -p $HOME/.kube
```
```sh
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
``` 
```sh
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

- After that, you can run the following command to deploy the pod network on the master node:

```sh
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```
<br>


## Joining Worker Node to the Kubernetes Cluster

With the kubernetes-master node up and the pod network ready, we can join our worker nodes to the cluster. In this tutorial, we only have one worker node, so we will be working with that.

- First, log into your worker node. You will use your kubeadm join command that was shown in your terminal when we initialized the master node:

```sh
worker-node:~$ sudo kubeadm join 192.168.7.10:6443 --token 0svxov.3a7nxiqruo1iz4bj --discovery-token-ca-cert-hash sha256:797d55a78a64ab77c491dea8584b26ad6b93b8fffbda39d1294e1ad25a9ec92e
```
- You should see a similar output like the screenshot below when it completes joining the cluster:
![image (2)](https://user-images.githubusercontent.com/62143875/215049615-f5b1e8a4-993c-484c-bc2c-7bd0a9b1878a.png)

- Once the joining process completes, switch the master node terminal and execute the following command to confirm that your worker node has joined the cluster:
```sh
cp-node:~$ kubectl get nodes
```
- In the screenshot from the output of the command above, we can see that the worker node has joined the cluster:
![unnamed](https://user-images.githubusercontent.com/62143875/215050385-da613b4d-6f4a-432d-96f0-5dd1df031875.png)
<br>


## Deploying an Application (nginx) to the Kubernetes Cluster

At this point, we have set up a running Kubernetes cluster. Let’s try to deploy a service to it. We will test the cluster by deploying the Nginx webserver
- Execute the following command on the master node to create a Kubernetes deployment for Nginx:


```sh
cp-node:~$ kubectl create deployment nginx --image=nginx
```
- To make the nginx service accessible via the internet, run the following command:
```sh
kubectl create service nodeport nginx --tcp=80:80
```
The command above will create a public-facing service for the Nginx deployment. This being a nodeport deployment, Kubernetes assigns the service a port in the range of 32000+.
- You can get the current services by issuing the command:
```sh
kubectl get svc
```
![image](https://user-images.githubusercontent.com/62143875/215068400-bf125b4c-b1e3-4436-86c0-c3866cd4ea62.png)
- You can see that our assigned port is 32634. Now you can visit the worker node IP address and port combination in your browser and view the default Nginx index page:
![image](https://user-images.githubusercontent.com/62143875/215068592-a55df539-46a1-42ea-997c-9635669e6221.png)


- You can delete a deployment by specifying the name of the deployment. For example, this command will delete our deployment:
```sh
kubectl delete deployment nginx
```

Congratulations! Your Kubernetes cluster is now up and running
<br>

## Let's integrate with Cloud One - Container Security
As we created the Kubernetes cluster, is now time to integrate it with Cloud One - Container Security. 
- Let's start by adding a new cluster:
![image](https://user-images.githubusercontent.com/62143875/215070900-06310618-1b2d-408d-beec-353e49e0c01c.png)
- On your Kubernetes master node, create a file called overrides.yaml
![image](https://user-images.githubusercontent.com/62143875/215071074-8922c4fc-08f7-4823-8d78-2523fcb3a16f.png)

cloudOne:
    apiKey: 2KuGwnfHc3PyY4wBim2afmJO62f
    endpoint: https://container.trend-us-1.cloudone.trendmicro.com
    runtimeSecurity:
        enabled: true
    vulnerabilityScanning:
        enabled: true
    exclusion:
        namespaces: [ kube-system ]



