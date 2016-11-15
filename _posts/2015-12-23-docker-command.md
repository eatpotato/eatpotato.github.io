---
layout: post
title: Docker初探
date:       2015-12-23
author:     xue
tags:
    - docker
---

# Docker简单使用

&emsp;&emsp;Docker已经火了挺长一段时间了，自己虽然常常听到这么一个概念，但是一直都没有时间了解，刚好前段时间借了一本docker的书，也算是对Docker有了一个比较系统的认识。

&emsp;&emsp;Docker是基于Go语言实现的云开源项目，诞生于2013年初。

## Docker容器虚拟化的好处?

&emsp;&emsp;举个简单的应用场景的例子。假设用户试图基于最常见的LAMP(Linux+Apache+MySQL +PHP)组合来运维一个网站。按照传统的做法，首先，需要安装Apache、MySQL和PHP以及它们各自运行所以来的环境;之后分别对它们进行配置（包括创建合适的用户和配置参数等）;经过大量的操作后，还需要进行功能测试，看是否正常工作；如果不正常，则意味着更多的时间代价和不可控的风险。可以想象，如果再加上更多的应用，事情会变得更加难以处理。  
&emsp;&emsp;更可怕的是，一旦需要服务器迁移（例如从阿里云迁移到腾讯云），往往需要重新部署和调试。这些琐碎而无趣的“体力活”，极大地降低了工作效率。  
&emsp;&emsp;而Docker提供了一种更为聪明的方式，通过容器来打包应用，意味着迁移只需要在新的服务器上启动需要的容器就可以了。这无疑将节约大量的宝贵时间，并降低部署过程中出现问题的风险。  

&emsp;&emsp;具体说来,Docker在开发和运维过程中，具有如下几个方面的优势:  
&emsp;&emsp;&emsp;  - 更快速的交付和部署  
&emsp;&emsp;&emsp;  - 更高效的资源利用   
&emsp;&emsp;&emsp;  - 更轻松的迁移和扩展  
&emsp;&emsp;&emsp;  - 更简单的更新管理  

&emsp;&emsp;上面的话摘自书本《Docker技术入门与实践》中，下面说说我自己对docker的认识和体会。  
&emsp;&emsp;我在两个月前听了灵雀云创始人左玥在我们学校的关于docker的分享会，回答问题环节还得到了移动电源一枚，精神上和物质上都是受益匪浅:)。在这次分享会上，左玥对docker有一段很形象的描述(我也只是记得大概)，好比我们运输货物，我们可以选择飞机、轮船、货车作为我们的运输工具，只不过这些运输工具的运输能力可能有差别，但是当我们选择运输工具的时候会很麻烦，比方说运送水果的船上不能同时运输化学品，还有用大货轮只运几箱水果是不是很大材小用？那这些问题可以通过集装箱来解决，在一艘大船上，可以把货物板正的摆放起来，并且各种各样的货物被集装箱标准化了。集装箱与集装箱之间不会互相影响。那么我就不需要专门运送水果的船和专门运输化学品的船了，只要这些货物在集装箱在集装箱里装的好好的，我可以用任意交通工具把他们都运走。  
&emsp;&emsp;所以我理解的docker，是一种轻量级的虚拟化，可以实现虚拟机隔离应用环境的功能，你可以把它当作一个进程来看待，与整个操作系统的虚拟化相比，它对资源的需求低很多。再者，它很好的隔离了各个应用程序，避免了端口号冲突等问题的发生。最最后，它的移植性很好，由集中箱的概念可知，它的接口是统一的，这种兼容性可以让用户在不同平台之上轻松地迁移应用。  
&emsp;&emsp;下面列出一些我在使用过程中的Docker命令：

## Docker镜像

>\# 查看镜像信息  
>$ docker images  

>\# 获取镜像信息  
>$ docker pull imagename[:version]

>\# 删除镜像（可通过镜像的标签或镜像ID）  
>$ docker rmi ID or imagename:TAG

镜像常见的容器存在时，镜像文件默认是无法被删除的（可以使用-f强制删除，但不推荐）

镜像创建的三种方式：

* 基于已有镜像的容器创建  

* 基于本地模板导入

* 基于Dockerfile创建 

>\# 基于已有镜像的容器创建命令为：  
>$ docker commit [OPTIONS] CONTAINER [REPOSITORY[:TAG]]  
>\# [OPTIONS]为：  
>1. -a,--author=""作者信息。  
>2. -m,--message=""提交信息。  
>3. -p,--pause=true提交时暂停容器运行。  
>\# Ex:sudo docker commit -m "Add a new file" -a "Xue" a925cb403f0 test  
>\# 顺利的话会返回新创建的镜像的ID信息

>\# 上传镜像到仓库（默认为DockerHub官方仓库）  
>$ docker push NAME[:TAG]

## Docker容器

>\# 查看所有的容器   
>$ docker ps -a  

>\# 新建容器   
>$ docker create -it NAME[:TAG]  
>\# 使用docker create命令创建的容器处于停止状态，可以使用docker start命令来启动它

>\# 新建并启动容器   
>$ docker run NAME[:TAG] /bin/bash

>\# 查看种植状态的容器的ID信息  
>$ sudo docker ps -a -q    

>\# 终止容器  
>$ docker stop ID

>\# 进入容器  
>$ docker exec -ti ID /bin/bash

>\# 删除容器  
>$ docker rm [OPTIONS] CONTAINER ID

>\# 导出容器  
>$ docker export ID >filename  
>\# EX:sudo docker export ce5 >test_for_run.tar

