# Create your Kubernetes cluster on premise and integrate it with Cloud One - Container Security

- [Create your Kubernetes cluster on premise and integrate it with Cloud One - Container Security](#create-your-kubernetes-cluster-on-premise-and-integrate-it-with-cloud-one---container-security)
  - [Requirements](#requirements)
  - [Install Docker Environment](#install-docker-environment)
  - [Install Kubernetes v. 1.23 (not the latest version!)](#install-kubernetes-v-123-not-the-latest-version)
    - [Deploy a Pod Network](#deploy-a-pod-network)
    - [Joining Worker Node to the Kubernetes Cluster](#joining-worker-node-to-the-kubernetes-cluster)
    - [Deploying an Application (nginx) to the Kubernetes Cluster](#deploying-an-application-nginx-to-the-kubernetes-cluster)
  - [Install Helm](#install-helm)
  - [Let's integrate with Cloud One - Container Security](#lets-integrate-with-cloud-one---container-security)
<br>

 

For the environment i've used a minimal K8S cluster with two Ubuntu server 22.04 and our Product Cloud infrastructure.

This tutorial is similar to Markus playground, however it requires a longer deployment and on the other hand it reduces the costs of the cloud.

 <br><br>

 

## Requirements

> ***Note:*** Container Security supports Kubernetes 1.14 or newer.
<br>

- 2 Ubuntu 22.04 server, with 2 CPU and 2GB of RAM minimum.
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
<img src="https://user-images.githubusercontent.com/62143875/215043462-98134e2c-6bcd-4d38-8c56-687e07385cf6.png" width="50%" onclick="window.open('anotherpage.html', '_blank');" />


- Repeat the same steps on the `worker-node` as well.
<br>
 


 

## Install Docker Environment

 

- Tu run Kubernetes properly, you need a container runtime environment and in this tutorial I'm using a simple Docker environment. So let's install Docker on both Ubuntu nodes. Before we need to remove the floppy: the device doesn't have a floppy drive, but the floppy driver module is installed, so you have /dev/fd0, and many things will try to use it. You can remove it by launching these commands on both ubuntu machines:

```sh
sudo rmmod floppy
echo "blacklist floppy" | sudo tee /etc/modprobe.d/blacklist-floppy.conf
sudo dpkg-reconfigure initramfs-tools
```

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
sudo curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo gpg --dearmor -o /usr/share/keyrings/kubernetes-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list > /dev/null
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
 

 

### Deploy a Pod Network

 
- Let's start by initializing the Kubernetes cluster and launch this command on the `cp-node` only: 
```sh
cp-node:~$ sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```
> ***Note:*** 10.244.0.0/16 it's a virtual network and it shouldn't be on the same network of `cp-node` and `worker-node` (for this tutorial my servers are in a 192.168.0.0/20 cidr).

- Wait for Kubernetes control-plane initialization and take note of the last kubeadm join commmand. You will need it later for joining the worker node to this one. In my environment this is the command output:
<img src="https://user-images.githubusercontent.com/62143875/215048053-ff380893-397b-492b-97c3-fd351de2badb.png" width="50%">
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


### Joining Worker Node to the Kubernetes Cluster

With the kubernetes-master node up and the pod network ready, we can join our worker nodes to the cluster. In this tutorial, we only have one worker node, so we will be working with that.

- First, log into your worker node. You will use your kubeadm join command that was shown in your terminal when we initialized the master node:

```sh
worker-node:~$ sudo kubeadm join 192.168.7.10:6443 --token 0svxov.3a7nxiqruo1iz4bj --discovery-token-ca-cert-hash sha256:797d55a78a64ab77c491dea8584b26ad6b93b8fffbda39d1294e1ad25a9ec92e
```
- You should see a similar output like the screenshot below when it completes joining the cluster:
<img src="https://user-images.githubusercontent.com/62143875/215049615-f5b1e8a4-993c-484c-bc2c-7bd0a9b1878a.png" width="50%">

- Once the joining process completes, switch the master node terminal and execute the following command to confirm that your worker node has joined the cluster:
```sh
cp-node:~$ kubectl get nodes
```
- In the screenshot from the output of the command above, we can see that the worker node has joined the cluster:
<img src="https://user-images.githubusercontent.com/62143875/215050385-da613b4d-6f4a-432d-96f0-5dd1df031875.png" width="50%">
<br>


### Deploying an Application (nginx) to the Kubernetes Cluster

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
<img src="https://user-images.githubusercontent.com/62143875/215068400-bf125b4c-b1e3-4436-86c0-c3866cd4ea62.png" width="50%">
- You can see that our assigned port is 32634. Now you can visit the worker node IP address and port combination in your browser and view the default Nginx index page:
<img src="https://user-images.githubusercontent.com/62143875/215068592-a55df539-46a1-42ea-997c-9635669e6221.png" width="80%">


- You can delete a deployment by specifying the name of the deployment. For example, this command will delete our deployment:
```sh
kubectl delete deployment nginx
```

Congratulations! Your Kubernetes cluster is now up and running
<br>
<br>
## Install Helm
For the integration with Cloud One, Helm v3 is required. 
- To install it, simply launch these commands on the master node:
```sh
cp-node:~$ curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
```
Give the script execute permissions.
```sh
chmod 700 get_helm.sh
```
Run the installer.
```sh
sudo ./get_helm.sh
```
<br>
<br>



## Let's integrate with Cloud One - Container Security
As we created the Kubernetes cluster, is now time to integrate it with Cloud One - Container Security. 
- Let's start by adding a new cluster:
![image](https://user-images.githubusercontent.com/62143875/215070900-06310618-1b2d-408d-beec-353e49e0c01c.png)
By clicking on next, you will se some strings to use for the integration with Container Security.
![image](https://user-images.githubusercontent.com/62143875/215074665-43e6390f-95b9-43fe-9e2a-0bf40160f531.png)

- On your Kubernetes master node, create a file called overrides.yaml
```sh
cp-node:~$ sudo vim overrides.yaml
```
- Paste there the first rows:
```sh
cloudOne:
    apiKey: 2KuGwnfHc3PyY4wBim2afmJO62f
    endpoint: https://container.trend-us-1.cloudone.trendmicro.com
    runtimeSecurity:
        enabled: true
    vulnerabilityScanning:
        enabled: true
    exclusion:
        namespaces: [ kube-system ]
```


<img src="https://user-images.githubusercontent.com/62143875/215074509-847bf53c-c34a-4e9d-81e8-ed198878f183.png" width="50%">


- Then you can enroll the cluster `K8S_Cluster` by using the command below:
```sh
helm install \
     trendmicro \
     --namespace trendmicro-system --create-namespace \
     --values overrides.yaml \
     https://github.com/trendmicro/cloudone-container-security-helm/archive/master.tar.gz
```
- Let's go back to Cloud One console and create a New Scanner (which is the Smart Check component):
![image](https://user-images.githubusercontent.com/62143875/215076939-5bc444f6-7dc8-4542-af5f-38624bfb9d6d.png)
By clicking on next, you will see another set of strings to register Smart Check.
![image](https://user-images.githubusercontent.com/62143875/215077496-4f490a30-395b-4f43-b2c3-26f52884a1b5.png)

- Go back to your file overrides.yaml and replace its content with the new strings:
```sh
cp-node:~$ sudo vim overrides.yaml
```
- Replace the content with these new strings:
```sh
cloudOne:
    apiKey: 2KuL4JeZadT50Qznc4wV84lM8kV
    endpoint: https://container.trend-us-1.cloudone.trendmicro.com
```

<img src="https://user-images.githubusercontent.com/62143875/215077984-61a599d0-c8f2-43b8-87d3-aba8f85f887e.png" width="50%">


- You can register scanner K8S_scanner by using the command below:
```sh
helm install \
     deepsecurity-smartcheck \
     --values overrides.yaml \
     https://github.com/deep-security/smartcheck-helm/archive/master.tar.gz --set auth.secretSeed={password}
```
> ***Note:*** This is not on the Cloud One guide, but in the above command you must add the command `--set auth.secretSeed={password}` where `{password}` can be whathever you like
<br>
Now you can play with all the policies of Cloud One - Container Security



