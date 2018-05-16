---
title: 基于容器应用设计的原则，模式和反模式
date: 2018-05-14 10:24:02
tags:
 - Container
 - Architect
categories:
 - Architect
description: 在新的容器背景下，相应的原则和模式有助于帮助我们更好的构建“云原生”的应用。我们可以看到，这些原则和模式并非对之前模式的颠覆和推翻，更像是适应新环境的演进版本。
---

容器和容器编排（Kubernetes）的广泛使用，让我们可以轻松的构建基于微服务的“云原生”（Cloud Native）的应用。容器成为了云时代的新的编程单元，类似面向对象概念下的`对象`，J2EE中的`组件`或者函数式编程中的`函数`。

在面向对象时代，有许多著名的设计原则，模式和反模式等，例如：

* [SOLID][SOLID] (**单一功能、开闭原则、里氏替换、接口隔离以及依赖反转**）
* [Design Patterns: Elements of Reusable Object-Oriented Software](https://zh.wikipedia.org/wiki/%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F%EF%BC%9A%E5%8F%AF%E5%A4%8D%E7%94%A8%E9%9D%A2%E5%90%91%E5%AF%B9%E8%B1%A1%E8%BD%AF%E4%BB%B6%E7%9A%84%E5%9F%BA%E7%A1%80)

* [Anti-Pattern](https://zh.wikipedia.org/wiki/%E5%8F%8D%E9%9D%A2%E6%A8%A1%E5%BC%8F)

在新的容器背景下，相应的原则和模式有助于帮助我们更好的构建“云原生”的应用。我们可以看到，这些原则和模式并非对之前模式的颠覆和推翻，更像是适应新环境的演进版本。

## 原则

#### 单一职责原则 SINGLE CONCERN PRINCIPLE (SCP)
与OO的单一功能相对应，每一个容器应该提供单一的职责，只关注于做好一件事。单一职责使得容器更容易重用。通常容器对应于一个进程，而该进程专注于做好一件事。
![SINGLE CONCERN PRINCIPLE (SCP)](Container-Application-Design/1526261691982-7ead3978-fe73-423f-a675-90309a8bc859-image.png) 

#### 高可观测性原则 HIGH OBSERVABILITY PRINCIPLE (HOP)
容器像对象一样，应该是一个封装良好的黑盒子。但是在云的环境下，这个黑盒子应该提供良好的观测接口，使得其在云的环境下得到相应的监控和管理。这样，整个应用才能提供一致的生命周期的管理。
![HIGH OBSERVABILITY PRINCIPLE (HOP)](Container-Application-Design/1526261707323-33e04e90-0449-4fea-8846-49a7a3cf6915-image-resized.png) 

可观测性包含：
* 提供健康检查 Health Check，或者心跳
* 提供状态
* 把日志输出到标准输出(STDOUT)和标准出错(STDERR)
* 等等 

#### 生命周期确认原则 LIFE-CYCLE CONFORMANCE PRINCIPLE (LCP)
生命周期确认原则指的是容器应该提供和平台交互来处理相应的生命周期的变化。
![LIFE-CYCLE CONFORMANCE PRINCIPLE (LCP)](Container-Application-Design/1526261801128-be721a67-971b-4f35-960e-0ce907328f73-image-resized.png) 

* 捕获并响应Terminate (SIGTERM)信号，来尽快优雅的终止服务进程，以避免kill (SIGKILL)信号强行终止进程。例如一下的NodeJS代码。
```js
process.on('SIGTERM', function () {
  console.log("Received SIGTERM. Exiting.")
  server.close(function () {
    process.exit(0);
  });
});
```
* 返回退出码
```js
process.exit(0);
```

#### 镜像不可变原则 IMAGE IMMUTABILITY PRINCIPLE (IIP)
在运行时，配置可以不同，但是镜像应该是不可变的。
![IMAGE IMMUTABILITY PRINCIPLE (IIP)](Container-Application-Design/1526261901806-402b717e-5379-476a-ab90-d419ff516bc8-image.png) 
我们可以理解为镜像是个类，是容器是对象实例，类是不变的，而容器是拥有不同配置参数的镜像实例。

#### 进程用完既丢原则 PROCESS DISPOSABILITY PRINCIPLE (PDP)
在云环境下，我们应该假定所有的容器都是临时的，它随时有可能被其它的容器实例所替代。
![PROCESS DISPOSABILITY PRINCIPLE (PDP)](Container-Application-Design/1526261921352-50cf453f-a1eb-4d16-bde3-c47f7ecb5c96-image.png) 
这也就意味着需要把容器的状态保存在容器之外。并且尽可能快速的启动和终止容器。通常越小的容器就越容易实现这一点。

#### 自包含原则 SELF-CONTAINMENT PRINCIPLE (S-CP)
容器在构建的时候应该包含所有的依赖，也就是所说容器在运行时不应该有任何的外部依赖。
![SELF-CONTAINMENT PRINCIPLE (S-CP)](Container-Application-Design/1526261951920-41f3d5f6-38ef-4619-91a1-1f802c90696a-image.png) 

#### 限制运行资源原则 RUNTIME CONFINEMENT PRINCIPLE (RCP)
容器的最佳实践应该是在运行时指定容器对资源配置的需求。例如需要多少的内存，CPU等等。这样做可以使得容器编排能都更有效的调度和管理资源。
![RUNTIME CONFINEMENT PRINCIPLE (RCP)](Container-Application-Design/1526261967663-aaa59614-51c8-43cc-bde6-237e644fbbb2-image.png) 

## 模式
许多容器应用的模式和Pod的概念相关，Pod是Kubernetes为了有效的管理容器而提出的概念，它是容器的集合，我们可以理解为“超容器”（我随便发明的）。Pod包含的容器之间就好像运行在同一台机器上，这些容器共享Localhost主机地址，可以本机通信，共享卷等等。
![](Container-Application-Design/1526261996216-83455d4d-0512-4dbb-877e-1cd6e97e76ed-image.png) 

Kubernetes 类似云上OS，提供了用容器构建云原生应用的最佳实践。我们看看这些常见的模式都有什么。

#### 边车（侧斗）（Sidecar）
![](Container-Application-Design/1526262020618-7b150d6f-b6a3-4791-8212-38551258c423-image.png) 
Sidecar是最常见的模式，在同一个Pod中，我们需要把不同的责任分在不同的容器中，对外部提供一个完整的功能。
![](Container-Application-Design/1526262054188-bc378617-e76c-4498-9c0b-82f7704e6496-image-resized.png) 

这样的例子有很多，例如：
* 上图中的Node后端和提供缓存的Redis
* Web服务器和收集日志的服务
* Web服务器和负责监控服务器性能数据的服务

这样做有点类似面向对象的[组合模式](https://en.wikipedia.org/wiki/Composite_pattern)，好处有很多：
![Composite Pattern Implementation - UML Class Diagram](Container-Application-Design/1526262095283-87098624-d8fc-4286-a2af-ac94f4b0f47b-image.png) 

* 应用单一职责原则，每一个容器只负责专注做好一件事。
* 隔离，容器之间不会出现互相竞争资源，当一个次要功能（例如日志收集或者缓存）失效或者崩溃的时候，对主要功能的影响降至最小。
* 可以对每一个容器进行独立的生命周期管理
* 可以对每一个容器进行独立的弹性扩张
* 可以方便的替换其中一个容器

#### 代理（大使）容器
![Proxy Pattern Implementation - UML Class Diagram](Container-Application-Design/1526262238219-a6221168-99d3-4f03-bfc6-ca2fad38c325-image.png) 
类似于面向对象的[Proxy模式](https://en.wikipedia.org/wiki/Proxy_pattern)，利用Pod中一个容器提供对外的访问连接。如下图中Node后端总是通过Service Discovery容器来和外部进行通信。
![](Container-Application-Design/1526262264669-62a724a4-88d8-4511-9120-328cd7a4af15-image-resized.png) 

这样做，负责Node模块开发的只需要假定所有的通信都是来自于本机，而把通信的复杂性交给代理容器，去处理诸如负载均衡，安全，过滤请求，必要时中断通信等功能。

#### 适配器容器
![Adapter  Pattern Implementation - UML Class Diagram](Container-Application-Design/1526262303219-9324cca1-515f-4424-b9b8-c19b8814b9c3-image.png) 

大家常常会把面向对象的Proxy模式，Bridge模式和[Adapter模式](https://en.wikipedia.org/wiki/Adapter_pattern)搞混，因为单单从UML关系图上来看，它们都大同小异。似乎只是取了不同的名字。事实也确实如此，就像几乎所有的OO模式都是组合模式的衍生，所有容器模式都是边车模式的衍生。

在下图的例子中，如果Logging Adapter的名字不提及Adapter，我们不会认为这是个适配器模式。
![](Container-Application-Design/1526262348948-fce2d10f-320e-4d52-89eb-931bf394454c-image-resized.png) 

其实适配器模式关注的是如果把Pod内部的不同容器的功能通过适配器统一的暴漏出来。在上图中，如果我们再多加一个容器，它同时会向卷中写入日志的化，这样就更清楚了。Logging Adapter适配不同容器用不同的接口提供的日志，并提供统一的访问接口。

#### 容器链
![hain of Responsability Implementation - UML Class Diagram](Container-Application-Design/1526262383289-ee67fbb1-8251-4af3-bb61-f28db3e9bbc0-image.png) 

类似于OO的[责任链模式](http://www.oodesign.com/chain-of-responsibility-pattern.html)，把负责不同功能的容器按照依赖顺序链在一起，也是一种常见的模式。
![](Container-Application-Design/1526262452948-23ea67c9-470f-4f7b-8a03-c76307b4415f-image-resized.png) 

#### 准备就绪的Pod
![](Container-Application-Design/1526262469915-5b17a04f-67ce-46de-967f-8d8ef906ac68-image.png) 

通常作为服务的容器有一个启动的过程，在启动过程中，服务是不可用的。Kubernetes提供了[Readiness](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/)探测功能。
```yaml
readinessProbe:
  httpGet:
    path: /
    port: 5000
  timeoutSeconds: 1
  periodSeconds: 5
```
和其它模式相比，这个更像是一个使用Kubernetes的最佳实践。

## 反模式
#### 构建环境和运行环境混杂在一起
应该使得用于生产的运行环境的镜像尽可能的小，避免在运行环境的镜像中包含构建时的残留。

例如下面的Dockerfile例子：
```bash
FROM ubuntu:14.04

RUN apt-get update
RUN apt-get install gcc
RUN gcc hello.c -o /hello
```
在这个构建的镜像中，有很多不需要也不应该出现在生产环境中的东西，例如gcc，源代码hello.c。这样的结果既不安全（直接暴漏源代码），也会有性能开销（过大的镜像体积导致加载变慢）。

Docker17.05 以后提供的[multi-stage builds](https://docs.docker.com/develop/develop-images/multistage-build/)也可以解决这个问题。

#### 直接使用Pod
避免直接使用Pod，用Deployment来管理Pod。利用Deployment可以很方便的对Pod进行扩展和管理。

#### 使用latest标签
Latest标签用于标记最近的稳定版本，然而在创建容器时，尽可能避免在生产环境使用Latest标签。即使使用imagePullPolicy选项为alway。

#### 快速失败的任务
Job是Kubernetes提供的只运行一次的容器，和service正好相反。要避免快速失败
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
如果你尝试在你的cluster里面创建以上的Job，你可能会碰到如下的状态。
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
因为任务快速失败。Kubernetes认为任务没能成功启动，尝试创建新的容器以恢复这个失败，导致的Cluster会在短时间创建大量的容器，这样的结果可能会消耗大量的计算资源。

在Spec中使用.spec.activeDeadlineSeconds来避免这个问题。这个参数定了等待多长时间重试失败的Job。

---
> 来源: https://my.oschina.net/taogang/blog/1809904







[SOLID]: https://zh.wikipedia.org/wiki/SOLID_(%E9%9D%A2%E5%90%91%E5%AF%B9%E8%B1%A1%E8%AE%BE%E8%AE%A1)	"SOLID"