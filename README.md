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
### Configure host file
Add the IP address and hostname to host file of each machine on /etc/hosts
```
#vi /etc/hosts
127.0.0.1 localhost.localdomain localhost
158.177.112.229 master01.example.com master01
158.177.112.232 node02.example.com node02
158.177.112.238 node01.example.com node01
158.177.112.227 infra.example.com infra
```
or use the following:
````
cat << EOF > /etc/hosts
127.0.0.1 localhost.localdomain localhost
158.177.112.229 master01.example.com master01
158.177.112.232 node02.example.com node02
158.177.112.238 node01.example.com node01
158.177.112.227 infra.example.com infra
EOF
````
### Install and configure DNS
I did DNS installation on my node02.example.com. I used dnsmasq following this steps:
-	Use yum to install dnsmasq and bind-utils
```
yum install dnsmasq bind-utils -y
```
-	Add entries for planned OpenShift hostname and OpenShift wildcard route and path for resolv.con to /etc/dnsmasq.conf
```
# vi /etc/dnsmasq.conf
### Add the following
# somewhere other that /etc/resolv.conf
resolv-file=/etc/resolv.dnsmasq
address=/master01.example.com/158.177.112.229
address=/node01.example.com/158.177.112.238
address=/infra.example.com/158.177.112.227
address=/app.example.com/158.177.112.227
address=/node02.example.com/158.177.112.232
```
-	create resolv.dbsmasq file using the following
```
cp /etc/resolv.conf /etc/resolv.dnsmasq
```
-	Enable to start service automatic using the following command
```
systemctl enable --now dnsmasq
```
-	On each of Redhat openshift node add to /etc/resolv.conf the nameserver on node02.example.com As following:
```
[root@master01 ~]# cat /etc/resolv.conf
nameserver 158.177.112.232
nameserver 10.0.80.11
nameserver 10.0.80.12
```
OR
```
cat <<EOF> /etc/resolv.conf
nameserver 158.177.112.232
nameserver 10.0.80.11
nameserver 10.0.80.12
EOF
```
### Configure host access
The OpenShift Container Platform installer requires a user that has access to all hosts.
#### Configure Password less
Execute the following to enable ssh password-less
-	Generate ssh certificate on the master node using the following command:
```
ssh-keygen -t rsa
```
The public key will be written to “~/.ssh/id_rsa.pub”
-	Add the generated public key to the “~/.ssh/authorized_keys” of all the other nodes:
```
ssh-copy-id root@node01.example.com
ssh-copy-id root@infra.example.com
ssh-copy-id root@master01.example.com
```
### SELinux requirements
Security-Enhanced Linux (SELinux) must be enabled on all of the servers before installing OpenShift Container Platform or the installer will fail. Also, configure SELINUX=enforcing and SELINUXTYPE=targeted in the /etc/selinux/config file:
```
# vi /etc/selinux/config
# This file controls the state of SELinux on the system.
# SELINUX= can take one of these three values:
#     enforcing - SELinux security policy is enforced.
#     permissive - SELinux prints warnings instead of enforcing.
#     disabled - No SELinux policy is loaded.
SELINUX=enforcing
# SELINUXTYPE= can take one of three two values:
#     targeted - Targeted processes are protected,
#     minimum - Modification of targeted policy. Only selected processes are protected.
#     mls - Multi Level Security protection.
SELINUXTYPE=targeted
```
### Verify Network Manager is active on all nodes
```
systemctl status NetworkManager
systemctl enable NetworkManager
systemctl start NetworkManager
```
### Enable NTP
-	Execute the following command as root on all the servers:
```
systemctl start ntpd.service
systemctl enable ntpd.service
ntpq -p
```
### CONFIGURING AUTHENTICATION
The OpenShift Container Platform authentication is based on OAuth, which provides an HTTPbased API for authenticating both interactive and noninteractive clients. The OpenShift installer uses a secure by default approach, where DenyAllPasswordIdentityProvider is the default provider. Using this provider, only the local root user on a master machine can use OpenShift client commands and APIs.
You must configure another identity provider so that external users can access the OpenShift cluster.
#### htpasswd Authentication
OpenShift HTPasswdPasswordIdentityProvider validates users and passwords against a flat file generated with the Apache HTTPD htpasswd utility. This is not enterprise-grade identity management, but it is enough for a proof of concept (POC) OpenShift deployment.

Set HTPasswdPasswordIdentityProvider in the identityProviders stanza to validate user names and passwords against a flat file generated using htpasswd. The htpasswd utility is in the httpd-tools package:

On the master node run the following:
```
yum install httpd-tools -y
```
To create a flat file with a user name and hashed password, run
```
mkdir -p /etc/origin/master/
htpasswd -c -b /etc/origin/master/htpasswd admin yourPassword
```
To add additional user to the file use the following:
```
htpasswd /etc/origin/master/htpasswd salah
```
### Customize Inventory file for Installation
Ansible inventory files describe the details about the hosts in your cluster and the cluster configuration details for your OpenShift Container Platform installation.

Go to ansible files located at: /etc/ansible/hosts.
-	As a good practice rename your hosts file.
```
mv hosts hosts.org
```
-	Create a new host file for our inventory and add the following to it
```
# vi hosts
[OSEv3:children]
masters
nodes
etcd
# Set variables common for all OSEv3 hosts
[OSEv3:vars]
# SSH user, this user should allow ssh based auth without requiring a password
ansible_ssh_user=root
openshift_deployment_type=openshift-enterprise
openshift_disable_check=memory_availability,disk_availability
openshift_master_default_subdomain=app.example.com
# htpasswd auth
openshift_master_identity_providers=[{'name': 'htpasswd_auth', 'login': 'true', 'challenge': 'true', 'kind': 'HTPasswdPasswordIdentityProvider’}]

openshift_router_selector='node-role.kubernetes.io/infra=true'
openshift_registry_selector='node-role.kubernetes.io/infra=true'

## Enable using firewall instead of iptables
os_firewall_use_firewalld=true

# Defining htpasswd users

oreg_auth_user=12427188|os-lab-01
oreg_auth_password=eyJhbGciOiJSUzUxMiJ9.eyJzdWIiOiJjZGFmZTI5YWU0ZjY0NjEzODRmZWVjZGJlY2Q4YjMzYiJ9.WBQXPPxx6Cnq4GE2HbhaDNEEZbuz_pl_jbnhba5oyoexi6se0JPNnSy6RjB2dMD6eFBcqOfxhuvn-JHaPVmVfxN8D0bwrGZ9I8x3VYZLz5tFxmNj9JqabjxyTEDfPd2F7o4gt-vAN835159ditjShax480qZ_OypY0Sbm7h3klweXNzT91pcRxH4PXTrMZG_vRmn5HH_-KU6bs68xPqzaA9B4PIxYrPJlk6uuIeTZvbL5bG_lO05ZhklWDe-Y754P-lCofULLvhzeWUvLBeg_8YEyJfftWxqq95BlVKuec0RqcOIFXVSTnXe9-LzXoACFwhFqS4i9FgpgUpr_nrFkLxB9SlOr_wmn4DuexO-eHSvfnczM03l1UPSRuXIGCCn1GXttJ99hBg_Wp6yN2osz_zMTYcRe5dtL4aWPwCnywkyVajJIiCI8lhnuewm_dnUti0tj0rAuPbyIiPehmrEjFPkdd6rBnqzus7CKJThG2dVZ-yIImlJOsmO5D1JtP98qMLeTev-JVM9gqtxXw0Z7329VaGyVy-OWMGYc2hb93IWopB0XCi3NGPOCuzlFKgwfG-CVjTg9BKYrP68Avi7YiqXCg2RApXe-0Osyor33Q33EXLpuajtkZSGo2N9obAmEmTvAkuDLyokJOYxB5oUyrPaSmlpAJXSvYZpDFDZWSI

### Enable using different port instead of default 8443
openshift_master_api_port=443
openshift_master_console_port=443


# host group for masters
[masters]
master01.example.com

# host group for etcd
[etcd]
master01.example.com

# host group for nodes, includes region info
[nodes]
master01.example.com openshift_node_group_name='node-config-master'
node01.example.com openshift_node_group_name='node-config-compute'
infra.example.com openshift_node_group_name='node-config-infra'
```
-	Test the accessibility to all hosts by ping them using ansible
```
ansible all -m ping
```
-	Then we need check all our prerequisites are set. Go to /usr/share/ansible/openshift-ansible/playbooks/ and run
```
cd /usr/share/ansible/openshift-ansible/playbooks/
ansible-playbook prerequisites.yml
```
-	After successfully run prerequisites. Run the cluster deploy using following command:
```
ansible-playbook deploy_cluster.yml
```

### Post Installation
As the system:admin user, add the cluster-admin role to the admin user with the following command:
First login as system:admin if you aren’t. First check using root user by typing

```
oc whoami
oc login -u “system:admin”
oc adm policy add-cluster-role-to-user cluster-admin admin
```
Log to cluster using the admin user
```
oc login -u admin -p admin4redhat https://master01.example.com:8443 --insecure-skip-tls-verify=true
```
**Verifying Node Status**
To verify that all openshift nodes are in a ready state, execute the oc get nodes command:
```
oc get nodes
```
## Installing and configuring the Metrics Subsystem
The OpenShift metrics subsystem enables the capture and long-term storage of performance metrics for an OpenShift cluster. Metrics are collected for nodes and for all containers running in each node.

![](./images/openshift-metrics-subsystem.png)
The metrics subsystem is deployed as a set of containers based on the following open source projects:
__Heapster__
Collects metrics from all nodes in a Kubernetes cluster and forwards them to a storage engine for long-term storage. Red Hat OpenShift Container Platform uses Hawkular as the storage engine for Heapster.
The Heapster project was incubated by the Kubernetes community to provide a way for third- party applications to capture performance data from a Kubernetes cluster.
__Hawkular Metrics__
Provides a REST API for storing and querying time-series data. The Hawkular Metrics component is part of the larger Hawkular project.
Hawkular Metrics uses Cassandra as its data store. Hawkular was created as the successor to the RHQ Project (the upstream to Red Hat JBoss Operations Network product) and is a key piece of the middleware management capabilities of the Red Hat CloudForms product.
__Hawkular Agent__
Collects custom performance metrics from applications and forwards them to Hawkular Metrics for storage. The application provides metrics to the Hawkular Agent.
