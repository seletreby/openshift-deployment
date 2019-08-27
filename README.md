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
