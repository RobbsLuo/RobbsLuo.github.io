---
title: The Road of Standardization Construction of VIPS Operation and Maintenance
date: 2018-05-18 14:22:00
tags:
  - DevOps
  - VIPS
categories:
  - DevOps
description: This article is edited from Wang Junfeng's speech at the GOPS Global Operations Conference 2018 Shenzhen.
lang: en
---

Today's talk covers four areas:

![](The-Road-Of-Standardization-Construction-Of-VIPS-Operation-And-Maintenance/1526622683089-e53de777-e7e9-43a3-9796-082929f6da12-image-resized.png) 

## Problems We Once Faced

Vipshop is a mid-sized internet company in terms of scale, and we hope our practices can serve as a reference for peers of a similar size.

Vipshop's business logic is quite extensive. Beyond e-commerce, it also covers logistics and finance. During the early phase of rapid growth, the priority was to ship features quickly, which resulted in a wide variety of technology architectures. Later on, the introduction of third-party software for the finance business also brought some less-than-standard practices.

Different teams handled different business lines. Because the business lines were not unified, operations had many blind spots, making it hard to share headcount.

We built a large number of platforms for release, change management, and so on, yet it always felt like something was missing. We could not form a strong enough force to support the business teams, and we were operating in a fragmented way. Because the technology stacks were inconsistent, building platforms meant accounting for all kinds of special cases and even making compromises.

![](The-Road-Of-Standardization-Construction-Of-VIPS-Operation-And-Maintenance/1526622757566-c7d84c80-d860-44a9-98eb-d46e267b748e-image-resized.png) 

So we began to reflect:
* First, how can engineers strike a balance among `quality`, `cost`, and `efficiency`? The diagram below highlights the key questions. No one can claim to have perfected all three; the goal is to find the best balance point among them.
* Second, after building so many tooling platforms, why are operations staff still exhausted?
* Third, while racing headlong forward on operations platform construction, how do we stay true to our original intent?

![](The-Road-Of-Standardization-Construction-Of-VIPS-Operation-And-Maintenance/1526622822459-efc353d7-1eb0-4d67-8eae-06818735481e-image-resized.png) 
From this we summarized four lessons:
* First, platform builders must deeply understand the pain points of operations.
* Second, technology selection for platform construction is not the most important factor; truly understanding operations is what matters most. That is not to say technology selection is unimportant — it just is not the most important.
* Third, the degree of standardization sets the ceiling for operations automation.
* Fourth, the level of automation determines new growth points for operations. In a word: standardization is urgent.

![](The-Road-Of-Standardization-Construction-Of-VIPS-Operation-And-Maintenance/1526622880585-b4096aba-b0d8-4c68-9ef6-912d8824cfbd-image-resized.png) 

## The Road of Standardization Construction

###  The Component Concept

![](The-Road-Of-Standardization-Construction-Of-VIPS-Operation-And-Maintenance/1526622977798-6bee3895-00bc-4961-bff5-6315e81afd13-image-resized.png) 

The component mindset laid the foundation for standardization:
* On the technical growth side, our approach is to have component expert teams take charge. The expert team defines the direction of the component and explores best practices, which supports technical accumulation and the skill development of the team members.
* Component-based service-ification: operations staff transform outward into technology output, providing service-based products. Developers only need to consume standard APIs without worrying about underlying details.
* Eliminate business silos and give business teams new goals to pursue.

![](The-Road-Of-Standardization-Construction-Of-VIPS-Operation-And-Maintenance/1526623024415-6b09350f-eb66-483a-a012-88ab0dc81405-image-resized.png) 

### Standardization Construction Blueprint

This is the big picture of standardization construction. Operations standardization is a very large program, so we split it into more than a dozen smaller projects.
![](The-Road-Of-Standardization-Construction-Of-VIPS-Operation-And-Maintenance/1526623121436-ccb8e90d-c823-47ec-bbcf-01fb8520055a-image-resized.png) 

Combined with Vipshop's business characteristics, we broke it down based on the company's specific business situation. On the left is the technology process stack, and on the right are the deliverables. The items in red relate to business characteristics, while those below red relate to operations.

![](The-Road-Of-Standardization-Construction-Of-VIPS-Operation-And-Maintenance/1526623181608-62dd2199-2ef8-4caa-98bc-ae13f28f6d9c-image-resized.png) 

### Configuration Repository Management

When a developer submits a request for operations to change a configuration in production, it is very painful. Does the developer have permission to do it themselves? Or does operations handle it — but operations doesn't understand it, while the developer thinks operations is clueless and can't be bothered to explain. Operations has no time to study hundreds of business systems.

The solution is layered governance: let the specialists do what they specialize in.

For example, when a shopping-cart parameter change affects business logic, that kind of task is handed to the developers. Tasks related to components are handled by the expert team.

Of course, developers cannot directly modify files in some path on production. So we built the Janitors platform, which organizes all kinds of configuration files in a standardized way and grants developers permission to manage the parameters of their own business systems, allowing them to make changes that take effect immediately.

Operations focuses on the layer below, performing operations management based on Puppet.

![](The-Road-Of-Standardization-Construction-Of-VIPS-Operation-And-Maintenance/1526623270372-27bca87a-88b8-4c9e-8dbf-ade82a2418d1-image-resized.png) 

### Monitoring Standardization

When the number of machines exceeds 10,000, Zabbix becomes overwhelmed. Vipshop initially ran multiple sets of Zabbix for monitoring.

![](The-Road-Of-Standardization-Construction-Of-VIPS-Operation-And-Maintenance/1526623312925-f65f893d-6806-404c-bbc1-b2ad4b960bd6-image-resized.png) 

Ideal monitoring:
* Unified, fast, and precise
* A single entry point
* Automated, with standardized monitoring plugins
* No manual intervention required for deploying or adding monitoring
* Standardized monitoring plugins. Everyone has their own idea of what monitoring standards should be, and the result is multiple monitoring metrics — which can be fatal.
* Customizable monitoring views that fully unlock the value of the data. This is operations-leaning. We want to maximize the value of the runtime metrics from production — to see which systems make money while using the fewest resources, and which systems burn money while consuming the most.
* Empower developers and keep the system scalable.

![](The-Road-Of-Standardization-Construction-Of-VIPS-Operation-And-Maintenance/1526623366408-20a56b8f-2bd4-434d-93b3-a82644a01a97-image-resized.png) 

None of the above is satisfied by Zabbix.

Breakdown of monitoring standardization goals:
* The first layer is the event source, i.e., CMDB application information standardization.
* The second layer is monitoring module standardization. The expert team is responsible for designing monitoring and setting thresholds for technical components, with monitoring templates placed under unified version control.
* The third layer is alerting rule standardization. The monitoring system and the alerting system are decoupled, each with its own responsibility. Alerts are differentiated by device tier, application tier, and severity.

The unified source for alert recipients is the CMDB.

![](The-Road-Of-Standardization-Construction-Of-VIPS-Operation-And-Maintenance/1526623412346-0e342a3b-3734-4ab5-b1db-66b405ad9fb3-image-resized.png) 

We built our own product, VIPFalcon, based on secondary development of open-falcon. It currently monitors around 25,000 nodes with more than 5 million metrics, and uses data stream computing to re-aggregate the collected data. This data lands in Hive for data analysis, fully unlocking the value of the data.

![](The-Road-Of-Standardization-Construction-Of-VIPS-Operation-And-Maintenance/1526623459856-530145b7-8c67-4d75-b36e-3ca76aa3a4d3-image-resized.png) 

After monitoring standardization, comparing VIPFalcon with Zabbix: from a single place you can see all the infrastructure monitoring information of the entire company, without needing multiple entry points. In terms of collection extensibility, VIPFalcon is plugin-based, oriented around the HostGroup dimension, and maintained via Git. In terms of management, integration with the operations ecosystem, and programming language support, VIPFalcon is well ahead of Zabbix.

![](The-Road-Of-Standardization-Construction-Of-VIPS-Operation-And-Maintenance/1526623492880-06a3fdfd-86d0-4dd6-8de2-aa2a6f4ca7a4-image-resized.png) 

As a result of this work, internally we achieved quality improvements, higher work efficiency, lower maintenance costs, better user experience, and better risk control.

Outwardly, we exported value by opening up the operations ecosystem, empowering developers, and helping them think about resource usage from their perspective.

Fine-grained operations and business cost accounting are an output of the value of the data.

![](The-Road-Of-Standardization-Construction-Of-VIPS-Operation-And-Maintenance/1526623668710-5d19c184-cdda-4ee0-a774-2b657ca3d9eb-image-resized.png) 

### Change Standardization

Changes are tightly coupled with everyone in operations — every "blame" that operations takes is related to this area. The key is two ideas:
* First, we proposed a risk matrix. We built an SDK. To describe the matrix in one sentence: it splits the original change risk into two dimensions — the object and the technical risk. Briefly, the object refers to a business system: it may be a core system, an important system, or an unimportant one. We can profile it and assign a score. Combined together, these give each change a precise score.
* A standard change template library means that changes go through expert-designed templates for every change to each component. Changes must be based on the experts' templates; you cannot improvise a change plan however you like.

![](The-Road-Of-Standardization-Construction-Of-VIPS-Operation-And-Maintenance/1526623711827-15fb2866-1d6e-40bb-bce6-abc617fb9639-image-resized.png) 

After this work was solidified, we turned all kinds of changes into individual APPs, so that on the change platform you can supply parameters and execute a change with one click.

## Connecting the Ecosystem and Empowering Everyone

What do we mean by ecosystem? We have built many automation systems. If each one still requires a person to perform a task — for example, making a change still requires someone to click buttons — we are still in the early stage of automation. The ecosystem we want to build is one where systems drive systems; all systems interact through API calls, and humans are involved as little as possible.

### CMDB Replaces Process

To connect the operations ecosystem, the original approach was process-centric. Most people dislike processes; no one really loves them. Later, we shifted to an approach centered on operations processes + the CMDB. A process is a rather dogmatic thing. For example, a process might tell me that I can do change A at 3 PM this afternoon — but can I really? It does not weigh the context, the object, or the timing. If the object has a problem at 3 PM, the process should not proceed, but the process naively tells you to go ahead.

![](The-Road-Of-Standardization-Construction-Of-VIPS-Operation-And-Maintenance/1526623803451-9dc28059-81c2-40a6-9b8d-12a31d720abd-image-resized.png) 

This is Vipshop's specific business. In the middle are process control and a CMDB-centric approach, and from top to bottom are the operations-related deliverables — from components, to deployment standards, to all kinds of self-service platforms. Once the processes are connected, monitoring can fire alerts, and we want monitoring to drive self-healing. For example, when a disk alert reaches 90%, an alert is generated, which invokes a disk-cleanup APP on the automation platform, and that APP executes the cleanup. After cleanup, it tells the monitoring system to send a text message: a 90% alert was detected and the cleanup has been executed. This is a fairly ideal working state, and it has already been realized.

![](The-Road-Of-Standardization-Construction-Of-VIPS-Operation-And-Maintenance/1526623850017-9060d80e-f9aa-466b-b0c6-6ae9fda6218b-image-resized.png) 

### Approach to Connecting Changes

When it comes to ecosystem connectivity, our initial thinking was rather grand. We originally wanted to design a perfect process that would completely connect everything. We later realized that was naive; it is very hard to have a perfect plan that connects all the operations-related tooling. So our approach shifted to connecting them one by one. For example, to connect systems A, B, C, and D: first connect A-B, then connect C-D. A few issues with changes are ones that everyone encounters (see the slides). We built a very good and very flexible automated change system, but this is a double-edged sword: DevOps changes can go out of control and change processes may not be followed.

![](The-Road-Of-Standardization-Construction-Of-VIPS-Operation-And-Maintenance/1526623909934-7bea1880-1054-4e9d-8e18-ca53600fa7ab-image-resized.png) 
Design principles for connecting changes:
* First, the process system provides an SDK to all automation tools, offering change control, change collection, and SDK self-governance capabilities.
* Second, the automation capability platform sorts out the change risk matrix, integrates the SDK, and reports the change risk matrix.
* Third, the process system provides an SDK to all automation tools, offering change control, change collection, and SDK self-governance capabilities.

![](The-Road-Of-Standardization-Construction-Of-VIPS-Operation-And-Maintenance/1526623944201-368e7f65-28d4-4bf6-80dd-fed7fdffc643-image-resized.png) 

This is one of our systems (see diagram below). In the middle is a standard template library where the technical expert team designed a set of standard changes. The SDK is linked on the left side to the load balancing management platform, the scheduled-task management platform, the cloud platform, the operations platform, and others — connecting these platforms to empower developers.

When a developer wants to make a simple configuration change on a business line, the old way was for the developer to find operations and ask for the change. Operations would say "wait," submit a change process, and once the process was rolling, find that the boss wasn't around and the boss's approval was needed. If everything went smoothly, this took the better part of a day.

Now the developer submits a request on the tools platform, and if central control considers the risk very low, it goes through directly — the developer's task is done.

The benefit is that standard changes are solidified, processes are simplified, change risk is effectively controlled, and developers are empowered.

![](The-Road-Of-Standardization-Construction-Of-VIPS-Operation-And-Maintenance/1526623980216-a03adf71-a356-4373-b78f-75e5ea381397-image-resized.png) 

### Monitoring Connectivity Enables Automation

Our idea is that across the entire server lifecycle — initialization, deployment, running, pausing, and service decommissioning — none of the monitoring setup requires manual intervention. All entry points are connected between the monitoring system and the CMDB. There is only one source of information: the CMDB. When the CMDB detects a change in some piece of information, it writes to a message queue; the monitoring system then consumes that queue and executes the corresponding monitoring mount or unmount flow.

![](The-Road-Of-Standardization-Construction-Of-VIPS-Operation-And-Maintenance/1526624031971-3cd2945b-633e-45a8-95a3-7dd1f0331ddc-image-resized.png) 

This is the concrete implementation (see diagram below). In the middle is still the monitoring system, and on the left are alerting and data aggregation.

![](The-Road-Of-Standardization-Construction-Of-VIPS-Operation-And-Maintenance/1526624050910-6b4e1f4a-83ac-4952-bb2b-9162ab111592-image-resized.png) 

After the previous standardization and ecosystem connectivity, the gains are twofold:
* First, operations finally has comprehensive, centralized data. Monitoring data is captured, all production change data is in hand, and runtime data all settles into a unified pool. Why is this important? If you want to do AIOps, everything is empty talk without data. By connecting the ecosystem this way, data is deposited in one place, laying the groundwork for future intelligent operations.
* Second, through these efforts, we struck a balance between efficiency and process — ensuring efficiency while keeping risk under control.

![](The-Road-Of-Standardization-Construction-Of-VIPS-Operation-And-Maintenance/1526624081158-be8607c6-40a8-4441-8a7d-d8d7ccb0f857-image-resized.png) 

### Looking Back

Returning to the three questions posed at the beginning, there are clear gains across quality, efficiency, and cost.

* On quality, we transformed from manual labor into platform builders, taking the first step from automation toward intelligence.
* On efficiency, we lowered the barrier to entry, improved efficiency, and enforced process control.
* On cost, after aggregating the data, we profiled each application and gained data-backed insight into resource utilization.

![](The-Road-Of-Standardization-Construction-Of-VIPS-Operation-And-Maintenance/1526624130023-948f7fee-19ed-4c58-a632-4bcda7f7020b-image-resized.png) 

## A Few Reflections
* **Only on the soil of standardization can automation and intelligence take root and grow.**
* **Standardization requires strong leadership and a clear methodology.**
* **DevOps construction needs a business perspective. We don't do technology for technology's sake; we build around cost, quality, and efficiency, and the vision for platform building should aim higher.**
* **A single branch of service must be consolidated into a combined-arms force to deliver real combat power. Climbing one more rung up the food chain depends on the ability to integrate.**
