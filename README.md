

Production-grade Kubernetes environment
---------------------------------------

You were tasked with defining the architecture for your company’s new self-managed Kubernetes environment. The only requirements are:

*   Ubuntu 24.04 as the base OS
    
*   Kubernetes version must be v1.30.3

Hey. I have set up the Kubernetes environment with 3 masters and 2 nodes to create a multi-master Kubernetes cluster using the “kubeadm” self-manged service.


![syself-dev](https://github.com/user-attachments/assets/e94a4bbf-3e3f-4adb-a5b1-1e336b617109)

Pre-requisite:

*   3 machines for master, ubuntu 24.04, 2CPU, 4GB RAM
    
*   2 machines for worker, ubuntu 24.04, 1 CPU, 1GB RAM
    
*   1 machine for loadbalancer, ubuntu 24.04, 1 CPU, 1GB RAM
    
*   All machines must be accessbile on the internet.
    
*   sudo privelege
    

**1\. Setting up loadbalancer**
-------------------------------

In this setup, I have used HAPROXY as the primary loadbalancer.

### **What we are loadbalancing?**

We have 3 master nodes, which means the user can connect to either of the 3 api-servers. The loadbalancer will used to loadbalance between the 3 api-servers.

### **a. Login to the loadbalancer node, switch to the root user and update your repository and your system**

```sudo apt-get update && sudo apt-get upgrade -y```

### **b. Install haproxy**

`sudo apt-get install haproxy -y`

### **c. Edit haproxy configuration**

```vi /etc/haproxy/haproxy.cfg```

Add the below lines to create a frontend configuration for loadbalancer:

```
frontend fe-apiserver
    bind 0.0.0.0:6443   
    mode tcp    
    option tcplog    
    default\_backend be-apiserver
```

Add the below lines to create a backend configuration for master 1, master 2, and master 3 nodes at port 6443. Note: 6443 is the default port of of kube-apiserver

```
frontend be-apiserver
    mode tcp
    option tcplog
    balance roundrobin
    default_server inter 10s downinter 5s rise 2 fall 2 slowstart 60s maxconn 250 maxqueue 256 weight 100
              server master1 <Private-ip of master1>:6443 check
              server master2 <Private-ip of master2>:6443 check
              server master3 <Private-ip of master3>:6443 check
```
### **d. Restart and Verify haproxy**

```
systemctl restart haproxy
systemctl status haproxy
```

Make sure your haproxy is in running status.

You can even check if 6443 port is open or not by running this:

```
nc -v localhost 6443
```

**2\. Turn off swap**
---------------------

By disabling swap, Kubernetes can maintain predictable performance, accurate resource allocation, and reliable memory management

Apply below commands on master1, master2, master3, worker1, worker2

```
swapoff -a
sudo sed -i '/ swap / s/^\\\\(.\*\\\\)$/#\\\\1/g' /etc/fstab
```

**3\. Install Container Runtime**
---------------------------------

A container runtime is required on every node in a Kubernetes cluster because it is responsible for running the containers that make up your applications. Without a container runtime, the node wouldn't be able to execute the containers required to run your applications.

Apply below commands on master1, master2, master3, worker1, worker2)

Pre-requisite:

**Enable IPv4 packet forwarding**

sysctl params required by setup, params persist across reboots


 ``` cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf net.ipv4.ip_forward = 1 EOF```

Apply sysctl params without reboot sudo sysctl --system


Verify that net.ipv4.ip_forward is set to 1:


```sysctl net.ipv4.ip_forward ```

I have installed container-d on my allnodes(master1, master2, master3, worker1, worker2) following this [documentation](https://docs.docker.com/engine/install/ubuntu/).

After successfully installing conatinerd, set /etc/containerd/config.toml to:
```
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]

  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true
```
After this change, make sure to restart the conatiner-d:

```
sudo systemctl restart containerd
```
**4\. Install kubeadm and kubelet**
-----------------------------------

These commands are for Kubernetes v1.30:

a.  Update the apt package and install needed packages
    

```
sudo apt-get updatesudo apt-get install -y apt-transport-https ca-certificates curl gpg
```

b.  Download the public signing key for the Kubernetes package repositories:
    

```
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

c.  Add the appropriate Kubernetes apt repository.

    
```
echo 'deb \[signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg\] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

d.  Update the apt package, install kubelet, kubeadm:
    

```
sudo apt-get updatesudo apt-get install -y kubelet kubeadmsudo apt-mark hold kubelet kubeadm

systemctl daemon-reload

systemctl start kubelet

systemctl enable kubelet.service
```

**5\. Configure kubeadm to bootstrap the cluster**
--------------------------------------------------

First, we initialize our first master node. We will use master1 to initialize our first control plane.

a.  Login to the master1, Switch to the root user, and execute the following command to initialize the cluster:
    

```
kubeadm init --control-plane-endpoint "LOAD\_BALANCER\_IP:LOAD\_BALANCER\_PORT" --upload-certs
```

Here, the `LOAD\_BALANCER\_IP` is the IP Address of the loadbalancer. We have kept loadbalncer port at 6443. So the configuration looks like: 

```
kubeadm init --control-plane-endpoint "LOAD\_BALANCER\_IP:6443" --upload-certs
```

This command successfully generates the kubeconfig file containing loadbalancerip.

Your output looks like:


```
To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join 172.31.42.74:6443 --token cnslau.kd5fjt96jeuzymzb \
    --discovery-token-ca-cert-hash sha256:871ab3f050bc9790c977daee9e44cf52e15ee37ab9834567333b939458a5bfb5 \
    --control-plane --certificate-key 824d9a0e173a810416b4bca7038fb33b616108c17abcbc5eaef8651f11e3d146

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use 
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 172.31.42.74:6443 --token cnslau.kd5fjt96jeuzymzb \
    --discovery-token-ca-cert-hash sha256:871ab3f050bc9790c977daee9e44cf52e15ee37ab9834567333b939458a5bfb5 

```

The output consists of 3 major tasks:

1.  Setup kubeconfig using:
    
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

2.  Setup new controlplane
    

```
    kubeadm join 172.31.42.74:6443 --token cnslau.kd5fjt96jeuzymzb \
    --discovery-token-ca-cert-hash sha256:871ab3f050bc9790c977daee9e44cf52e15ee37ab9834567333b939458a5bfb5 \
    --control-plane --certificate-key 824d9a0e173a810416b4bca7038fb33b616108c17abcbc5eaef8651f11e3d146
```

3.  Join worker node using:
    

```
kubeadm join 172.31.42.74:6443 --token cnslau.kd5fjt96jeuzymzb \
    --discovery-token-ca-cert-hash sha256:871ab3f050bc9790c977daee9e44cf52e15ee37ab9834567333b939458a5bfb5
```

Note: While you are setting up the cluster ensure that you are executing the command provided by your output and don't copy and paste from here and save the output in some secure file for future use.

b. Login to master2 and master3, switch to the root user, and execute the kubeadm join command to nodes to the controlplane:

```
kubeadm join 172.31.42.74:6443 --token cnslau.kd5fjt96jeuzymzb \
    --discovery-token-ca-cert-hash sha256:871ab3f050bc9790c977daee9e44cf52e15ee37ab9834567333b939458a5bfb5 \
    --control-plane --certificate-key 824d9a0e173a810416b4bca7038fb33b616108c17abcbc5eaef8651f11e3d146
```
    

Your output should be like: 


```
This node has joined the cluster and a new control plane instance was created:

* Certificate signing request was sent to apiserver and approval was received.
* The Kubelet was informed of the new secure connection details.
* Control plane (master) label and taint were applied to the new node.
* The Kubernetes control plane instances scaled up.
* A new etcd member was added to the local/stacked etcd cluster.
```

c. We have initialized both the master nodes, now we can work on bootstrapping the worker nodes:

Login to worker1 and worker2, switch to the root user, and execute the kubeadm join command on both workers:

```
kubeadm join 172.31.42.74:6443 --token cnslau.kd5fjt96jeuzymzb \
    --discovery-token-ca-cert-hash sha256:871ab3f050bc9790c977daee9e44cf52e15ee37ab9834567333b939458a5bfb5
```

Your output should look like:

```
This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.
```


**6\. Configure kubeconfig and kubectl on loadbalancer node**
-------------------------------------------------------------

We will set the kubectl utility on our loadbalancer node as it is a good practice because when we use kubectl command to make a connection with api-server, the request goes through the loadbalancer that we have configured. 

Kubectl requires kubeconfig file to work with the api-server as it contains the cluster information, user credentials, and context.

Let's configure the kubeconfig file:

a.  Login to loadbalancer node, switch to the root user
    
b.  Create a directory - .kube at $HOME of root
    

`mkdir -p $HOME/.kube`

c.  SCP configuration file from any one master node to loadbalancer node or manually cat the content from master1 create config file in client machine.
    
d.  Provide appropriate ownership to the copied file
    

`chown $(id -u):$(id -g) $HOME/.kube/config`

e. Install kubectl binary

`snap install kubectl --classic`

f. Verify the cluster

`kubectl get nodes`

It gives the output with of all the nodes but they are in notready state because there is no networking between the pods or in the cluster

6\. Install CNI

Configuring CNI is essential for pod networking, IP management, and enforcing network policies.

From the loadbalancer(Where u have kubectl configured) node execute:

`kubectl apply -f https://reweave.azurewebsites.net/k8s/v1.30/net.yaml`

This installs CNI component to your cluster.

You can now verify your HA cluster using:

`Kubectl get nodes`


```
NAME         STATUS     ROLES     AGE       VERSION
worker-1     Ready     <none>     1d        v1.30.3
worker-2     Ready     <none>    1d         v1.30.3
```


YAYY!!We are done with setting high availability k8s cluster with multiple master and worker nodes with the primary requirement of Ubuntu 24.04 as the base OS and Kubernetes version of v1.30.3.

### **Best Practices for Production Kubernetes Clusters:**

**CSI Driver**:

 CSI drivers provide a standardized interface between Kubernetes and storage systems, enabling dynamic provisioning and management of persistent volumes. CSI enhances Kubernetes' storage capabilities, making the platform more flexible, powerful, and adaptable to various storage needs in both on-premises and cloud environments.
 
**Security:**

Implement RBAC (Role-Based Access Control) for access

Use Network Policies to control pod-to-pod communication

Regularly update and patch all components

**Monitoring and Logging:**

Set up a monitoring solution like Prometheus and Grafana

Implement centralized logging with solutions like ELK stack or Loki.

**Backup and Disaster Recovery:**

Regularly backup etcd data.

**Resource Management:**

Use Resource Quotas and Limit Ranges to control resource consumption

**Update Strategy:**

Plan for regular updates of Kubernetes and all components.

**Secret Management:**

Use a secure secret management solution like HashiCorp Vault.

**Node Management:**

Implement GitOps practices using tools like ArgoCD or Flux for declarative and version-controlled cluster management

[Link to whole doc](https://docs.google.com/document/d/17nPiqx7nn3Wwj51JWcgsCzt5dGhsSkSOzJujUqgv98k/edit?usp=sharing)
