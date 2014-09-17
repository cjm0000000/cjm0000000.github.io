---
layout: post
title: "run docker on centos7"
description: ""
category: Docker
tags: [Virtualization]
---
{% include JB/setup %}

最近[Docker](https://www.docker.com/)貌似很火，抱着尝鲜的心态，我在[CentOS 7](http://mirrors.aliyun.com/centos/7/isos/x86_64/)下部署了Docker，在此记录下全过程。

## 安装系统
由于本人对CentOS 6系统稍微熟悉一些，就打算在CentOS下运行Docker，但是想运行最新版的Docker，需要3.8或更高的内核，官方原文如下：

> If you need the latest version, you can always use the latest binary which works on kernel 3.8 and above.

另外还有一个限制：只能运行在64位系统下。

> Please note that due to the current Docker limitations, Docker is able to run only on the __64 bit__ architecture.


如果想运行最新版的Docker，只能选CentOS 7了，所以这里选择CentOS 7的x86_64版本。

### 安装CentOS 7
从阿里云镜像下载CentOS7的最小镜像。开始在VirtualBox安装，大致安装过程如下：

点击"Test this media & Install CentOS 7"菜单
<img class="imgaligncenter" src="/images/run-docker-on-centos7-step1.jpg" />
<center>图 1</center>
----------
选择语言，这里选择英语。
<img class="imgaligncenter" src="/images/run-docker-on-centos7-step2.jpg" />
<center>图 2</center>
----------
根据需要设置时区、语言、键盘等，System下的INSTALLTION DESTINATION必须点进去设置，如图4；
网络也需要设置，这里配置为桥接网卡，具体配置见图5；
<img class="imgaligncenter" src="/images/run-docker-on-centos7-step3.jpg" />
<center>图 3</center>
----------
设置硬盘
<img class="imgaligncenter" src="/images/run-docker-on-centos7-step4.jpg" />
<center>图 4</center>
----------
配置网络
<img class="imgaligncenter" src="/images/run-docker-on-centos7-step5.jpg" />
<center>图 5</center>
----------
这里需要设置root用户的密码，如图7；创建用户可选。
<img class="imgaligncenter" src="/images/run-docker-on-centos7-step6.jpg" />
<center>图 6</center>
----------
<img class="imgaligncenter" src="/images/run-docker-on-centos7-step7.jpg" />

等到安装完成，点击reboot按钮重启。接下去通过SSH连接上去创建用户，配置环境。

### 配置CentOS 7
#### 创建用户



## 安装Docker


## 参考资料

- [CentOS - Docker Documentation](https://docs.docker.com/installation/centos/)


