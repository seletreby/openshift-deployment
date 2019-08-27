# OpenShift 3.11 Deployment
## Environment Prerequisite preparation
I worked on preparing small development Openshift environment based on OpenShift 3.11. The environment will be hosted on Softlayer. 
I created three VMs with the following specs:

 Hostname | IP | Function | CPU | Memory | Storage 
 ---|:---|:---|:---|:---|:---|
master01.example.com | 158.177.112.229 | Master Node | 16 | 32 | 100 | 
node01.example.com | 158.177.112.23 | Worker Node | 16 | 32 | 100 |
infra.example.com | 158.177.112.227 | Infra node | 16 | 32 | 100 |
node02.example.com | 158.177.112.23 | DNS Node | 16 | 32 | 100 |

### Register the VMs to Redhat Subscription Manager
Unfortunately Softlayer is not registered to the subscription manager that have OpenShift package. So I need to unregister for its subscription manager and use mine to attach an active OpenShift Container Platform subscription. I did that using following steps:
* Unregister from Softlayer Satellite using the following commands:
```
subscription-manager remove --all
subscription-manager unregister
```
* Subscribe using your credential and redhat default satellite
```
subscription-manager register --username salah_eletreby --serverurl=subscription.rhsm.redhat.com:443/subscription
```
* Copy original redhat configuration (and backup Softlayer)
```
cp /etc/rhsm/rhsm.conf /etc/rhsm/rhsm.conf.sl.bkp

cp /etc/rhsm/rhsm.conf.kat-backup /etc/rhsm/rhsm.conf
```
* Pull the latest subscription data
```
subscription-manager refresh
```
* List all available subscription
```
subscription-manager list --available --matches '*OpenShift*'
```
-	In the output for the previous command, find the pool ID for an OpenShift Container Platform subscription and attach it. As the below
```
subscription-manager attach --pool=8a85f9936a6f0b99016a6f3effc90f09
```
-	Disable all yum repositories
  - Disable all the enabled RHSM repositories:
  ```
  subscription-manager repos --disable="*"
  ```
   -	List the remaining yum repositories and note their names under repo id, if any:
   ```
   yum repolist
   ```
   -	Use yum-config-manager to disable the remaining yum repositories:
   ```
   yum-config-manager --disable \*
   ```
 -	Enable only the repositories required by OpenShift Container Platform 3.11
 ```
 subscription-manager repos \
    --enable="rhel-7-server-rpms" \
    --enable="rhel-7-server-extras-rpms" \
    --enable="rhel-7-server-ose-3.11-rpms" \
    --enable="rhel-7-server-ansible-2.6-rpms"
```
### Install base packages
-	On all the nodes install the following packages as root user
```
yum install wget git net-tools bind-utils yum-utils iptables-services bridge-utils bash-completion kexec-tools sos psacct -y 
```
-	Install ansible
```
yum install -y openshift-ansible
yum install atomic-openshift-node-3.11* -y
```
### Install and configure Docker
#### Using overlay2 driver
I added 100 GB raw disks to master01, node01 and infra for the docker. You can use "lsblk" command to list all the available disks. To achieve this, execute the following commands on all my nodes using root user:
```
mkfs.xfs -n ftype=1 /dev/xvdc
mkdir /var/lib/docker
mount -t xfs /dev/xvdc /var/lib/docker
cp /etc/fstab /etc/fstab.orig
echo "/dev/xvdc   /var/lib/docker                       xfs     defaults        0 0" >>/etc/fsta
init 6
```
**Install Docker package on all nodes:**
```
yum install docker-1.13.1 -y
```
Configure docker on all nodes using following steps:
```
cp /usr/lib/systemd/system/docker.service /usr/lib/systemd/system/docker.service.orig
sed -i -e 's/--storage-driver=devicemapper/--storage-driver=overlay2/' /usr/lib/systemd/system/docker.service
systemctl daemon-reload
systemctl restart docker
systemctl enable docker
systemctl start docker
```
-	Verify using the following:
``` docker info ```
