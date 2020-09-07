---
title:       "GCP Network Connection (1)"
subtitle:    "gcp network connections - VPC peering/VPN - poc"
description: "gcp network connections - VPC peering/VPN - poc"
date:        2020-09-07
author:      "Jamie Zhang"
image:       "img/background-10.jpg"
tags:        ["GCP", "VPC", "Mongo"]
categories:  ["Cloud" ]
published: true
---

To document the setup process for VPC peering accross different GCP project VPCs for demo purpose only.

# Setup VPC peering accross two GCP projects
in this demo, will create two GCP projects, and setup VPC network peering to work as network bridge to make sure the vm in one of GCP project can access the MongoDB installed on VM of another GCP project.
<img src='/img/2020-09-07-gcp-network-connection/vpc-peering-network.png' style="height: 380px;margin-left: 0px;"/>

## preparation
### Mongo backups
1> backup MongoDB  
to use mongo shell command to backup data, samples as below
```
mongodump --host 127.0.0.1 --port 27017 --out /Users/jamie/Documents/work-benches/mongo/dump/20200907 --gzip --collection users --db test
```

2> install Google Cloud SDK in local  
please refer to https://cloud.google.com/storage/docs/gsutil_install  

>__output__:  
> 2020-09-07T11:20:57.540+0800    writing test.users to   
2020-09-07T11:20:57.792+0800    done dumping test.users (10004 documents)

3> upload the backup file to Cloud Storage
```
gsutil cp -r ../20200907 gs://mongo-backup-repo
```

### GCP resource creation
|Resource|GCP project|GCP network|GCP Subnet|Remark|
| :----| :----| :----: | :----: | :---- |
|test-vm-01|test-project-01|test-vpc-01|asia-east1|with Mongo server installed and init data use the backups <br/> from Cloud Storage|
|test-vm-02|test-project-02|test-vpc-02|asia-east1|with MongoShell installed|

Notes:   
1.VPC network & subnet & firewall  
create a custom mode VPC with one subnet  

> VPC  
> name: test-vpc-01  
> subnet mode: Custom subnets  
> routing mode: Regional   
> 
> Subnet  
> name: asia-east1  
> region: asia-east1    
> ip ranges: 10.140.0.0/20  
> private google access: on
> 
> ---
> 
> name: test-vpc-02  
> subnet mode: Custom subnets  
> routing mode: Regional   
> 
> Subnet  
> name: asia-east1  
> region: asia-east2    
> ip ranges: 10.150.0.0/20  
> private google access: on
> 

for custom mode VPC, no any default firewall rules would be created, you must create it by yourself.
<img src='/img/2020-09-07-gcp-network-connection/test1-firewall-01.png' style="height: 250px;margin-left: 0px;"/>
<img src='/img/2020-09-07-gcp-network-connection/test1-firewall-02.png' style="height: 320px;margin-left: 0px;"/>

2.Create VM instance on Google Compute Engine & restore the MongoDB data  
please refer to https://docs.mongodb.com/manual/tutorial/install-mongodb-on-debian/ to install mongo on VM

```
gsutil cp -r gs://mongo-backup-repo/20200907 ./backup/

mongorestore --gzip --archive= ./backup/20200907/test/users.bson.gz --db test
``` 
## Configure VPC Peering
both side create VPC network peering
1. vpc peering from test-vpc-01 to test-vpc-02  
<img src='/img/2020-09-07-gcp-network-connection/vpc-peering-01-02.png' style="height: 420px;margin-left: 0px;"/>

2.vpc peering from test-vpc-02 to test-vpc-01
<img src='/img/2020-09-07-gcp-network-connection/vpc-peering-02-01.png' style="height: 420px;margin-left: 0px;"/>

3.test connectivity
1> ping from both side, should be successfully from both side.  

2> install MongoDB shell on test-vm-02 to verify the mongo connection works as expected.  
refer to https://docs.mongodb.com/manual/tutorial/install-mongodb-on-debian/, only install the the mongo shell.
```
sudo apt-get install -y mongodb-org-shell=4.4.0 
```
<img src='/img/2020-09-07-gcp-network-connection/vpc-peering-result-01.png' style="height: 350px;margin-left: 0px;"/>

notes: _regarding how to secure the MongoDB access, please find the reference on the bottom of this page._

Userful References:  
https://cloud.google.com/vpc/docs/using-vpc-peering  
https://ciphertrick.com/setup-mongodb-authentication-connect-using-mongoose/  


