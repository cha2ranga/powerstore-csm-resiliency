This README document provides instructions on setting up a RKE2 kubernetes cluster and integrate with Dell PowerStore CSI and CSM resiliency for persistent storage. 

RKE2 Installation [@RKE2 Installation Guide](https://ranchermanager.docs.rancher.com/how-to-guides/new-user-guides/kubernetes-cluster-setup/rke2-for-rancher) 



## RKE2 Installation

Activity flow,
![Diagram](https://github.com/cha2ranga/powerstore-csm-resiliency/blob/main/images/diagram1.jpg)


Install Rocky linux 9 vms and update the packages. In this CSI-Drivers installation, we are going to use iSCSI and NFS options. make sure to install iscsi and multipath packages as well. 

Touch follwoing file on both master and workder nodes
touch /etc/rancher/rke2/config.yaml


## Package installation
if you add yum repo file you will end up with sha1 issue. easy way is use curl commad

for the servers, this will install latest stable release 
```bash
curl -sfL https://get.rke2.io | sh -
```

for the workers you can first download the script. then set the environment variable as "agent"  
(anther way  curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE="agent" sh -

```bash
curl -sfL https://get.rke2.io --output install.sh
chmod +x install.sh
export INSTALL_RKE2_TYPE=agent
 ./install.sh
```

INSTALL_RKE2_TYPE
Type of rke2 service. Can be either "server" or "agent".
Default is "server".


## enable services for server nodes
```bash
systemctl enable rke2-server
systemctl start rke2-server
journalctl -u rke2-server -f
```
package will be install under /var/lib/rancher/rke2

## get the token from following file location. (all token are same)

```bash
[root@rke2-m1 rke2]# cat /var/lib/rancher/rke2/server/token
K10c3fbaeea3194c08cfbd9f38bee30de444d43103fe8exxxxxxxxxxxxxxxxx::server:my-shared-secret
```

Then update the you can get the token and update the /etc/rancher/rke2/config.yaml token.


## now export the paths. 
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

[root@rke2-m1 ~]# kubectl get nodes
NAME      STATUS   ROLES                       AGE     VERSION
rke2-m1   Ready    control-plane,etcd,master   9m31s   v1.24.12+rke2r1

## this file contains cluster details. you can rename it as kubeconfig and use it. you can even copy it to .kube/config directoy

/etc/rancher/rke2/rke2.yaml


## start services for worker nodes
```bash
systemctl enable rke2-agent.service
```


## now you can add the worker nodes
```bash
mkdir -p /etc/rancher/rke2/
vi /etc/rancher/rke2/config.yaml
```

config.yaml
server: https://rke2-m1.devops.sg.csc:9345
token: K10fee940b0712a2f59f74d5f62db4393c6a3e5e19xxxxxxxxxxxxxxxxx::server:1ee80a11xxxxxxxxxxxxx54f32d

```bash
systemctl enable rke2-agent.service
systemctl start rke2-agent.services
journalctl -u rke2-agent.service -f 
```


## uninstall script is in /usr/bin/rke2-uninstall.sh (Incase if you want to clean up and reinstall the packages)

```bash
curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE="agent" sh -

systemctl enable rke2-agent.service

systemctl restart rke2-agent.service
```


## Upgrade RKE2

upgrade to latest version. 
```bash
 curl -sfL https://get.rke2.io | INSTALL_RKE2_CHANNEL=latest sh - && systemctl restart rke2-agent
```
## upgrade to specific version
```bash
https://github.com/rancher/rke2/releases/tag/v1.25.9%2Brke2r1
```
## Master node upgrade
```bash
curl -sfL https://get.rke2.io | INSTALL_RKE2_VERSION=v1.25.9+rke2r1 sh -
systemctl restart rke2-server && journalctl -u rke2-server -f
```
## Workder Node upgrade
```bash
curl -sfL https://get.rke2.io | INSTALL_RKE2_VERSION=v1.25.9+rke2r1 sh -
systemctl restart rke2-agent &&  journalctl -u rke2-agent.service -f
```



## Install Rancher UI

When I installed cluster with 1.25 or 1.26, helm repo produced an error due to pod security policy. Even though I tried to disable it during helm upgrade, there was no luck.

Then install the cluster with 1.24 and then setup Rancher UI. Later upgrade to 1.26.0 packages.
 
You can download the values file for rancher helm. Then edit that one as well. In this case, at least you will know your editing settings.



![Helm Values](https://github.com/cha2ranga/powerstore-csm-resiliency/blob/main/images/helm_values1.jpg)


Other option is you can pass the values during during helm install

```bash
[root@rke2-m1 ~]# helm install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --set hostname=rancher.devops.csc \
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

If you provided your own bootstrap password during installation, browse to https://rancher.devops.csc to get started.


If this is the first time you installed Rancher, get started by running this command and clicking the URL it generates:
echo https://rancher.devops.csc/dashboard/?setup=$(kubectl get secret --namespace cattle-system bootstrap-secret -o go-template='{{.data.bootstrapPassword|base64decode}}')


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
## Install PowerStore CSI Drives - Use helm installation method.


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


01. Clone the powerstore CSI drives repo

```bash
git clone -b v2.6.0 https://github.com/dell/csi-powerstore.git
```

02. Edit the secret file (sample values in /csi-powerstore/samples/secret/secret.yaml) (refer secret.yaml)

03. create namespace for csi-powerstore
```bash
k create ns csi-powerstore
```

04. create secret and verify

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

05. copy the sameple values for CSI-PowerStore
```bash
[root@rke2-m1 csi-powerstore]# pwd
/root/csi-powerstore
[root@rke2-m1 csi-powerstore]#
[root@rke2-m1 csi-powerstore]# cd dell-csi-helm-installer && cp ../helm/csi-powerstore/values.yaml ./my-powerstore-settings.yaml
[root@rke2-m1 dell-csi-helm-installer]#
```

06. copy sameple powerstore csi helm values and edit. (refer my-powerstore-settings.yaml)

When it comes to PowerStore CSI resiliency, enable following podmon parameters,

![podmon](https://github.com/cha2ranga/powerstore-csm-resiliency/blob/main/images/podmon.jpg)

07. Install the csi drives
```bash
[root@rke2-m1 dell-csi-helm-installer]# ./csi-install.sh --namespace csi-powerstore --values ./my-powerstore-settings.yaml
```

Since we are installing only iSCSI and NFS, you can ignore the NVMe warnings. 
Installation output. 

![CSI Drivers Installation Output](https://github.com/cha2ranga/powerstore-csm-resiliency/blob/main/images/csi_installation_output.jpg)


Status of the CSI drives pods,

![Drive Status](https://github.com/cha2ranga/powerstore-csm-resiliency/blob/main/images/csi_driver_pods.jpg)


08. create storageclass 

there are sample storageclass available under /csi-powerstore/samples/storageclass directory

![sc](https://github.com/cha2ranga/powerstore-csm-resiliency/blob/main/images/sc1.jpg)

Once you edit the storage class yaml file, use 'kubectl apply -f <sc_name.yaml> to create storageclass

![storageclass](https://github.com/cha2ranga/powerstore-csm-resiliency/blob/main/images/sc2.jpg)


## Resiliency Test

In order to enable resiliency you must add following labes
```bash
podmon.dellemc.com/driver: csi-powerstore
```

Let's crete two statefulsets with powerstore pvc. One with the "podmon.dellemc.com/driver: csi-powerstore" lable

statefulset_resiliency.yaml  >>>>> resiliency enabled. 


statefulset.yaml >>>>> no resiliency configured

You can apply both yaml file to create statefulset

Statis of the staefulsets,

![workloads](https://github.com/cha2ranga/powerstore-csm-resiliency/blob/main/images/sfs1.jpg)

PVC Status

![pvc](https://github.com/cha2ranga/powerstore-csm-resiliency/blob/main/images/sfs2.jpg)


Let's verify the statefulset configured with resiliency at PowerStore side,
As you can see below, PVC is attached to rke2-w1

![PowerStore](https://github.com/cha2ranga/powerstore-csm-resiliency/blob/main/images/sfs3.jpg)

Now let's shutdown the "rke2-w1" node. 

![Node Down](https://github.com/cha2ranga/powerstore-csm-resiliency/blob/main/images/sfs4.jpg)


After few min (lessthan 5min) pod will recreate automatically. As you can see, pod now recreated on "rke-w2" node

![PowerStore](https://github.com/cha2ranga/powerstore-csm-resiliency/blob/main/images/sfs5.jpg)

From the PowerStore UI, now you will see volume is attached to "rke-w2" node. 

![PowerStore](https://github.com/cha2ranga/powerstore-csm-resiliency/blob/main/images/sfs6.jpg)

You can verify futher in PowerStore evnts

![PowerStore evnts](https://github.com/cha2ranga/powerstore-csm-resiliency/blob/main/images/sfs7.jpg)



