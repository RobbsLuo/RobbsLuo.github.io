---
title: Principles, Patterns, and Anti-Patterns of Container-Based Application Design
date: 2018-05-14 10:24:02
tags:
 - Container
 - Architect
categories:
 - Architect
lang: en
description: In the new container context, corresponding principles and patterns help us better build "cloud-native" applications. As we will see, these principles and patterns are not a wholesale overturning of previous patterns; rather, they are evolutionary versions adapted to a new environment.
---

The widespread adoption of containers and container orchestration (Kubernetes) makes it easy to build microservice-based "cloud-native" applications. Containers have become the new unit of programming in the cloud era, analogous to the `object` in object-oriented concepts, the `component` in J2EE, or the `function` in functional programming.

In the object-oriented era, there were many well-known design principles, patterns, and anti-patterns, for example:

* [SOLID][SOLID] (**Single Responsibility, Open-Closed, Liskov Substitution, Interface Segregation, and Dependency Inversion**)
* [Design Patterns: Elements of Reusable Object-Oriented Software](https://zh.wikipedia.org/wiki/%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F%EF%BC%9A%E5%8F%AF%E5%A4%8D%E7%94%A8%E9%9D%A2%E5%90%91%E5%AF%B9%E8%B1%A1%E8%BD%AF%E4%BB%B6%E7%9A%84%E5%9F%BA%E7%A1%80)

* [Anti-Pattern](https://zh.wikipedia.org/wiki/%E5%8F%8D%E9%9D%A2%E6%A8%A1%E5%BC%8F)

In the new container context, the corresponding principles and patterns help us build better "cloud-native" applications. As we will see, these principles and patterns are not a wholesale rejection of previous patterns; rather, they are evolutionary versions adapted to the new environment.

## Principles

#### Single Concern Principle (SCP)
Corresponding to the single responsibility of OO, each container should provide a single concern and focus on doing one thing well. A single concern makes a container easier to reuse. Typically a container corresponds to one process, and that process focuses on doing one thing well.
![SINGLE CONCERN PRINCIPLE (SCP)](Container-Application-Design/1526261691982-7ead3978-fe73-423f-a675-90309a8bc859-image.png) 

#### High Observability Principle (HOP)
Containers, like objects, should be well-encapsulated black boxes. However, in a cloud environment, this black box should expose good observability interfaces so that it can be monitored and managed appropriately in the cloud. In this way, the entire application can provide consistent lifecycle management.
![HIGH OBSERVABILITY PRINCIPLE (HOP)](Container-Application-Design/1526261707323-33e04e90-0449-4fea-8846-49a7a3cf6915-image-resized.png) 

Observability includes:
* Providing health checks, or heartbeats
* Providing status
* Outputting logs to standard output (STDOUT) and standard error (STDERR)
* etc.

#### Life-Cycle Conformance Principle (LCP)
The life-cycle conformance principle means that containers should interact with the platform to handle the corresponding lifecycle changes.
![LIFE-CYCLE CONFORMANCE PRINCIPLE (LCP)](Container-Application-Design/1526261801128-be721a67-971b-4f35-960e-0ce907328f73-image-resized.png) 

* Capture and respond to the Terminate (SIGTERM) signal to gracefully terminate the service process as quickly as possible, in order to avoid being forcibly killed by the kill (SIGKILL) signal. For example, the following NodeJS code.
```js
process.on('SIGTERM', function () {
  console.log("Received SIGTERM. Exiting.")
  server.close(function () {
    process.exit(0);
  });
});
```
* Return an exit code
```js
process.exit(0);
```

#### Image Immutability Principle (IIP)
At runtime, configurations may differ, but the image should be immutable.
![IMAGE IMMUTABILITY PRINCIPLE (IIP)](Container-Application-Design/1526261901806-402b717e-5379-476a-ab90-d419ff516bc8-image.png) 
We can think of an image as a class and a container as an object instance: the class is immutable, while the container is an instance of the image with different configuration parameters.

#### Process Disposability Principle (PDP)
In a cloud environment, we should assume that all containers are ephemeral and may be replaced by other container instances at any time.
![PROCESS DISPOSABILITY PRINCIPLE (PDP)](Container-Application-Design/1526261921352-50cf453f-a1eb-4d16-bde3-c47f7ecb5c96-image.png) 
This means that a container's state should be stored outside the container, and containers should start and stop as quickly as possible. Generally, the smaller the container, the easier this is to achieve.

#### Self-Containment Principle (S-CP)
A container should include all of its dependencies at build time; in other words, a container should not have any external dependencies at runtime.
![SELF-CONTAINMENT PRINCIPLE (S-CP)](Container-Application-Design/1526261951920-41f3d5f6-38ef-4619-91a1-1f802c90696a-image.png) 

#### Runtime Confinement Principle (RCP)
A best practice for containers is to declare the resource configuration requirements at runtime, such as how much memory, CPU, and so on. Doing so enables the container orchestrator to schedule and manage resources more effectively.
![RUNTIME CONFINEMENT PRINCIPLE (RCP)](Container-Application-Design/1526261967663-aaa59614-51c8-43cc-bde6-237e644fbbb2-image.png) 

## Patterns
Many container application patterns relate to the concept of a Pod. A Pod is a concept introduced by Kubernetes to manage containers effectively; it is a collection of containers, which we can think of as a "super-container" (a term I just made up). Containers within a Pod behave as if they are running on the same machine: they share the localhost address, can communicate locally, share volumes, and so on.
![](Container-Application-Design/1526261996216-83455d4d-0512-4dbb-877e-1cd6e97e76ed-image.png) 

Kubernetes is like an OS in the cloud, providing best practices for building cloud-native applications with containers. Let's look at some of the common patterns.

#### Sidecar
![](Container-Application-Design/1526262020618-7b150d6f-b6a3-4791-8212-38551258c423-image.png) 
The Sidecar is the most common pattern. Within the same Pod, we separate different responsibilities into different containers that together provide a complete feature to the outside world.
![](Container-Application-Design/1526262054188-bc378617-e76c-4498-9c0b-82f7704e6496-image-resized.png) 

There are many such examples, for instance:
* The Node backend and the Redis cache in the figure above
* A web server and a log-collection service
* A web server and a service responsible for collecting server performance metrics

This is somewhat analogous to the object-oriented [Composite pattern](https://en.wikipedia.org/wiki/Composite_pattern), and there are many benefits:
![Composite Pattern Implementation - UML Class Diagram](Container-Application-Design/1526262095283-87098624-d8fc-4286-a2af-ac94f4b0f47b-image.png) 

* Apply the Single Concern Principle: each container focuses on doing one thing well.
* Isolation: containers do not compete with each other for resources. When a secondary function (such as log collection or caching) fails or crashes, the impact on the primary function is minimized.
* Each container can be managed independently throughout its lifecycle.
* Each container can scale elastically and independently.
* Any one container can be easily replaced.

#### Ambassador (Proxy) Container
![Proxy Pattern Implementation - UML Class Diagram](Container-Application-Design/1526262238219-a6221168-99d3-4f03-bfc6-ca2fad38c325-image.png) 
Similar to the object-oriented [Proxy pattern](https://en.wikipedia.org/wiki/Proxy_pattern), this uses a container within the Pod to provide the external access connection. As shown in the figure below, the Node backend always communicates with the outside world through the Service Discovery container.
![](Container-Application-Design/1526262264669-62a724a4-88d8-4511-9120-328cd7a4af15-image-resized.png) 

In this way, the developers of the Node module only need to assume that all communication comes from the local machine, while the complexity of communication is delegated to the ambassador container, which handles concerns such as load balancing, security, request filtering, and, when necessary, terminating communication.

#### Adapter Container
![Adapter  Pattern Implementation - UML Class Diagram](Container-Application-Design/1526262303219-9324cca1-515f-4424-b9b8-c19b8814b9c3-image.png) 

People often confuse the object-oriented Proxy pattern, Bridge pattern, and [Adapter pattern](https://en.wikipedia.org/wiki/Adapter_pattern), because the UML diagrams look largely the same. It seems like they are just different names for the same thing. This is indeed the case—just as almost all OO patterns are derivatives of the Composite pattern, all container patterns are derivatives of the Sidecar pattern.

In the example below, if the name "Logging Adapter" did not mention "Adapter," we would not consider it an adapter pattern.
![](Container-Application-Design/1526262348948-fce2d10f-320e-4d52-89eb-931bf394454c-image-resized.png) 

In fact, the adapter pattern is concerned with how to uniformly expose the functionality of different containers inside the Pod through an adapter. In the figure above, it becomes clearer if we add one more container that also writes logs to the volume. The Logging Adapter adapts the logs produced by different containers through different interfaces and provides a unified access interface.

#### Container Chain
![hain of Responsability Implementation - UML Class Diagram](Container-Application-Design/1526262383289-ee67fbb1-8251-4af3-bb61-f28db3e9bbc0-image.png) 

Similar to the OO [Chain of Responsibility pattern](http://www.oodesign.com/chain-of-responsibility-pattern.html), chaining together containers responsible for different functions in dependency order is also a common pattern.
![](Container-Application-Design/1526262452948-23ea67c9-470f-4f7b-8a03-c76307b4415f-image-resized.png) 

#### Ready Pod
![](Container-Application-Design/1526262469915-5b17a04f-67ce-46de-967f-8d8ef906ac68-image.png) 

Typically, a container running as a service has a startup process during which the service is unavailable. Kubernetes provides a [Readiness](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/) probe feature.
```yaml
readinessProbe:
  httpGet:
    path: /
    port: 5000
  timeoutSeconds: 1
  periodSeconds: 5
```
Compared with the other patterns, this is more of a Kubernetes best practice.

## Anti-Patterns
#### Mixing the Build Environment with the Runtime Environment
The image used for the production runtime environment should be as small as possible, and should avoid including leftovers from the build time.

Consider the following Dockerfile example:
```bash
FROM ubuntu:14.04

RUN apt-get update
RUN apt-get install gcc
RUN gcc hello.c -o /hello
```
The resulting image contains many things that are neither needed nor appropriate for a production environment, such as gcc and the source code hello.c. This is both insecure (directly exposing the source code) and incurs a performance overhead (an oversized image leads to slower loading).

The [multi-stage builds](https://docs.docker.com/develop/develop-images/multistage-build/) introduced in Docker 17.05 can solve this problem.

#### Using Pods Directly
Avoid using Pods directly; manage Pods with a Deployment. Using a Deployment makes it easy to scale and manage Pods.

#### Using the latest Tag
The latest tag is meant to mark the most recent stable version. However, when creating containers, you should avoid using the latest tag in production environments as much as possible, even when the imagePullPolicy option is set to Always.

#### Fast-Failing Jobs
A Job is a Kubernetes container that runs only once, the opposite of a service. Avoid fast failure.
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: bad
spec:
  template:
    metadata:
      name: bad
    spec:
      restartPolicy: Never
      containers:
      - name: box
        image: busybox
        command: ["/bin/sh", "-c", "exit 1"]
```
If you try to create the above Job in your cluster, you might encounter the following status.
```bash
$ kubectl describe jobs 
Name:   bad
Namespace:  default
Image(s): busybox
Selector: controller-uid=18a6678e-11d1-11e7-8169-525400c83acf
Parallelism:  1
Completions:  1
Start Time: Sat, 25 Mar 2017 20:05:41 -0700
Labels:   controller-uid=18a6678e-11d1-11e7-8169-525400c83acf
    job-name=bad
Pods Statuses:  1 Running / 0 Succeeded / 24 Failed
No volumes.
Events:
  FirstSeen LastSeen  Count From      SubObjectPath Type    Reason      Message
  --------- --------  ----- ----      ------------- --------  ------      -------
  1m    1m    1 {job-controller }     Normal    SuccessfulCreate  Created pod: bad-fws8g
  1m    1m    1 {job-controller }     Normal    SuccessfulCreate  Created pod: bad-321pk
  1m    1m    1 {job-controller }     Normal    SuccessfulCreate  Created pod: bad-2pxq1
  1m    1m    1 {job-controller }     Normal    SuccessfulCreate  Created pod: bad-kl2tj
  1m    1m    1 {job-controller }     Normal    SuccessfulCreate  Created pod: bad-wfw8q
  1m    1m    1 {job-controller }     Normal    SuccessfulCreate  Created pod: bad-lz0hq
  1m    1m    1 {job-controller }     Normal    SuccessfulCreate  Created pod: bad-0dck0
  1m    1m    1 {job-controller }     Normal    SuccessfulCreate  Created pod: bad-0lm8k
  1m    1m    1 {job-controller }     Normal    SuccessfulCreate  Created pod: bad-q6ctf
  1m    1s    16  {job-controller }     Normal    SuccessfulCreate  (events with common reason combined)
```
Because the Job fails fast, Kubernetes considers the Job to have failed to start successfully and tries to create new containers to recover from this failure, causing the cluster to create a large number of containers in a short period of time, which can consume a significant amount of computing resources.

Use `.spec.activeDeadlineSeconds` in the Spec to avoid this problem. This parameter defines how long to wait before retrying a failed Job.

---
> Source: https://my.oschina.net/taogang/blog/1809904






[SOLID]: https://zh.wikipedia.org/wiki/SOLID_(%E9%9D%A2%E5%90%91%E5%AF%B9%E8%B1%A1%E8%AE%BE%E8%AE%A1)	"SOLID"
