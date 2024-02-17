# Spark on Kubernetes Setup

This is a guide to create kubernetes cluster, install spark operator, run and monitor spark applications.

## Table of Contents  
- [Spark on Kubernetes](#spark-on-kubernetes-gopala) 
  - [Preparing VMs](#1-prepare-vms)  
  - [Creating Kubernetes Cluster](#2-create-kubernetes-cluster)  
  - [Installing Spark](#3-deploy-spark)  
  - [Running Spark Applications](#4-run-spark-applications)  
  - [Monitoring Spark Applications](#5-monitoring-spark-applications)   
  - [Troubleshooting](#6-troubleshooting) 

## 1. Prepare VMs
- Create 3 VMs - 1 for control plane and 2 for worker nodes
- Add Inbound port rules as below
  - Master node: 8080,6443,10250,10259,10257,2379-2380
  - Worker node: 10250,30000-32767
- Log in to each VM and add the below settings:
   
Udpate settings
```shell
# disable swap:

cat /etc/fstab
sudo sed -i '/swap/d' /etc/fstab
sudo swapoff -a
```

```shell
# disable SELinux

sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

```shell
# update iptables settings

sudo cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

cat /etc/modules-load.d/k8s.conf
sudo modprobe overlay
sudo modprobe br_netfilter

sudo cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# reload and verify settings
sudo sysctl --system
lsmod | grep br_netfilter
lsmod | grep overlay
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
```

```shell
# update hostname

sudo hostnamectl set-hostname <master-node | worker-node-1 | worker-node2 | ...>
echo $(hostname -i)
echo $(hostname)
sudo -- sh -c "echo $(hostname -i) $(hostname) >> /etc/hosts"
cat /etc/hosts
```

```shell
# enable firewall

systemctl status firewalld
systemctl start firewalld
```

```shell
# update firewall
# *** master node ONLY ***

sudo firewall-cmd --permanent --add-port=6443/tcp
sudo firewall-cmd --permanent --add-port=2379-2380/tcp
sudo firewall-cmd --permanent --add-port=10250/tcp
sudo firewall-cmd --permanent --add-port=10251/tcp
sudo firewall-cmd --permanent --add-port=10252/tcp
sudo firewall-cmd --permanent --add-port=10255/tcp
sudo firewall-cmd --permanent --add-port=10257/tcp
sudo firewall-cmd --permanent --add-port=10259/tcp
sudo firewall-cmd --reload
```

```shell
#  update firewall
# *** worker node ONLY ***

sudo firewall-cmd --permanent --add-port=10250/tcp
sudo firewall-cmd --permanent --add-port=30000-32767/tcp
sudo firewall-cmd --permanent --add-port=10251/tcp
sudo firewall-cmd --permanent --add-port=10255/tcp
sudo firewall-cmd --reload
```    

Install containerd
```shell

sudo yum install yum-utils
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo yum install -y yum-utils containerd.io && rm -I /etc/containerd/config.toml
sudo systemctl enable containerd && sudo systemctl start containerd
sudo systemctl status containerd
```

Install kubeadmn
```shell

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


## 2. Create Kubernetes Cluster

Initialise control plane (master node)
```shell
# *** master node ONLY ***

sudo kubeadm init --pod-network-cidr 10.244.0.0/16 --apiserver-cert-extra-sans=<EXTERNAL_IP>
sudo systemctl enable kubelet && sudo systemctl status kubelet

# Note: 1. replace EXTERNAL_IP with public IP. This will let us use kubectl from a local machine and access Spark UI
#	2. copy the token from the above command to use it to add worker node to this cluster
```

Export kube config
```shell
# if current user:
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# else if root user
export KUBECONFIG=/etc/kubernetes/admin.conf

# verify connectivity
sudo kubectl get pods --all-namespaces
```

Deploy pod network
```shell
curl https://raw.githubusercontent.com/projectcalico/calico/v3.25.1/manifests/calico.yaml -O
sudo kubectl apply -f calico.yaml
```

Join worker node(s)
```shell
kubeadm join 10.0.0.5:6443 --token wwjc51.p84bdemib58f1oij \
	--discovery-token-ca-cert-hash sha256:c29c1e58c15ddce711a286614bb672c5ad73fa0ba49f8e47c4661e070e8b40b8

# verify this with kubectl on master node
sudo kubectl get nodes
sudo kubectl get pods --all-namespaces
```

## 3. Deploy Spark

Install helm
```shell
wget https://get.helm.sh/helm-v3.14.1-linux-amd64.tar.gz
# curl -O https://get.helm.sh/helm-v3.14.1-linux-amd64.tar.gz
tar -zxvf helm-v3.14.1-linux-amd64.tar.gz
mv linux-amd64/helm /usr/local/bin/helm
```

Install spark-on-k8s-operator via helm
```shell

# create service account
vim spark-operator-rbac.yaml
kubectl apply -f spark-operator-rbac.yaml

/usr/local/bin/helm repo add spark-operator https://googlecloudplatform.github.io/spark-on-k8s-operator
/usr/local/bin/helm upgrade -i my-release spark-operator/spark-operator --set serviceAccounts.spark.create=false --set serviceAccounts.sparkoperator.create=false -v=10
```

## 4. Run Spark Applications

Create a sample Spark Application
```shell
vim SparkPiApplication.yaml
kubectl apply -f SparkPiApplication.yaml
```

## 5. Monitoring Spark Applications

Access Spark UI from local machine
```shell
kubectl port-forward <driver-pod-name> 4040:4040
You can then open up the Spark UI at http://localhost:4040/

# Note: for this step install kubectl on your local machine, copy ~/.kube/config file and update the server url with pubilc IP 
```

Access Driver logs:
```shell
kubectl get pods
kubectl logs <driver-pod-name>
```

Install fuent-bit to export Kubernetes and Spark logs
```
TODO:
```

## 6. Troubleshooting

check spark applications status
```shell
kubectl get sparkapplications

# sample
NAME       STATUS   ATTEMPTS   START                  FINISH       AGE
spark-pi   FAILED              2024-02-15T23:43:55Z   <no value>   33s
```

check a failed spark application state
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

check operator pod logs
```log4j
kubectl describe pod my-release-spark-operator-5778dd4cd9-n6tcz

# sample 1 - missing image
# Events:
#   Type     Reason     Age                     From               Message
#   ----     ------     ----                    ----               -------
#   Normal   Scheduled  10m                     default-scheduler  Successfully assigned spark-operator/my-release-spark-operator-5778dd4cd9-n6tcz to worker-node-1
#   Warning  Failed     8m39s (x6 over 9m58s)   kubelet            Error: ImagePullBackOff
#   Normal   Pulling    8m25s (x4 over 9m59s)   kubelet            Pulling image "ghcr.io/googlecloudplatform/spark-operator:v1beta2-1.2.0-3.0.0"
#   Warning  Failed     8m25s (x4 over 9m58s)   kubelet            Failed to pull image "ghcr.io/googlecloudplatform/spark-operator:v1beta2-1.2.0-3.0.0": rpc error: code = NotFound desc = failed to pull and unpack image "ghcr.io/googlecloudplatform/spark-operator:v1beta2-1.2.0-3.0.0": failed to resolve reference "ghcr.io/googlecloudplatform/spark-operator:v1beta2-1.2.0-3.0.0": ghcr.io/googlecloudplatform/spark-operator:v1beta2-1.2.0-3.0.0: not found
#   Warning  Failed     8m25s (x4 over 9m58s)   kubelet            Error: ErrImagePull
#   Normal   BackOff    4m58s (x21 over 9m58s)  kubelet            Back-off pulling image "ghcr.io/googlecloudplatform/spark-operator:v1beta2-1.2.0-3.0.0"
```

```
# sample 2 - missing serviceaccount
# Exception in thread "main" io.fabric8.kubernetes.client.KubernetesClientException: Failure executing: POST at: https://10.96.0.1/api/v1/namespaces/default/pods. Message: Forbidden!Configured service account doesn't have access. Service account may have been revoked. pods "spark-pi-driver" is forbidden: error looking up service account default/spark: serviceaccount "spark" not found.
```

check driver pod logs
```
# sample 1 - wrong image

kubectl describe pod spark-pi-driver
# ...
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
```

```
# sample d2 - affinity checks

kubectl describe pod spark-pi-driver
#  Warning  FailedScheduling  24m (x35 over 3h14m)  default-scheduler  0/2 nodes are available: 1 node(s) didn't match Pod's node affinity/selector, 1 node(s) had untolerated taint {node-role.kubernetes.io/control-plane: }. preemption: 0/2 # nodes are available: 2 Preemption is not helpful for scheduling..
```

spark pod logs - not available
```shell
Error from server: Get "https://10.0.0.4:10250/containerLogs/default/spark-pi-driver/spark-kubernetes-driver": dial tcp 10.0.0.4:10250: connect: no route to host
sudo firewall-cmd --list-all  # you should see that port `10250` is updated

# port is not in firewall settings
sudo firewall-cmd --add-port=10250/tcp --permanent
sudo firewall-cmd --reload
sudo firewall-cmd --list-all  # you should see that port `10250` is updated
```

check pod logs - no executor
```shell
kubectl logs spark-pi-driver

# sample driver pod 2
# E0216 12:08:15.842338      10 reflector.go:140] pkg/mod/k8s.io/client-go@v0.25.3/tools/cache/reflector.go:169: Failed to watch *v1beta2.ScheduledSparkApplication: failed to list *v1beta2.ScheduledSparkApplication: scheduledsparkapplications.sparkoperator.k8s.io is forbidden: User "system:serviceaccount:default:spark" cannot list resource "scheduledsparkapplications" in API group "sparkoperator.k8s.io" at the cluster scope
```

check node selector, taints etc
```shell
kubectl describe node worker-node
```

use master node as worker node
```
kubectl taint node master-node node-role.kubernetes.io/control-plane:NoSchedule-
```


