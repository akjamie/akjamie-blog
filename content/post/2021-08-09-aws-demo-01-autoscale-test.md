---
layout: post
title:       "Demo - AWS EC2 instance auto scaling"
subtitle:    "AWS PCA Prep Series"
description: "To mark down some key steps for later review"
date:        "2021-08-09"
author:      "Jamie Zhang"
image:       ""
tags:        ["AWS", "Cloud"]
categories:  ["Cloud" ]
---

I'm on the path of AWS Professinal Certified Architect exam preparation, want to document some key points in the demos for later review to enhance the understanding of knowledge. In this demo, i'm going to test the AWS auto scaling and utilize the EC2 network knowledges. 

# Objective
1. To build a health checking springboot application and package as docker image hosted on AWS ECR.  
2. Setup basic network infrastructure, VPC, Subnet, Internet Gateway, Route table,Security group, etc.  
3. Create Auto Scaling Group using prepred Lanuch Template, and expose the access entry through Application Loader Balancer.  
4. To use Apache HTTP server benchmarking tool(ab) to trigger billions of request to test the instance scale out/in.  

# Architecture diagram
high level architecture diagram is for easier understanding of this demo. The diagram illustrates an application load balancers to route traffic to EC2 instances hosted in subnet-a(ap-northeast-1c) and subnet-c(ap-northeast-1d), these EC2 instances also are managed by auto scaling group which could automate the capacity management and fault tolerance and ensure the capacity to handle the current traffic demand. Also notification service configured on auto scaling group to timely notify the scaling machanism.
<img src='/img/2021-08-09-aws-demo-01-autoscale-test/aws-auto-sacle-test-arch.png' style="height: 810px;margin-left: 0px;"/>

# Implementation
## Prepare the Docker image
build a simple springboot application and expose a /health interface. let's see the controller.
```
@GetMapping("/health")
public ResponseEntity hello(HttpServletRequest request) {
    final String sourceIp = Utils.getIpAddr(request);
    final String localIp = Utils.getLocalIpAddress();
    final String hostName = Utils.getHostName();

    HashMap<String, Object> map = new HashMap<>();
    map.put("sourceIp", sourceIp);
    map.put("targetIp", localIp);
    map.put("hostName", hostName);
    map.put("status", "UP");

    return ResponseEntity.status(HttpStatus.OK).contentType(MediaType.APPLICATION_JSON).body(map);
}
```

use jib-maven-plugin to build docker image and push the docker image to aws ECR(create a repository first), configure the build plugin as below shown.

```
<plugin>
    <groupId>com.google.cloud.tools</groupId>
    <artifactId>jib-maven-plugin</artifactId>
    <version>${jib.version}</version>
    <configuration>
        <from>
            <image>openjdk:alpine</image>
        </from>
        <to>
            <image>public.ecr.aws/a0i4s5a9/${project.artifactId}:${project.version}</image>
            <credHelper>ecr-login</credHelper>
        </to>
        <container>
            <creationTime>USE_CURRENT_TIMESTAMP</creationTime>
        </container>
        <allowInsecureRegistries>true</allowInsecureRegistries>
    </configuration>
</plugin>
```

after tested the interface, use **mvn clean compile jib:build** to build the image, once done you could see the image in AWS ECR.
<img src='/img/2021-08-09-aws-demo-01-autoscale-test/aws-ecr-image-health.png' style="height: 87px;margin-left: 0px;"/>

## Create infrastructure
To create below resources for this demo:  

| Types | Resource(ID/Name) | Region/AZ | Remark |
| :--- | :---- | :--- | :--- |
| VPC | vpc-0e4918ac3af48cba4 | ap-northeast-1(Tokyo) | CIDR: 172.32.0.0/16, DNS hostnames enabled |
| Internet Gateway| igw-018b97e71d32a2ecd | ap-northeast-1 | attach to VPC and subnet Route Table to enable internet access|
| Subnet | subnet-a | ap-northeast-1c | CIDR: 172.32.0.0/20, public subnet|
| Subnet | subnet-b | ap-northeast-1c | CIDR: 172.32.16.0/20, public subnet|
| Subnet | subnet-c | ap-northeast-1d | CIDR: 172.32.32.0/20, public subnet|
| Security Group| sg-0e5aae3682dea4349/security-group-external| | inboound: all ICMP - ipv4 and http/https|
| Security Group| sg-044e680ce2a997f4a/security-group-external-management| | quite open and should add IP range constrains|
| Security Group| sg-0112367fdf4f05fe5/security-group-internal| | inboound: all traffic from sg-044e680ce2a997f4a and http/https from sg-0e5aae3682dea4349 |

## Create instance templete and auto scaling group
create instance template to use t2.micro instance type and disable public ipv4, the ec2 instance backed by auto scaling group are only accessible from the bastion host secured by security-group-external-management. for the instance count, minumum: 1, desired: 2, maxium: 3

no much details here on this step, the wizard page on aws console is quite clear to guide you create these parts, you can also try Terra form for these resource creation.

notes: the customer AMI may bring additional charges to your account free tier limit, hence i use user data to init the docker run environement instead, the content of user data is listed in below:
```
#! /bin/sh
sudo yum update -y
sudo amazon-linux-extras install docker
sudo service docker start
sudo usermod -a -G docker ec2-user

docker run -d -p 80:8080 --name health-check public.ecr.aws/a0i4s5a9/utilities-healthcheck:1.0.1
```

## Test connectivity
when goes to this step, that mean the ELB and auto scaling group as well the target group/health check related configuration have been done, now test the connectivity.
access the FQDN display on the ELB detail page on browser directly, if get expected response, then great, if not, dont be sad, please check the security group settings and the subnet routes again.

# Pressure testing
install apacke ab to make request, please see below picture
<img src='/img/2021-08-09-aws-demo-01-autoscale-test/ab-test.png' style="height: 400px;margin-left: 0px;"/>

check the traffic change status on the ELB monitoring dashboard, capture a screenshot for reference:
<img src='/img/2021-08-09-aws-demo-01-autoscale-test/picture-requests.png' style="height: 400px;margin-left: 0px;"/>

CPU utilization on the auto scale group monitoring dashboard:
<img src='/img/2021-08-09-aws-demo-01-autoscale-test/cpu-utilization.png' style="height: 426px;margin-left: 0px;"/>


Instances count during the peak traffic volume:
<img src='/img/2021-08-09-aws-demo-01-autoscale-test/instance-list.png' style="height: 320px;margin-left: 0px;"/>

last but not the least, during the test preparation, some additional changes generated, here paste the bills for reference, hope helps.
<img src='/img/2021-08-09-aws-demo-01-autoscale-test/bills.png' style="height: 629px;margin-left: 0px;"/>

# Others  
## Issues encountered
The whole demo process is not smooth as write a blog, actually encountered lots of issues, for example:  
1. Load balancer only could point to instance in public subnet, at the beginning i set the subnet-a and subnet-c as private subnet, and create NAT for out bound access since installation of docker.  
2. Bation host created in the subnet-b cannot ssh to the ec2 instance created by the auto scaling group, lately i found i didn't configure the private key on the bastion host. because all the instance shared the same RAS key pair, so once i stall the private key on the bastion host, i can ssh to any instances created using the key pair by given security access is allowed  
3. NAT is not free of charge, it charges by hour!!  
4. EC2 instance created but cannot access even allow PUBLIC IPV4 or associate elastic IP, because there is no internet routing config in the Route Table for self created subnets.  
5. User data specified on the EC2 instance does not work, check the EC2 logs found it cannot execute the user date correctly, after investigation, the root cause is missing **#! /bin/sh** on the first line of the user data.

there are some other security group configure issue, while these are eaiser to fix ^_^.

Thank you for reading.



