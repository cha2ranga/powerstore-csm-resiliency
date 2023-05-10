This README will guide you on how to set up an RKE2 Kubernetes cluster and connect it with Dell PowerStore CSI and CSM resiliency for persistent storage.

RKE2 Installation [@RKE2 Installation Guide](https://ranchermanager.docs.rancher.com/how-to-guides/new-user-guides/kubernetes-cluster-setup/rke2-for-rancher) 



## Activity flow,
![Diagram](https://github.com/cha2ranga/powerstore-csm-resiliency/blob/main/images/diagram1.jpg)


## Create Rocky 9 VMS and upgrade the packages. 
VM packages and upgrade guide  [@K8s VMs setup](https://github.com/cha2ranga/k8s-installation) 

Install Rocky Linux 9 vms and update the packages. In this CSI-Drivers installation, we will use iSCSI and NFS options. Make sure to install iscsi and multipath packages as well. 

## Create RKE2 Cluster
Touch follwoing file on both master and workder nodes
touch /etc/rancher/rke2/config.yaml

This setup I'm using single master and three worker nodes

**Package installation**

When I installed the cluster with version 1.25 or 1.26, the helm repo produced an error due to the pod security policy. Even though I tried to disable it during the helm upgrade, there was no luck.

To avoid this issue, I installed the cluster with 1.24 and then set up Rancher UI. Later upgrade to 1.26.0 packages.


This way, you can install to specific version. 
*curl -sfL https://get.rke2.io | INSTALL_RKE2_VERSION=v1.25.9+rke2r1 sh -*


To avoid encountering a sha1 issue, it is recommended to utilize the curl command instead of adding the yum repo file.

for the servers, this will install latest stable release 
```bash
curl -sfL https://get.rke2.io | sh -
```

For the workers, you can first download the script. Then set the environment variable as "agent."  
(anther way  
```bash
curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE="agent" sh -
```

```bash
curl -sfL https://get.rke2.io --output install.sh
chmod +x install.sh
export INSTALL_RKE2_TYPE=agent
 ./install.sh
```

INSTALL_RKE2_TYPE
Type of rke2 service. Can be either "server" or "agent".
Default is "server".


**Enable services for server nodes**


```bash
systemctl enable rke2-server
systemctl start rke2-server
journalctl -u rke2-server -f
```
Package will be installed under /var/lib/rancher/rke2


Get the token from following file location.

```bash
[root@rke2-m1 rke2]# cat /var/lib/rancher/rke2/server/token
K10c3fbaeea3194c08cfbd9f38bee30de444d43103fe8exxxxxxxxxxxxxxxxx::server:my-shared-secret
```

Then update the /etc/rancher/rke2/config.yaml token.


Let's export the paths. 

```bash
export CONTAINER_RUNTIME_ENDPOINT=unix:///run/k3s/containerd/containerd.sock
export CONTAINERD_ADDRESS=/run/k3s/containerd/containerd.sock
export PATH=/var/lib/rancher/rke2/bin:$PATH
export KUBECONFIG=/etc/rancher/rke2/rke2.yaml
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc
alias k=kubectl
complete -o default -F __start_kubectl k
```
```bash
[root@rke2-m1 ~]# kubectl get nodes
NAME      STATUS   ROLES                       AGE     VERSION
rke2-m1   Ready    control-plane,etcd,master   9m31s   v1.24.12+rke2r1
```

Following file contains cluster details. you can rename it as kubeconfig and use it. you can even copy it to .kube/config directoy

/etc/rancher/rke2/rke2.yaml


Let's start services for worker nodes

```bash
systemctl enable rke2-agent.service
```


Now you can add the worker nodes
```bash
mkdir -p /etc/rancher/rke2/
vi /etc/rancher/rke2/config.yaml
```

config.yaml
server: https://rke2-m1.xxxxx.xxxx.csc:9345
token: K10fee940b0712a2f59f74d5f62db4393c6a3e5e19xxxxxxxxxxxxxxxxx::server:1ee80a11xxxxxxxxxxxxx54f32d

```bash
systemctl enable rke2-agent.service
systemctl start rke2-agent.services
journalctl -u rke2-agent.service -f 
```


!!! uninstall script !!! 
In case if you want to perform a clean up, you can find the files in follwoing location

```bash
/usr/bin/rke2-uninstall.sh
```

## Upgrade RKE2

upgrade to latest version. 
```bash
 curl -sfL https://get.rke2.io | INSTALL_RKE2_CHANNEL=latest sh - && systemctl restart rke2-agent
```

Upgrade to specific version
```bash
https://github.com/rancher/rke2/releases/tag/v1.25.9%2Brke2r1
```

Master node upgrade

```bash
curl -sfL https://get.rke2.io | INSTALL_RKE2_VERSION=v1.25.9+rke2r1 sh -
systemctl restart rke2-server && journalctl -u rke2-server -f
```

Workder Node upgrade
```bash
curl -sfL https://get.rke2.io | INSTALL_RKE2_VERSION=v1.25.9+rke2r1 sh -
systemctl restart rke2-agent &&  journalctl -u rke2-agent.service -f
```



## Install Rancher UI

You can download the values file for rancher helm. Then edit that one as well. In this case, at least you will know your editing settings.


![Helm Values](https://github.com/cha2ranga/powerstore-csm-resiliency/blob/main/images/helm_values1.jpg)


Other option is you can pass the values during during helm install

```bash
[root@rke2-m1 ~]# helm install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --set hostname=rancher.xxxxx.xxxx \
  --set replicas=1 \
  --set bootstrapPassword=<YOURPASSWORD>
NAME: rancher
LAST DEPLOYED: Tue Apr 25 12:38:36 2023
NAMESPACE: cattle-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Rancher Server has been installed.

NOTE: Rancher may take several minutes to fully initialize. Please standby while Certificates are being issued, Containers are started and the Ingress rule comes up.

Check out our docs at https://rancher.com/docs/

If you provided your own bootstrap password during installation, browse to https://rancher.xxxxx.xxxx to get started.


If this is the first time you installed Rancher, get started by running this command and clicking the URL it generates:
echo https://rancher.xxxxx.xxxx/dashboard/?setup=$(kubectl get secret --namespace cattle-system bootstrap-secret -o go-template='{{.data.bootstrapPassword|base64decode}}')


To get just the bootstrap password on its own, run:
kubectl get secret --namespace cattle-system bootstrap-secret -o go-template='{{.data.bootstrapPassword|base64decode}}{{ "\n" }}'



Happy Containering!

```



Here are the newly added pods,
![Rancher UI Pods](https://github.com/cha2ranga/powerstore-csm-resiliency/blob/main/images/rancher_ui_pods.jpg)


During this installtion ingress service will be configured. In my case, I'm using master node to host Rancher UI. you can add "A" record in DNS to access Rancher UI. 

![Rancher login UI](https://github.com/cha2ranga/powerstore-csm-resiliency/blob/main/images/rancher_dashboard.jpg)

Cluster Details

![Rancher dashboard](https://github.com/cha2ranga/powerstore-csm-resiliency/blob/main/images/cluster_details.jpg)


## Install PowerStore CSI Drives - Used helm installation method.


[@Dell PowerStore CSI - Installation guide Documentation](https://dell.github.io/csm-docs/docs/csidriver/installation/helm/powerstore/#prerequisites) 


volumesnapshot CRD is avaialble with the default RKE2 installation.

![VolumeSnapshot CRD](https://github.com/cha2ranga/powerstore-csm-resiliency/blob/main/images/vs_crd.jpg)


PowerStore Details

```bash
Management IP Address: 172.24.xxx.xxx


GlobalID: PSxx2c5cbxxxx4


user: rancher


password: xxxxxxxxxxxx


iSCSI Discovery IP: 172.24.xxx.xxx


NAS Server Name: csi-nas


NAS Server IP: 172.24.xxx.xxx
```


**01. Clone the powerstore CSI drives repo**

```bash
git clone -b v2.6.0 https://github.com/dell/csi-powerstore.git
```

**02. Edit the secret file**

sample values in /csi-powerstore/samples/secret/secret.yaml 

(refer secret.yaml)

**03. create namespace for csi-powerstore**

```bash
k create ns csi-powerstore
```

**04. create secret and verify**

```bash
[root@rke2-m1 secret]# pwd
/root/csi-powerstore/samples/secret
[root@rke2-m1 secret]#
[root@rke2-m1 secret]# kubectl create secret generic powerstore-config -n csi-powerstore --from-file=config=secret.yaml
secret/powerstore-config created
[root@rke2-m1 secret]#


[root@rke2-m1 secret]# k -n csi-powerstore get secrets
NAME                TYPE     DATA   AGE
powerstore-config   Opaque   1      79s
[root@rke2-m1 secret]#
```

**05. copy the sameple values for CSI-PowerStore**

```bash
[root@rke2-m1 csi-powerstore]# pwd
/root/csi-powerstore
[root@rke2-m1 csi-powerstore]#
[root@rke2-m1 csi-powerstore]# cd dell-csi-helm-installer && cp ../helm/csi-powerstore/values.yaml ./my-powerstore-settings.yaml
[root@rke2-m1 dell-csi-helm-installer]#
```

**06. copy sameple powerstore csi helm values and edit**

refer my-powerstore-settings.yaml

When it comes to PowerStore CSI resiliency, enable following podmon parameters,

![podmon](https://github.com/cha2ranga/powerstore-csm-resiliency/blob/main/images/podmon.jpg)

**07. Install the csi drives**

```bash
[root@rke2-m1 dell-csi-helm-installer]# ./csi-install.sh --namespace csi-powerstore --values ./my-powerstore-settings.yaml
```

Since we are installing only iSCSI and NFS, you can ignore the NVMe warnings. 

**Installation output**

![CSI Drivers Installation Output](https://github.com/cha2ranga/powerstore-csm-resiliency/blob/main/images/csi_installation_output2.jpg)


Status of the CSI drives pods,

![Drive Status](https://github.com/cha2ranga/powerstore-csm-resiliency/blob/main/images/csi_driver_pods.jpg)


**08. create storageClass**


there are sample storageclass available under */csi-powerstore/samples/storageclass* directory

![sc](https://github.com/cha2ranga/powerstore-csm-resiliency/blob/main/images/sc1.jpg)

Once you edit the storage class yaml file, use 'kubectl apply -f <sc_name.yaml> to create storageclass

![storageclass](https://github.com/cha2ranga/powerstore-csm-resiliency/blob/main/images/sc2.jpg)


## Resiliency Test

In order to enable resiliency you must add following label

```bash
podmon.dellemc.com/driver: csi-powerstore
```

Let's create two statefulsets with PowerStore pvc. One with the "podmon.dellemc.com/driver: csi-powerstore" lable

**statefulset_resiliency.yaml**  >>>>> *resiliency enabled*

[@statefulset_resiliency.yaml](https://raw.githubusercontent.com/cha2ranga/powerstore-csm-resiliency/main/statefulset_resiliency.yaml) 


**statefulset.yaml** >>>>> *no resiliency configured*

[@statefulset.yaml](https://raw.githubusercontent.com/cha2ranga/powerstore-csm-resiliency/main/statefulset.yaml) 


You can apply both yaml file to create statefulset

Status of the staefulsets,

![workloads](https://github.com/cha2ranga/powerstore-csm-resiliency/blob/main/images/sfs1.jpg)

PVC Status

![pvc](https://github.com/cha2ranga/powerstore-csm-resiliency/blob/main/images/sfs2.jpg)


Let's verify the statefulset configured with resiliency at the PowerStore side,
As you can see below, PVC is attached to rke2-w1

![PowerStore](https://github.com/cha2ranga/powerstore-csm-resiliency/blob/main/images/sfs3.jpg)

Now let's shutdown the "rke2-w1" node. 

![Node Down](https://github.com/cha2ranga/powerstore-csm-resiliency/blob/main/images/sfs4.jpg)


The pod will automatically recreate within a few minutes (less than 5 minutes). You can observe that the recreated pod is now on the "rke-w2" node. 
Without this resiliency option, the pod will remain in a "Terminating" state until the worker node is operational again, which typically requires manual cleaning up. 

![PowerStore](https://github.com/cha2ranga/powerstore-csm-resiliency/blob/main/images/sfs5.jpg)

From the PowerStore UI, you will see volume attached to the "rke-w2" node. 

![PowerStore](https://github.com/cha2ranga/powerstore-csm-resiliency/blob/main/images/sfs6.jpg)

You can verify futher in PowerStore events

![PowerStore evnts](https://github.com/cha2ranga/powerstore-csm-resiliency/blob/main/images/sfs7.jpg)



