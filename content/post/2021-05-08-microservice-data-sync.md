---
title:       "Microservice design - practice"
subtitle:    "Data sync between systems"
description: "To mark down a batch data sync accross system solution"
date:        "2021-05-08"
author:      "Jamie Zhang"
image:       "/img/background-05.jpg"
tags:        ["Microservice", "Batch"]
categories:  ["Microservice" ]
---

# Summary
To mark down a batch data sync accross system solution, tested daily sync data volume is around 10w.   
In this solution, adopted the message driven batch processing rather than traditional fixed time schedule batch.  
Also applied the microservice design to eliminate the system dependency and well protect the data/system boundry,which is also aligned with the DEVOPS,,both team could focus on their own domain/services' development and maintenance.

# Techniques used
  - Springboot + Spring cloud config, constrained by infra, services are running on in hourse VM instances.
  - Quartz, based on java, easy to custom against requirements, also support clustering.
  - Oracle database
  - JWT is adopted for system to system authentication and authorization.
  
# Key Components
## Scheduling Platform
The platform is implemented using Springboot + Quartz + JDBC, store the all the Quartz related configs in database, it support clustering and easy to build support portal to server better user experience on operation angle.
## Data Provider
To provide restful API to expose the data rather than database directly, well protected data security and integrity, especially for golden source data.
## Data Consumer
Springboot appliation, mainly to load data for data calculation and aggregation, output data per requirement.
  
# Solution
There are two solutions i proposed, day 1 solution is the one using on production, day 2 solution is what i laterly think about after leave the project.
## Day 1 solution
### Process Flow  
1. One schedule task for data sync configured in Scheduling Platform, triggers by upstream system message.  
2. Once triggered, a work task will be added into working queue and keep scheduled at preconfigured interval untill it's in completed status.    
3. In each round scheduling, it calls downstream rest API to start the data sync process, Downstream api must support idompotent feature and calling Scheduling Platform callback api to report process status.   
4. In downstream internal processing, main thread distributes the workload to sub thread for the each file to be synced, and aggregate the finaly process status, report the reconcilation result back to Scheduling Platform.  
5. Alert mechanism is leveraged in Scheduling Platform to report the exceptional cases like overdue, failure according the collected task process status.  
More details please refer to below diagram.
<img src='/img/2021-05-08-microservice-data-sync/micro-service-data-sync-s1.jpg' style="height: 831px; margin-left: 0px;"/>

### Advantages and drawbacks
1.**Advantages**  
- Centralized task scheduling, and task scheduling implementation in Scheduling Platform is not complex.  
- Task sync status is managed in downstream, it's stateless for Scheduling Platform, it's good to decouple the downstream and Scheduling Platform.  
2.**Drawbacks**  
- Resource utilization rate is quite slow, only one instance is handling the whole data process at a time.  
- It's hard to do extension to support more data sync source, like to sync a new file from another data provider(in rest services), especially on the system to system trust mechanism design.

## Day 2 solution(not verified)
based on the solution one, day2 solution aims to resolve the drawbacks to better optimize the existing data sync and scheduling flow.  
Improvements are:  
1. Bring in task group concept(or actions under task), to enrich the scheduling platform to support both single task and task group, and add SYNC/ASYNC flag in the config, if all the tasks under group are in ASYNC way, then scheduling platform could handle the tasks in async way, so that the traffic could be route the multiple downstream instances to get better resource untilization.  
2. Decoupled the file sync processes, each file has it's own task schedule and status, it's independent with each other, no matter the data source from same system or different system.
3. Each file sync api could have it's separate authentication setting.
<img src='/img/2021-05-08-microservice-data-sync/micro-service-data-sync-s2.jpg' style="height: 688px; margin-left: 0px;"/>

# The end
No matter what techniques are used in the behind, the solution could support business requirement is good solution, at the beginning, not every architect could put all the scenarios identified and unidentified into consideration to good support user's needs. Hence continue to fine tune it is required


