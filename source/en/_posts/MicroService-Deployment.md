---
title: Microservice Deployment: Blue-Green Deployment, Rolling Update, Canary Release
date: 2018-06-05 09:49:21
tags:
    - Deployment
    - MicroService
categories:
    - DevOps
lang: en
description: During the iterative process of a project, "going live" is inevitable. Going live corresponds to deployment, or redeployment; deployment corresponds to changes; and changes mean risk.
---

> **During the iterative process of a project, "going live" is inevitable. Going live corresponds to deployment, or redeployment; deployment corresponds to changes; and changes mean risk.**

There are currently many techniques used for deployment—some simple, some complex; some require downtime, while others can complete the deployment without downtime. The purpose of this article is to summarize the deployment schemes currently in common use.

## 1. Blue/Green Deployment

### 1.1. Definition

Blue-green deployment means that, without taking down the old version, you deploy the new version and then test it. Once confirmed to be OK, you switch traffic to the new version, and then the old version is also upgraded to the new version.

### 1.2. Characteristics

Blue-green deployment requires no downtime and carries relatively little risk.

### 1.3. Deployment Process

**Step 1.** Deploy version 1 of the application (the initial state).
All traffic from external requests is directed to this version.

![](MicroService-Deployment/1528248341535-ce4d52c7-efb4-492c-adcb-5a26eb55b089-image.png) 

**Step 2.** Deploy version 2 of the application.

The code of version 2 differs from version 1 (new features, bug fixes, etc.).

**Step 3.** Switch traffic from version 1 to version 2.

![](MicroService-Deployment/1528248357041-38b4cee4-d452-48fe-a395-befc390c6d6b-image.png) 

**Step 4.** If version 2 passes testing normally, delete the resources (for example, instances) being used by version 1, and from then on use version 2 officially.


### 1.4. Summary

From the process, it is not hard to see that our application remains online throughout the deployment. Furthermore, during the rollout of the new version, nothing in the old version is modified, so the state of the old version is unaffected during deployment. This keeps the risk very low, and as long as the old version's resources are not deleted, we can theoretically roll back to the old version at any time.

### 1.5. Considerations for Blue-Green Deployment

When you switch to the blue environment, you need to properly handle in-flight business and new business. If your database backend cannot handle this, it can be a rather troublesome problem.

* You may encounter situations that require handling both a "microservice-architected application" and a "traditional-architected application" at the same time. If the two are not coordinated well within the blue-green deployment, it can still lead to service interruptions.
* You need to consider the issue of synchronized database and application migration / rollback in advance.
* Blue-green deployment requires infrastructure support.
* Performing blue-green deployment on a non-isolated infrastructure (VM, Docker, etc.) carries the risk that either the blue environment or the green environment could be disrupted.

## 2. Rolling Update

### 2.1. Definition of Rolling Update

Rolling update: generally, one or more servers are taken out of service, updated, and then put back into use. This process repeats until all instances in the cluster have been updated to the new version.

### 2.2. Characteristics

Compared with blue-green deployment, this deployment method is more resource-efficient—it does not require running two clusters or twice the number of instances. We can deploy incrementally, for example, taking out only 20% of the cluster for upgrade at a time.

This approach also has many drawbacks, for example:

* There is no definitively OK environment. With blue-green deployment, we can clearly know that the old version is OK, whereas with rolling updates, we cannot be certain.

* It modifies the existing environment.

* Rollback is very difficult if needed. For example, suppose in a given release we need to update 100 instances, updating 10 instances at a time, with each deployment taking 5 minutes. When the rolling update reaches the 80th instance and a problem is found that requires a rollback, that rollback is a painful and lengthy process.

* Sometimes we may also dynamically scale the system. If the system automatically scales up or down during deployment, we also have to determine which node is running which code. Although there are some automated operations tools, it is still nerve-wracking.

* Because the update is gradual, there will be a brief period during the rollout when the old and new versions coexist inconsistently. If the scenario has strict requirements on the release, you need to consider how to ensure compatibility.

## 3. Canary Deployment (Canary Release)

### 3.1. Definition

A canary release is a release method that allows for a smooth transition between black and white. A/B testing is one form of canary release: let some users continue using A while others start using B. If users have no objections to B, gradually expand the scope and migrate all users to B. Canary release can ensure the stability of the overall system, allowing problems to be discovered and adjusted during the initial canary phase to limit their impact. The canary deployment we usually talk about is one form of canary release.

> Note: The canary in the coal mine. In the 17th century, British miners found that canaries were extremely sensitive to firedamp (methane gas). Even the slightest trace of firedamp in the air would cause the canary to stop singing; and when the firedamp concentration exceeded a certain threshold, although the dull-witted humans remained oblivious, the canary would have already succumbed to the poison. At the time, given the relatively rudimentary mining equipment, workers would bring a canary down into the mine each time as a "firedamp detection indicator" so that they could make an emergency evacuation in a dangerous situation.

**The structure of a canary release is shown below:**

![](MicroService-Deployment/1528248583114-c63637d7-9059-4b89-bc0c-3d6adf914253-image.png) 

### 3.2. Steps

* Prepare the artifacts for each stage of deployment, including: build artifacts, test scripts, configuration files, and deployment manifest files.

* Remove the "canary" server from the load balancing list.

* Upgrade the "canary" application (drain its existing traffic and deploy).

* Perform automated testing on the application.

* Add the "canary" server back to the load balancing list (connectivity and health checks).

* If the "canary" passes testing in live use, upgrade the remaining servers. (Otherwise, roll back.)

In addition, a canary release can also set routing weights, dynamically adjusting different weights to validate the new and old versions.
