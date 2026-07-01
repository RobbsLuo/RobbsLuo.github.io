---
title: On Architecture
date: 2018-05-16 11:26:29
tags:
  - Architect
  - Humanity
categories:
  - Architect
description: A systematic introduction to architecture
lang: en
---

Recently I ran an internal architecture exchange/training session at my company, covering the concept of architecture, its forms, and the principles of architectural design.
[View the slides](http://www.rowkey.me/arch-ppt/index.html)

## What Is Architecture
* Personnel allocation and project planning within a department
* Floor planning and functional facility planning in building design
* Road layout, functional building design, and entertainment facility design in a city
* Urban planning, expressway planning, and high-speed rail route planning for a country
>**All is architecture!!!**

## The Essence of Architecture
* **Core lifecycle**: Sub-lifecycles whose subject remains unchanged after splitting.
* **Non-core lifecycle**: Sub-lifecycles whose subject changes after splitting.
  ![](Talk-About-The-Architecture/1526437772448-7783bdeb-4306-4847-85db-3754065cdea8-image.png)
> **The essence of architecture lies in continuously splitting lifecycles (in a tree structure) so that business can run in parallel spatially. Each lifecycle that is split off has its own boundary and does not affect other lifecycles; all changes are settled within its own lifecycle. This is what high cohesion means.**

## Software Architecture
> **Software lifecycle: software development lifecycle + software runtime lifecycle (software access, software features, software monitoring)**

![](Talk-About-The-Architecture/1526437909260-9eff08f6-b8c8-436d-81ff-2ebc3f639464-image.png)

## Three Elements of Good Software Architecture
* **Firmness**: Achieve a satisfactory level of freedom from damaging failure.
* **Commodity**: Utility to accomplish the tasks it is purported to be for.
* **Delight**: Pleasure in use.
> **Buildings (Solid, Useful, Beautiful) -> Software (Firmness, Commodity, Delight)**

## Overview of the Architecture Process
1. **Business architecture**: A top-down view of the architecture, including business rules, business modules, and business processes. It mainly decomposes the business of the whole system, designs the domain models, and transforms real-world business into abstract objects.
2. **Technical architecture**: A cross-sectional view of the architecture, an abstraction from hardware to application, including abstraction layers and programming interfaces. Technical architecture and business architecture are complementary; every part of the business architecture has its own technical architecture. These two parts must be done well first.
3. **Data architecture**: The storage architecture, mainly referring to the design of data structures. It determines the characteristics of the application's data sources and is the foundation of both business architecture and technical architecture.
4. **Deployment architecture**: The topology architecture, including how many nodes the system is deployed on, the relationships between nodes, high availability of servers, fault tolerance, network interfaces and protocols, etc. It determines how the application runs, its runtime performance, maintainability, and scalability, and is the foundation of all architectures.
5. **Organizational architecture**: The team architecture, including the organizational form of the project, personnel composition, and responsibilities. It is the supporting facility for all the architectures above. A good organizational architecture ensures the effective implementation and advancement of the other architectures.
> **As business and load change, the architecture needs to be continuously reviewed and refactored to drive its evolution.**

## Business Architecture
![Top-down view](Talk-About-The-Architecture/1526438152354-6a181918-13db-4d45-855f-88257de3f429-image.png)
* Business execution is the core module of the application and its primary function.
* Data analysis is an auxiliary module of the application, helping with data-driven R&D, business intelligence research, and improving user experience.
* System management is the foundational part of the application. Doing a good job of deployment, monitoring of various metrics, and backup of critical data helps the application iterate and deploy quickly and run stably.

## Technical Architecture - Overview
![Cross-sectional view](Talk-About-The-Architecture/1526438216252-267d0e5e-63b3-44dc-b988-c9eb1758a10f-image.png)
* Business data sources, data rule engines, and analysis rules support the presentation of the interactive UI.
* Infrastructure and common services build up the underlying business logic.

## Technical Architecture - Specifics
![](Talk-About-The-Architecture/1526438239613-1cdabcae-9196-4aad-869d-94285ae76d41-image.png)
* Both UI and applications belong to the application layer, which provides the concrete business implementation. Among them, UI is the primary form of expression for the view layer.
* Applications and services belong to the control logic and access channels, where services hold the main business logic. Services can expose service interfaces for applications to call.
* Core, drivers, and data make up the data layer. This is the operation logic for all data related to the business and corresponds to the model layer. Among them, the core can expose interfaces outward for services to call.

## Evolution of Architecture
![](Talk-About-The-Architecture/1526438275615-26fee321-9534-4a04-b51e-7806434b66e7-image.png)
* In a monolithic application architecture, all logic resides within the application and core modules, including business logic, data operations, and so on.
* Through a data-driven approach, data operation interfaces are unified to shield the differences between data sources, allowing vertical partitioning based on resources.
* Modules are decoupled and expose APIs outward as services, forming a distributed service architecture.

## Deployment Architecture
![](Talk-About-The-Architecture/1526438314473-c5da5a10-c62d-49a0-9209-5e43e54b88ca-image-resized.png)
* The interactive UI is deployed independently, including forms such as web and app.
* The application is deployed independently as a single node or a cluster.
* Services, the core, and drivers are deployed as a whole.
* Data sources are deployed independently.
* A simple application/service cluster: LVS (using keepalived for active/standby) + Nginx (reverse proxy) + Tomcat (business container).

## Data Architecture
![](Talk-About-The-Architecture/1526438343067-0e351afe-72f7-4a7f-a6be-72fab9cd54c8-image.png)
* The data interaction logic and data flow presented by the interactive UI determine the main data design of the business.
* The original business data, logs, and the data needed for statistics support the report output required by data analysis.
* Real-time information, under certain rules and state machine engines, can power dashboard features such as real-time status monitoring.

## Five Attributes of Data
* **Access frequency**: Read/write frequency. Read-only and frequently accessed data can be redundantly stored in multiple copies.
* **Consistency requirement**: Data with high consistency requirements must be strictly guaranteed to be accurate.
* **Access permission**: In API design, data of different granularities is exposed according to different permissions. PO -> VO is a description of the same entity under different permissions.
* **Data importance**: Cannot be lost / partial loss allowed / only a cache / no need to persist.
* **Data confidentiality**: Plaintext internally allowed / plaintext internally not allowed / can be made public.

## Data Design
* Fully understand the interactive UI so you know which data the UI is associated with and which data can be cached.
* Fully understand the business so you know clearly which data needs to be recorded and the relationships between data.
* Database design should pay attention to storage efficiency:
  - Reduce transactions
  - Reduce JOIN queries
  - Use indexes appropriately
  - Consider using caching
  - ...
* In data statistics scenarios, data statistics with high real-time requirements can use Redis; non-real-time data can use separate tables, with data updated through asynchronous queue computation or scheduled calculations. In addition, for statistical data with high consistency requirements, transactions or scheduled reconciliation mechanisms are needed to ensure accuracy.

## What Is an Architect
* Lifecycle identification: rationally split lifecycles.
* Identify problems and the subject of the problem. Never mistake a solution for a problem. Discovering a problem is always more important than solving it!!!
* Focus on business and technology, and ensure business growth.
* OKR architecture: be responsible for breakthroughs in key technologies, resolve technical feasibility issues, and deliver the key results that take things from 0 to 1.
* **Authority must match responsibility, to ensure the architecture is executed!!!**

## Essential Qualities of an Architect
* Stand high, see far, dig deep.
* Master a certain technology so you can draw analogies at a fundamental level and quickly grasp other technologies.
* Treat all technologies equally: there is only suitable or unsuitable, never liked or disliked.
* Have a broad vision, understand the pros and cons of different technologies, know which open-source project can directly satisfy this or that requirement, and be able to judge whether you need to reinvent the wheel.
* Master design patterns, but do not overuse them.
* Split the system into multiple subsystems or modules, keeping modules as loosely coupled as possible, so that development tasks that previously could only run serially can proceed in parallel, and the timeline can be shortened by investing more people.
* Clearly know where the system's bottlenecks are, continuously locate bottlenecks in technical difficulty, R&D progress, performance, memory, and other aspects, constantly assign key personnel to resolve bottlenecks, and eliminate hidden risks before they explode.
* Be able to anticipate how requirements may change and make forward-looking designs accordingly.

## Six-Step Architecture Thinking Method
![Six-Step Architecture Thinking Method](Talk-About-The-Architecture/1526438498119-90bbb34f-ef90-43cb-939b-aa0f696262ce-image.png)

## Architecture Principles - Overview
![](Talk-About-The-Architecture/1526438513492-00ba90a2-24fb-460d-aa4c-d30bed417433-image.png)

## Architecture Principles
* **Avoid over-engineering**: The simplest solution is the easiest to implement and maintain and also avoids wasting resources. But the solution should include provisions for extension.
* **Redundancy design**: Provide node redundancy for services and databases to ensure high availability. This is achieved through database master-slave replication and application clusters.
* **Active-active data centers**: For disaster recovery and to fundamentally guarantee high availability of the application. Multiple active data centers must be built so that a failure of one data center due to uncontrollable factors does not render the entire system unavailable.
* **Stateless design**: APIs, interfaces, and so on must not have front-back dependencies; one resource is not affected by changes to another. A stateless system can scale better. If state is unavoidable, either the client manages it or the server manages it with a distributed cache.
* **Rollback capability**: Any business, especially critical business, must have a recovery mechanism. This can be implemented with log-based WAL or event-based Event Sourcing, etc.
* **Disabling / self-protection**: Provide rate-limiting mechanisms so that when upstream traffic exceeds the system's load capacity, overflowing requests can be rejected. This can be done via manual switches or automatic switches (monitoring abnormal traffic behavior) to block traffic at the front of the application.
* **Traceability**: When a problem occurs in the system, you can locate the trajectory of the request and the request information at each step. Distributed tracing systems solve this kind of problem.
* **Monitorability**: Being monitorable is key to keeping the system running stably. This includes monitoring of business logic, application processes, and system resources the application depends on (CPU, disk, etc.). Every system needs to monitor well at these levels.
* **Fault isolation**: Isolating the resources (threads, CPU) and services that a system depends on ensures that a failure of one service does not affect calls to other services. Fault isolation can be achieved through thread pools or by deploying nodes separately.
* **Mature and controllable technology selection**: Use mainstream, mature technologies with good documentation and ample support resources. Choose the appropriate technology rather than the hottest one to implement the system.
* **Tiered storage**: Memory -> SSD disk -> traditional hard disk -> magnetic tape. Data can be stored in tiers according to its importance and lifecycle.
* **Caching design**: Isolating requests from backend logic and storage is a mechanism based on the locality principle. This includes client-side caching (pre-distributing resources), Nginx caching, local caching, and distributed caching.
* **Asynchronous design**: For interfaces where the caller does not care about the result or allows a delayed result, responding asynchronously via a queue can significantly improve system performance; when calling other services, not waiting for the service to return before returning directly also improves response performance. Asynchronous queues are also a common means of solving distributed transactions.
* **Forward-looking design**: Based on industry experience and judgment, design extensibility and backward compatibility in advance.
* **Horizontal scaling**: Compared to vertical scaling, being able to solve problems by adding more machines is the top priority, and the system's load capacity can scale close to infinitely. In addition, automatically adjusting capacity based on system load via `cloud computing` technology can save costs while ensuring service availability.
* **Build and release in small steps**: Iterate the project quickly, and fail fast. Avoid project plans with overly long spans of time.
* **Automation**: Automating packaging and testing is called continuous integration; automating deployment is called continuous deployment. Automation is the fundamental guarantee for rapid iteration and trial-and-error.

## Architecture Principle - Scalability
![Three-axis scaling theory](Talk-About-The-Architecture/1526438712437-0ddc81c3-0e3f-437a-8b2f-721e5a730e49-image.png)
* X-axis: horizontal duplication or cloning, goal-oriented. Examples include database read-write separation, table replication, replication, etc. Monolithic applications or dependent services are made redundant, and load balancing improves the system's load capacity.
* Y-axis: function/service-oriented, such as vertical applications and distributed services. This splits a monolithic application into smaller applications or services based on function.
* Z-axis: resource-oriented, such as horizontal database sharding. Resources are partitioned to spread the load across different nodes.
> * **Avoid relying on the database's computational features (functions, stored procedures, triggers, etc.). Put the load on the business application side, which is easier to scale.**
> * **Scalability plan principle: Design for 20x, implement for 3x, deploy for 1.5x (DID).**

## Five Techniques for Improving System Response Performance
* **Asynchrony**: Queue buffering, asynchronous requests.
* **Concurrency**: Use multiple CPUs and threads to execute business logic.
* **Locality principle**: Caching, tiered storage.
* **Reduce IO**: Merge fine-grained interfaces into coarse-grained ones; for frequent overwrite operations, only perform the last one. One thing to pay special attention to here: **avoid calling external services inside loops in your code. A better practice is to use a coarse-grained batch interface outside the loop and make a single request.**
* **Partitioning**: Keep the size of frequently accessed data sets within a reasonable range.

## References
* "Scalability Rules" by Martin L. Abbott / Michael T. Fisher (Chinese edition edited by Chen Bin)
* "Liao Liao Jia Gou (On Architecture)" by Wang Gaikai
* "An Architect's First Lesson" by Cai Xueyong
* Six-Step Architecture Thinking Method by Xia Huaxia @ Meituan

---
> Source: http://www.rowkey.me/arch-ppt/index.html
