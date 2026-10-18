---
title: CKA 学习计划
sidebar_position: 1
---

# CKA 学习计划

> 数据来源：WorkBuddy 项目看板「CKA 备考总计划」及其 15 个周任务，快照时间 **2026-10-18**。
> 看板是唯一事实来源，本页可能滞后；进度有变化以看板为准。

## 进度总览

**7 / 15 个周任务完成**（Week 0 ~ Week 6），当前进行 Week 7。表里有 8 个 ✅ 是因为 Week 5 拆出了一个 Week 5B 子任务专攻 Gateway API，看板上它挂在 Week 5 下面，不单独计数。

一条主干：先用四周把集群从零搭起来、把核心对象跑一遍，再用 Week 5~6 打通网络与存储，最后把 30% 权重的排障拆成两周专门练。轻量内容（配置、密钥、控制器）不单独占周，压在对应周里。

## 周次与时间表

| 周次 | 主题 | 计划时间 | 状态 |
|---|---|---|---|
| Week 0 | 准备阶段：环境搭建与基础概念 | 8 月 | ✅ 完成 |
| Week 1 | 集群架构与控制平面组件 + kubeadm | 8 月 | ✅ 完成 |
| Week 2 | 核心概念：Pod / Node / Namespace | 08-22 ~ 08-25 | ✅ 完成 |
| Week 3 | 配置与密钥：ConfigMap / Secret / 资源限制 | 09-01 ~ 09-05 | ✅ 完成 |
| Week 4 | 工作负载与发布：Deployment / RS / DaemonSet / Job | 10-03 ~ 10-04 | ✅ 完成 |
| Week 5 | 服务与网络：Service / Ingress / NetworkPolicy | 10-06 收尾 | ✅ 完成 |
| Week 5B | Gateway API：GatewayClass / Gateway / HTTPRoute | 10-06 起 | ✅ 完成 |
| Week 6 | 存储与 Volume：PV / PVC / StorageClass | 10-12 | ✅ 完成 |
| **Week 7** | **调度与自动伸缩：NodeSelector / Taint / Affinity / HPA** | **10-19** | 🔵 进行中 |
| Week 8 | RBAC 与安全：ServiceAccount / Role / ClusterRole | 10-26 | ⬜ 待开始 |
| Week 9A | 排障基础：应用与网络（Troubleshooting 30% · 应用侧） | 11-02 | ⬜ 待开始 |
| Week 9B | 排障进阶：集群与节点（Troubleshooting 30% · 集群侧） | 11-09 | ⬜ 待开始 |
| Week 10A | etcd 备份 / 集群维护 / Helm | 11-16 | ⬜ 待开始 |
| Week 10B | Kustomize / CRD / Operator | 11-16（后半周） | ⬜ 待开始 |
| Week 11 | 模拟考试与弱项补强 | 11-23 | ⬜ 待开始 |
| Week 12 | killer.sh 第 2 次 + 冲刺 + 考前 | 11-30 | ⬜ 待开始 |

> 考试窗口：**12-07 之后**，目前尚未预约，时间可自由调整。
> Week 10A 和 10B 共用 11-16 这一周，前半周做 etcd/维护，后半周做 Kustomize/CRD。

## 本周（Week 7）待办

**核心知识点**：nodeSelector / nodeName、Taints & Tolerations、Affinity & Anti-Affinity（node & pod）、topologySpreadConstraints、HPA、LimitRange / ResourceQuota

- [ ] 给节点打 label，用 nodeSelector 调度
- [ ] `kubectl taint nodes` 添加污点，测试容忍
- [ ] 配置 nodeAffinity 的 required / preferred
- [ ] 配置 podAntiAffinity 分散 Pod
- [ ] 部署 metrics-server + HPA，压测触发自动伸缩

> ⚠️ 本地集群目前**没装 metrics-server**，`kubectl top` 不可用，HPA 这一项要先补装才能做（见 [CKA 备考总览](./01-CKA备考总览.md) 第一节的环境状态表）。

参考文档方向：`kubernetes.io/docs` → Concepts → Scheduling, Preemption and Eviction

## 考纲对照结论（2026-10-09 核定）

原计划基于旧考纲。按 `CKA_Curriculum_v1.35.pdf` 逐条核对后，2024-11 新增的三条考点处理如下：

| 新增考点 | 处理 |
|---|---|
| Gateway API | ✅ 已补：Week 5B |
| Kustomize + CRD / Operator | ✅ 已补：Week 10B |
| HA 控制面 + CNI/CSI/CRI 扩展接口 | ✅ 已补：Week 11 纸面补漏 |

其余四个领域（Storage / Troubleshooting / Workloads / Services & Networking）覆盖完整。

> 这份对照是按 **v1.35** 考纲做的。考试环境已在 2026-09 升到 **v1.37**，权重未变，但考前建议用官方 curriculum 再核一次。

## 四条铁律

1. 能用命令式（`kubectl create` / `apply` / `run`）就不手写 YAML
2. 学会查 `kubernetes.io/docs`
3. 善用 `kubectl explain`
4. 考场时间管理：先易后难，标记跳过

## 两条纪律

1. **Troubleshooting 占 30%，突击练不出手感**：所以 Week 9 拆成 9A/9B 两周，且从 Week 6 起每周末抽 30 分钟做一次排障小练习
2. **Week 10A 的优先级**：etcd 备份恢复 ＞ drain/cordon ＞ kubeadm upgrade ＞ 证书续期 ＞ Helm（权重最低，可降级为纸面准备）

## 看板里但不属于 CKA 主线的任务

- 【拓展·非考纲】kube-proxy 规则链深挖：iptables / nftables / IPVS（待开始）
- K8S 集群可观测体系搭建：Prometheus + Grafana → OpenTelemetry → VictoriaMetrics 三阶段（待开始）

这两条不影响考试，属于能力补强，做完 CKA 主线再说。
