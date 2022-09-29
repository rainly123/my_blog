---
title: "kubernetes（一、核心对象）"
date: 2022-09-24T17:04:00+08:00
draft: true
toc: true

---

## 前言

至于什么是kebernetes我像不必过多赘述，docker诞生依赖，引发了容器技术的火爆，早在kebernetes诞生以前，docker公司就自研了docker swarm作为容器编排技术，它的优势就是和docker的生态无缝集成，除此之外，再无更多的核心竞争力。为了对抗docker在容器技术一家独大的情况，由google牵头，redhat等拥有深厚技术底蕴的巨头跟随，在google borg技术理念的基础上，最终打造出了容器编排技术中脱引而出的天之骄子kebernetes。

## kebernetes的本质

**容器的核心功能，就是修改和约束进程的动态表现，为其创造一个边界**，而对于像docker等大多数linux容器技术来说，**namespace技术就是用来修改进程视图的主要方法，而Cgroup技术则是提供约束的主要手段**,linux主要namespace类型：

1. pid namespace，隔离进程ID
2. user namespace 隔离用户和用户组
3. mount namespace 让被隔离进程只能看到当前namespace挂载点信息
4. network namespace 让隔离进程只能看到当前namespace网络配置
   那么cgroup主要约束哪些内容呢？
   **Cgroup的主要约束容器的资源访问上限，包括CPU，内存，网络，磁盘等**
   这就是linux容器技术实现的基本原理，所以我们在docker中看到的文件，设备，状态等等，在宿主机上完全看不到。所以回过头来我们再来聊一聊，容器是什么？说白了，容器就是一种特殊的进程而已。

## kebernetes的核心对象

kebernets是怎么做容器编排的呢，为什么在众多的容器技术之中能够脱引而出？我想这个和他的一个重要理念有关系——通过对象管理对象,在我们弄清楚kebernetes如何通过对象进行管理之前，我们先弄清楚几个对象的基本概念：

1. pod:pod是kebernetes的最小部署单元，一个pod中可以有多个容器，他们共享存储和网络。
2. service:pod对外的访问入口，通过lable selector选择一组pod提供服务。
3. deployment: 他是一个更高层次的API对象，官方建议用deployment进行replicatSets管理，pod rolling update
4. job：一次性任务
5. cronjob：定时任务
6. namespace：将对象逻辑上分配到不同的namespace，可以是不同的项目和用户分区等等。
7. configmap：是k8s原生配置中心，可以将配置以key-value的形式传递，通常用来保存不需要加密的配置信息
