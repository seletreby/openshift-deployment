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
