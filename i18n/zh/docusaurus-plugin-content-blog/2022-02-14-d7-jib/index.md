---
title: P2P加速Docker镜像分发(阿里Dragonfly2+google jib)(基于d7y 2.0.2)
author: anjia0532
author_title: 攻城狮
author_url: https://github.com/anjia0532
author_image_url: https://avatars.githubusercontent.com/u/15098916?s=400&u=23a4b699baa0ed924cf1db40b9edb614d0263621&v=4
tags: [docker, dragonfly, dragonfly2, containerd]
description: 本文介绍如何利用google jib 和 Dragonfly2 实现镜像加速构建，加速传输，加速拉起等功能
image: https://avatars.githubusercontent.com/u/15098916?s=400&u=23a4b699baa0ed924cf1db40b9edb614d0263621&v=4
hide_table_of_contents: false
---

简单介绍下

- [google jib](https://github.com/GoogleContainerTools/jib) ：支持gradle和maven，用于构建java应用镜像时，将基础镜像(jdk)，
- 依赖(jar lib),资源文件(resources),class文件等进行分层(layer),这样依赖，在拉取和推送镜像时，起到加速和节省带宽的目的。
- [Dragonfly2](https://github.com/dragonflyoss/Dragonfly2)：是阿里开源的一款基于P2P协议的，镜像和文件分发加速工具，
- 与dragonfly1相比，dragonfly2用golang重构了，运行时占用资源更少。理论上可以基于dragonfly做一个局域网CDN，及局域网镜像加速器，
- 文件通过dragonfly下载后，缓存到局域网内，再次请求时，如果局域网节点内有，且未过期，则通过p2p协议从局域网内拉取，防止占用公网带宽及某个节点过
载被打死的情况。

简单总结下，jib解决的是java应用动不动100M+甚至1G+的情况（变成了80M JDK(基本不变)+200M jar(基本不变)+300M resource(基本不变)+
1M class（每次发版会变）），
而dragonfly2解决的是节省公网带宽，减少内部registry节点过热的情况，加起来就是，容器push&pull的过程更快了

其实三年前写过阿里Dragonfly+google jib的文章，但是时间比较久远，有些内容已经过时,所以准备重新整理下。
之前文章如下：

- [加速和简化构建Docker(基于Google jib)](https://anjia0532.github.io/2019/02/08/google-jib/)
- [046-解决google jib多任务问题](https://anjia0532.github.io/2019/09/22/google-jib-alpine-tini/)
- [012-P2P加速Docker镜像分发(阿里Dragonfly)](https://anjia0532.github.io/2019/03/25/dragonfly/)
- [013-阿里Dragonfly体验之私有registry下载](https://anjia0532.github.io/2019/03/30/d7y-private-registry/)

## Google Jib

jib支持 [maven: jib-maven-plugin](https://github.com/GoogleContainerTools/jib/tree/master/jib-maven-plugin)
和[gradle: jib-gradle-plugin](https://github.com/GoogleContainerTools/jib/tree/master/jib-gradle-plugin) ,
以及常见的 [多模块场景](https://github.com/GoogleContainerTools/jib/tree/master/examples/multi-module)

翻了翻新版的  [jib-maven-plugin](https://github.com/GoogleContainerTools/jib/tree/master/jib-maven-plugin) 文档，
主要部分跟我之前的 [加速和简化构建Docker(基于Google jib)](https://anjia0532.github.io/2019/02/08/google-jib/) 差不多，不再`CV`了

但是加了不少新特性，比如

- [自定义Entrypoint](https://github.com/GoogleContainerTools/jib/tree/master/jib-maven-plugin#custom-container-entrypoint)
- [支持镜像加速器等jib全局配置](https://github.com/GoogleContainerTools/jib/tree/master/jib-maven-plugin#global-jib-configuration)
- [支持自定义CPU架构（architecture）和操作系统（os）](https://github.com/GoogleContainerTools/jib/tree/master/jib-maven-plugin#platform-object)
- [支持更多选项的基础镜像的设置](https://github.com/GoogleContainerTools/jib/tree/master/jib-maven-plugin#platform-object)

特别的说一下，jib的很多配置，除了改pom.xml外，还支持无侵入的通过命令行指定，并且，命令行传入优先级更高，
比如 `mvn compile com.google.cloud.tools:jib-maven-plugin:3.2.0:build -Dimage=<MY IMAGE>`中的`-Dimage=`
对应的是pom.xml里的`<configuration><to><image></image></to></configuration>`

## K8S(kubernetes)以rke2为例

安装k8s集群，如果只是简单测试一下，可以用 [k3s](https://docs.rancher.cn/docs/k3s/quick-start/_index/)
或者 [本地分布式开发环境搭建使用Vagrant和Virtualbox](https://jimmysong.io/kubernetes-handbook/develop/using-vagrant-and-virtualbox-for-development.html)
或者 [rke2](https://docs.rancher.cn/rke2/)
(如果会用ansible也可以用 [ansible playbook rke2](https://github.com/lablabs/ansible-role-rke2) )

如果想可视化看下 k8s 集群，可以用[kube-explorer](https://github.com/cnrancher/kube-explorer),
[k9s](https://github.com/derailed/k9s) 也可以用[rancher2.6](https://rancher.com/docs/rancher/v2.6/en/)(注意如果用rancher的话
，注意rke2安装的版本,以[最新稳定版本2.6.3](https://github.com/rancher/rancher/releases/tag/v2.6.3-patch1)为例，默认支持的是
k8s v1.21.7,也就是得用[rke2 v1.21.7+rke2r2](https://github.com/rancher/rke2/releases/tag/v1.21.7%2Brke2r2))

## Dragonfly2

别看官网文档，已经年久失修了，直接看 [github文档](https://github.com/dragonflyoss/Dragonfly2/tree/main/docs/zh-CN) ，
更保险点是看github代码，切记，切记，切记。

### helm安装

官方文档 [https://github.com/dragonflyoss/helm-charts/blob/main/charts/dragonfly/README.md](https://github.com/dragonflyoss/helm-charts/blob/main/charts/dragonfly/README.md)

考虑到国内特殊国情，可能会访问github失败，可以用 [https://gitee.com/projects/import/url](https://gitee.com/projects/import/url)
中转(也可以用我的 [https://gitee.com/anjia/dragonflyoss-helm-charts](https://gitee.com/anjia/dragonflyoss-helm-charts))

```bash
git clone https://gitee.com/anjia/dragonflyoss-helm-charts.git

cd ./dragonflyoss-helm-charts/charts/dragonfly

helm dependency update

helm install --create-namespace --namespace dragonfly-system dragonfly . -f values.yml
```

如果要自定义参数，通过 -f values.yml 来指定，如果默认则移除 `-f values.yml`,
支持的配置有 [https://github.com/dragonflyoss/helm-charts/tree/main/charts/dragonfly#values](https://github.com/dragonflyoss/helm-charts/tree/main/charts/dragonfly#values)

### 注意点

1. dragonfly的helm支持docker和containerd两种引擎，官方推荐使用containerd(因为支持fallback，docker不支持)，如果是加速多镜像库官方推荐使
用 [containerd1.5.x+](https://github.com/dragonflyoss/Dragonfly2/blob/main/docs/zh-CN/runtime-integration/containerd/mirror.md#%E9%80%89%E9%A1%B9-2-%E5%A4%9A%E9%95%9C%E5%83%8F%E4%BB%93%E5%BA%93),
因为 `/etc/containerd/config.toml` 是version2版本，支持多个注册中心的加速，否则只支持一个，当然也有办法解决，后边再说。
2. rke2 是通过 `/etc/rancher/rke2/registries.yaml` 来生成 `/var/lib/rancher/rke2/agent/etc/containerd/config.toml`的，
而目前版本的helm不支持自定义`/etc/containerd/config.toml`就会导致daemon启动失败， 提了个pr还没过
[https://github.com/dragonflyoss/helm-charts/pull/51](https://github.com/dragonflyoss/helm-charts/pull/51)，可以先手动修改
3. 通过d7y的helm修改的config.toml 一重启 rke2-server/agent 就会被覆盖，所以，最终要修改 `/etc/rancher/rke2/registries.yaml` ，
而这个改动需要重启rke2-server/agent才生效，所以注意测试是否对业务有影响，尽量一次改完
4. 注意污点(taints)对于d7y daemon的影响，如果确定要不走d7y的，注意别改 `/etc/rancher/rke2/registries.yaml`,
虽然 containerd 有fallback，但是多少影响点时间不是么，如果有污点也有用d7y 记得在values里加上对应的容忍(tolerations)
5. 注意d7y的磁盘规划，以及缓存时间的设置
6. 可以通过 多次运行 `time sudo /var/lib/rancher/rke2/bin/crictl --config=/var/lib/rancher/rke2/agent/etc/crictl.yaml pull xxx:latest`
镜像来评估d7y对于镜像的加速作用(如果是在一台执行，记得执行
`sudo /var/lib/rancher/rke2/bin/crictl --config=/var/lib/rancher/rke2/agent/etc/crictl.yaml rmi --prune`来清理无用镜像)
7. containerd1.4.x 支持多注册中心的办法：1. 等d7y官方支持，参见PR[chore: enable range feature gate in e2e](https://github.com/dragonflyoss/Dragonfly2/pull/1059)，
2，等rancher官方支持containerd1.5.9且你的集群升得动，3，改hosts劫持(但是不支持fallback),4,只加速最常用的一个注册中心,
5,将其他不常用的注册中心的镜像pull&push到加速的注册中心里（注意别有镜像冲突）6,起两套daemon分别监听65001 65002
8. d7y支持预热功能，但是consoleui版本的，暂时没测通，api版本可以，
9. 参见文档 [https://github.com/dragonflyoss/Dragonfly2/blob/main/docs/zh-CN/preheat/api.md](https://github.com/dragonflyoss/Dragonfly2/blob/main/docs/zh-CN/preheat/api.md)
10. [Harbor](https://goharbor.io/docs/main/administration/p2p-preheat/manage-preheat-providers/) p2p 预热支持d7y
11. containerd如果要配置私有镜像库加速，需要配置`127.0.0.1:65001`的auth，详见 [issues dragonflyoss/Dragonfly2/#1065](https://github.com/dragonflyoss/Dragonfly2/issues/1065#issuecomment-1041049794)

### 附赠：docker hub 转移镜像到阿里私服bash脚本

注意将xxxx替换成实际值
用法 `/path/to/pull_push.sh nginx:alpine`

```bash
#!/usr/bin/env bash

sudo service docker start

sudo docker login -uxxxx -pxxxxx registry.cn-zhangjiakou.aliyuncs.com

sudo docker pull $1

sudo docker tag $1 registry.cn-zhangjiakou.aliyuncs.com/xxxx/${1##*/}

sudo docker push registry.cn-zhangjiakou.aliyuncs.com/xxxx/${1##*/}
```

## 招聘小广告

山东济南的小伙伴欢迎投简历啊 [加入我们](https://www.zhipin.com/gongsi/e78fa84f96fef4e733J60tq8EA~~.html) ,
一起搞事情。
长期招聘，Java程序员，大数据工程师，运维工程师，前端工程师。

## 参考资料

- [我的博客](https://anjia0532.github.io/2022/02/14/d7-jib/)
- [我的掘金](https://juejin.cn/post/7065181687411376141/)
