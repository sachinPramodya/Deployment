## Kubernetes Single Master Installation Guide

### Step 1 : Install prerequisie libraries and settings
Set Kubetentes Version
```export  K8S_VERSION=1.14.2-00```

If there are newer versions available, you can find the versions using the following command
```apt list -a kubeadm```

Configure all the nodes with 
```sh prerequisites-kubernetes.sh```

### Step 2: Initialize master

Kubeadm Init on master

Before continuing, Please configure the following in kubeadm-config.yaml
#### Step 2.1: Set up kubernetes version
as per the previous step, you have to put the kuberntees version in the kubeadm-config.yaml file.
```
kubernetesVersion: <kubernetes-version>
```
You can do so like following. If the $K8S_VERSION is 1.14.2-00, then the k8s version would be v1.14.2
```
sed -i "s/<kubernetes-version>/v1.14.2/g"  kubeadm-config.yaml
```

#### Step 2.2: Configure API SANs
In order for your k8s API to work with kubectl and other applications with https it is essential that you enable the following in kubeadm config.
```
apiServer:
  certSANs:
  - 127.0.0.1
  - <loadbalancer_internal_ip>
  - <loadbalancer_external_ip>
  - <loadbalancer_subdomain> (eg: kubernetes.ecdev.lk)
```
to make this easier, what you can do is, first copy the kubeadm-config.yaml to the VM and run the following
```
export  LB_PRIVATE_IP=<loadbalancer internal ip>
export  LB_PUBLIC_IP=<loadbalancer external ip>
export  LB_DNS_NAME=<loadbalancer subdomain>

sed -i "s/<loadbalancer_internal_ip>/$LB_PRIVATE_IP/g"  kubeadm-config.yaml
sed -i "s/<loadbalancer_external_ip>/$LB_PUBLIC_IP/g"  kubeadm-config.yaml
sed -i "s/<loadbalancer_subdomain>/$LB_PUBLIC_IP/g"  kubeadm-config.yaml
```

#### Step 2.3: Configure Subnets
Have a look at pod subnet and service subnet. Make sure these subnets do not coincide with the subnets that exist in your external network. These are the default configuration. You can change as you see fit for your environment.
Please note that if you change podSubnet, you have to change the subnet in ```calico.yaml``` as well. To do that, open calico.yaml and search for ```CALICO_IPV4POOL_CIDR```
```
networking:
  dnsDomain: cluster.local
  podSubnet: 192.168.100.0/20
  serviceSubnet: 10.225.101.0/24
```

#### Step 2.4: Setup Master with Kubeadm
When you have reviewed all of the above, please proceed with the following command.
Please save up the cluster join command on a secure place as you will require it later on to connect worker nodes.

```kubeadm init --config kubeadm-config.yaml```

Output:
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 10.225.100.130:6443 --token <some-token> \
    --discovery-token-ca-cert-hash <some-hash-value>

#### Step 2.5 Setup kubectl 

Setup Kubectl
```
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```


### Step 4: Setup CNI

Setup CNI - Calico

```kubectl apply -f calico-rbac.yaml```
```kubectl apply -f calico.yaml```

NOTE: this template is tried and tested on kubernetes v1.14.2 
If issues arise on newer versions, please visit 
https://docs.projectcalico.org/v3.8/getting-started/kubernetes/.

Make sure the download the yamls and override the podcidr as mentioned in step 2.3

### Step 5: Join worker nodes to control plane

Join other worker nodes
To do this, ssh into each of the other worker nodes and run the command that you saved up in step 2. It would look like the following
```
kubeadm join 10.225.100.130:6443 --token <some-token> \
    --discovery-token-ca-cert-hash <some-hash-value>
```

