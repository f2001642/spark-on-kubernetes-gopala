# spark-on-kubernetes-gopala

This is a guide to create kubernetes cluster, install spark operator, run and monitor spark applications.

## Table of Contents  
- [Spark on Kubernetes](#spark-on-kubernetes-gopala) 
  - [Prepare VMs](#prepare-vms)  
  - [Create Kubernetes Cluster](#create-kubernetes-cluster)  
  - [Install Spark](#deploy-spark)  
  - [Running Spark Applications](#run-spark-applications)  
  - [Monitoring Spark Applications](#create-vms)  
  - [Troubleshooting](#troubleshooting) 

## Prepare VMs
- Create 3 VMs - 1 for control plane and 2 for worker nodes
- Add Inbound port rules required
  - Control plane ports: 8080,6443,10250,10259,10257,2379-2380
  - Worker node ports: 10250,30000-32767
4. Run the below commands to prepare the VMs for installing Kubernetes cluster
   
Note: Created my VMs using VMSS so the VNET is same and Inbound rules applied for all in one go.

Log in to each VM and run the following commands -

```shell
cat /etc/fstab
sudo sed -i '/swap/d' /etc/fstab
sudo swapoff -a

sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

# <file>
sudo cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

cat /etc/modules-load.d/k8s.conf
sudo modprobe overlay
sudo modprobe br_netfilter

# <file>
sudo cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
lsmod | grep br_netfilter
lsmod | grep overlay
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
```

check and enable firewall if not already
```
systemctl status firewalld
systemctl start firewalld
```

update hostname and firewall - master node ONLY
```
sudo hostnamectl set-hostname master-node
echo $(hostname -i)
echo $(hostname)
sudo -- sh -c "echo $(hostname -i) $(hostname) >> /etc/hosts"
cat /etc/hosts

sudo firewall-cmd --permanent --add-port=6443/tcp
sudo firewall-cmd --permanent --add-port=2379-2380/tcp
sudo firewall-cmd --permanent --add-port=10250/tcp
sudo firewall-cmd --permanent --add-port=10251/tcp
sudo firewall-cmd --permanent --add-port=10252/tcp
sudo firewall-cmd --permanent --add-port=10255/tcp
sudo firewall-cmd --reload
```

update hostname and firewall - worker node ONLY
```
sudo hostnamectl set-hostname worker-node-1
echo $(hostname -i)
echo $(hostname)
sudo -- sh -c "echo $(hostname -i) $(hostname) >> /etc/hosts"
cat /etc/hosts

sudo firewall-cmd --permanent --add-port=10250/tcp
sudo firewall-cmd --permanent --add-port=10251/tcp
sudo firewall-cmd --permanent --add-port=10255/tcp
sudo firewall-cmd --reload
```    

install containerd
```

sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo yum install -y yum-utils containerd.io && rm -I /etc/containerd/config.toml
sudo systemctl enable containerd && sudo systemctl start containerd
sudo systemctl status containerd
```

install kubeadmn
```

# <file>
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF

sudo yum install -y kubelet kubectl kubeadm
```


## Create Kubernetes Cluster

initialise control plane (master node)
```

sudo kubeadm init --pod-network-cidr 10.244.0.0/16 --apiserver-cert-extra-sans=EXTERNAL_IP
  # eg: sudo kubeadm init --pod-network-cidr 10.244.0.0/16 --apiserver-cert-extra-sans=172.166.192.249
  # eg: sudo kubeadm init --pod-network-cidr 10.244.0.0/16 --apiserver-cert-extra-sans=172.172.3.230

sudo systemctl enable kubelet && sudo systemctl status kubelet

```

export kube config
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
export KUBECONFIG=/etc/kubernetes/admin.conf
```

deploy pod network

```
# if current user:
curl https://raw.githubusercontent.com/projectcalico/calico/v3.25.1/manifests/calico.yaml -O
sudo kubectl apply -f calico.yaml
sudo kubectl get nodes

# else if root user
sudo kubectl get pods --all-namespaces
```

Join worker node(s)
```
kubeadm join 10.0.0.5:6443 --token wwjc51.p84bdemib58f1oij \
	--discovery-token-ca-cert-hash sha256:c29c1e58c15ddce711a286614bb672c5ad73fa0ba49f8e47c4661e070e8b40b8

# verify this with kubectl on master node
sudo kubectl get nodes
sudo kubectl get pods --all-namespaces

```

## Deploy Spark

install helm
```
wget https://get.helm.sh/helm-v3.14.1-linux-amd64.tar.gz
tar -zxvf helm-v3.14.1-linux-amd64.tar.gz
mv linux-amd64/helm /usr/local/bin/helm
```

install spark-on-k8s-operator via helm
```
/usr/local/bin/helm repo add spark-operator https://googlecloudplatform.github.io/spark-on-k8s-operator
/usr/local/bin/helm install my-release spark-operator/spark-operator --namespace spark-operator --create-namespace --set sparkJobNamespace=default
# /usr/local/bin/helm install my-release spark-operator/spark-operator --namespace spark-operator --create-namespace --set sparkJobNamespace=default --set image.tag=v1beta2-1.2.0-3.0.0
# /usr/local/bin/helm install my-release spark-operator/spark-operator --namespace spark-operator --create-namespace

# add service account
kubectl create serviceaccount spark
kubectl create clusterrolebinding spark-role --clusterrole=edit --serviceaccount=default:spark --namespace=default
# Note: make sure to add this spark configuration in manifest file: --conf spark.kubernetes.authenticate.driver.serviceAccountName=spark
```

## Run Spark Applications
```
vim SparkPiApplication.yaml
kubectl apply -f SparkPiApplication.yaml
```

## Troubleshooting
check spark applications status
```shell
kubectl get sparkapplications

# sample
NAME       STATUS   ATTEMPTS   START                  FINISH       AGE
spark-pi   FAILED              2024-02-15T23:43:55Z   <no value>   33s
```

check a failed spark application logs
```shell
kubectl get sparkapplications spark-pi -o=yaml

# sample log
# ...
# status:
#   applicationState:
#     errorMessage: "failed to run spark-submit for SparkApplication default/spark-pi:
#     ...
#     4/02/15 22:01:57 INFO KerberosConfDriverFeatureStep:  You have not specified a krb5.conf file locally or via a ConfigMap. Make sure that you have the krb5.conf locally on the driver image.
#     \nException in thread\"main\" io.fabric8.kubernetes.client.KubernetesClientException: Failure executing:
#       POST at: https://10.96.0.1/api/v1/namespaces/default/pods. Message: Pod \"spark-pi-driver\"
#       is invalid: spec.containers[0].resources.requests: Invalid value: \"1\": must be less than or equal to cpu limit of 100m. Received status: Status(apiVersion=v1,
#       code=422, details=StatusDetails(causes=[StatusCause(field=spec.containers[0].resources.requests,
#       message=Invalid value: \"1\": must be less than or equal to cpu limit of 100m,
#       reason=FieldValueInvalid, additionalProperties={})], group=null, kind=Pod, name=spark-pi-driver,
#       retryAfterSeconds=null, uid=null, additionalProperties={}), kind=Status, message=Pod
#       \"spark-pi-driver\" is invalid: spec.containers[0].resources.requests: Invalid
#       value: \"1\": must be less than or equal to cpu limit of 100m, metadata=ListMeta(_continue=null,
#       remainingItemCount=null, resourceVersion=null, selfLink=null, additionalProperties={}),
#       reason=Invalid, status=Failure, additionalProperties={}).\n
```

check pods created for spark app or not
```shell
kubectl get pods
```

check pod logs if in pending state
```shell
kubectl describe pod my-release-spark-operator-5778dd4cd9-n6tcz
kubectl describe pod spark-pi-driver 


# sample spark-operator pod
# Events:
#   Type     Reason     Age                     From               Message
#   ----     ------     ----                    ----               -------
#   Normal   Scheduled  10m                     default-scheduler  Successfully assigned spark-operator/my-release-spark-operator-5778dd4cd9-n6tcz to worker-node-1
#   Warning  Failed     8m39s (x6 over 9m58s)   kubelet            Error: ImagePullBackOff
#   Normal   Pulling    8m25s (x4 over 9m59s)   kubelet            Pulling image "ghcr.io/googlecloudplatform/spark-operator:v1beta2-1.2.0-3.0.0"
#   Warning  Failed     8m25s (x4 over 9m58s)   kubelet            Failed to pull image "ghcr.io/googlecloudplatform/spark-operator:v1beta2-1.2.0-3.0.0": rpc error: code = NotFound desc = failed to pull and unpack image "ghcr.io/googlecloudplatform/spark-operator:v1beta2-1.2.0-3.0.0": failed to resolve reference "ghcr.io/googlecloudplatform/spark-operator:v1beta2-1.2.0-3.0.0": ghcr.io/googlecloudplatform/spark-operator:v1beta2-1.2.0-3.0.0: not found
#   Warning  Failed     8m25s (x4 over 9m58s)   kubelet            Error: ErrImagePull
#   Normal   BackOff    4m58s (x21 over 9m58s)  kubelet            Back-off pulling image "ghcr.io/googlecloudplatform/spark-operator:v1beta2-1.2.0-3.0.0"

# sample driver pod 1 - wrong image
# Events:
#   Type     Reason       Age                   From               Message
#   ----     ------       ----                  ----               -------
#   Normal   Scheduled    3m28s                 default-scheduler  Successfully assigned default/spark-pi-driver to worker-node-1
#   Warning  FailedMount  3m28s                 kubelet            MountVolume.SetUp failed for volume "spark-conf-volume-driver" : configmap "spark-drv-4912ab8db15a74ca-conf-map" not found
#   Normal   BackOff      2m7s (x6 over 3m25s)  kubelet            Back-off pulling image "gcr.io/spark-operator/spark:v3.0.0"
#   Warning  Failed       2m7s (x6 over 3m25s)  kubelet            Error: ImagePullBackOff
#   Normal   Pulling      114s (x4 over 3m26s)  kubelet            Pulling image "gcr.io/spark-operator/spark:v3.0.0"
#   Warning  Failed       114s (x4 over 3m26s)  kubelet            Failed to pull image "gcr.io/spark-operator/spark:v3.0.0": rpc error: code = NotFound desc = failed to pull and unpack image "gcr.io/spark-operator/spark:v3.0.0": failed to resolve reference "gcr.io/spark-operator/spark:v3.0.0": gcr.io/spark-operator/spark:v3.0.0: not found
#   Warning  Failed       114s (x4 over 3m26s)  kubelet            Error: ErrImagePull

# sample driver pod 2 - affinity checks
#  Warning  FailedScheduling  24m (x35 over 3h14m)  default-scheduler  0/2 nodes are available: 1 node(s) didn't match Pod's node affinity/selector, 1 node(s) had untolerated taint {node-role.kubernetes.io/control-plane: }. preemption: 0/2 # nodes are available: 2 Preemption is not helpful for scheduling..
```
check pod logs 
```shell
kubectl logs spark-pi-driver

# sample driver pod 2
# E0216 12:08:15.842338      10 reflector.go:140] pkg/mod/k8s.io/client-go@v0.25.3/tools/cache/reflector.go:169: Failed to watch *v1beta2.ScheduledSparkApplication: failed to list *v1beta2.ScheduledSparkApplication: scheduledsparkapplications.sparkoperator.k8s.io is forbidden: User "system:serviceaccount:default:spark" cannot list resource "scheduledsparkapplications" in API group "sparkoperator.k8s.io" at the cluster scope
```

check node selector, taints etc
```shell
kubectl describe node worker-node
```


```
## use master node as worker node too

kubectl taint node master-node node-role.kubernetes.io/control-plane:NoSchedule-
## update node selector from spark application 
kubectl taint node master-node node-role.kubernetes.io/control-plane:NoSchedule-
```

# spark logs not available:
`Error from server: Get "https://10.0.0.4:10250/containerLogs/default/spark-pi-driver/spark-kubernetes-driver": dial tcp 10.0.0.4:10250: connect: no route to host`
```
sudo firewall-cmd --add-port=10250/tcp --permanent
sudo firewall-cmd --reload
sudo firewall-cmd --list-all  # you should see that port `10250` is updated
```

E0216 11:39:03.426045      10 reflector.go:140] pkg/mod/k8s.io/client-go@v0.25.3/tools/cache/reflector.go:169: Failed to watch *v1beta2.ScheduledSparkApplication: failed to list *v1beta2.ScheduledSparkApplication: scheduledsparkapplications.sparkoperator.k8s.io is forbidden: User "system:serviceaccount:default:spark" cannot list resource "scheduledsparkapplications" in API group "sparkoperator.k8s.io" at the cluster scope

