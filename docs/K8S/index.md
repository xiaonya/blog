---
title: 导读
sidebar_position: 1
---

# 导读

本笔记主要用于记录 k8s 的学习过程，包括一些基础概念，常用命令，常见问题等，后续会不断更新。目标是通过这个学习笔记，能够帮助自己更好地理解 k8s，同时也希望能够在熟练后能够考个 cka 的证书来证明自己，也有利于自己后续的就业和职业发展。

## 怎么用这个栏目

三份材料的使用方式不一样，别一次全读完：

| 章节 | 内容 | 怎么用 |
|---|---|---|
| [入门基础概念](./01-基础概念.md) | 集群构成、API 对象模型、工作负载、网络与存储模型 | **读一遍**，先建立全局图，不求记住细节 |
| [CKA 备考笔记](./CKA/00-CKA学习计划.md) | 学习计划（Week 0 ~ 12）+ 按周推进的实操笔记与复习测试 | **当主线**，先看计划表确认本周任务，实操完做闭卷测试 |
| [命令速查](./Command.md) | 按场景分类的 kubectl / kubeadm 命令 | **别通读**，练习时忘了就翻，考前过一遍 |

## 学习路径

学习按周推进，每周先在集群上实操，再用一份闭卷复习测试收口。

| 周次 | 主题 | 起步页 |
|---|---|---|
| Week 0-1 | 基线准备、集群搭建、架构与控制平面组件 | [基线准备](./CKA/Week0-1/01-基线准备.md) |
| Week 2 | Pod 生命周期、多容器、Namespace、kubectl 进阶 | [Pod 创建与生命周期](./CKA/Week2/01-Pod创建与生命周期.md) |
| Week 3 | 配置与密钥、资源限制与 QoS | [ConfigMap](./CKA/Week3/01-ConfigMap.md) |
| Week 4 | Deployment、探针、控制器家族、综合实战 | [Deployment 与滚动更新](./CKA/Week4/01-Deployment与滚动更新.md) |
| Week 5 | Service、CoreDNS、Ingress、NetworkPolicy、Gateway API | [Service 与 Endpoints](./CKA/Week5/01-Service与Endpoints.md) |
| Week 6 | PV/PVC、StorageClass、StatefulSet、存储排障 | [PV / PVC 生命周期](./CKA/Week6/01-PV与PVC生命周期.md) |

考纲权重、考试形式与备考策略见 [CKA 备考总览](./CKA/01-CKA备考总览.md)。

## 参考教程

- [feisky参考指南](https://kubernetes.feisky.xyz/introduction/index)
- [JimmySong的K8S手册](https://jimmysong.io/zh/book/kubernetes-handbook/)
- [CNCF中文手册](https://doc.cncf.vip/kubernetes-handbook)
- [官方文档](https://kubernetes.io/zh/docs/home/)
- [K8S Bootcamp](https://kubernetesbootcamp.github.io/kubernetes-bootcamp/index.html)
- [CNCF对CKA考试的介绍](https://doc.cncf.vip/kubernetes-handbook/she-qu-ji-sheng-tai/certification/about-cka-candidate)
- [SREdevops](https://k8s.sredevops.top/#/learning)

> CKA 考试允许访问 `kubernetes.io/docs`，所以从备考第一天起就练习用官方文档查字段，比背命令更划算。
