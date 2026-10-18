---
title: "CKA 备考总览"
sidebar_position: 2
---

# CKA 备考总览

> 集群状态：✅ 三节点 Ready（2026-08-09 搭建完成，见 `03-集群初始化.md`）
> 目标：CKA 认证（Kubernetes Administrator），官方认证考试，机考实操
> 环境：Master 192.168.10.12（本机练习环境）
> 官方参数核实：**2026-10-18**（来源：Linux Foundation CKA 考试页 + LF 认证 FAQ）

## 〇、重要前提：考试环境 ≠ 本地环境

- CKA 考试是**浏览器里的虚拟环境**（官方提供的集群 + 终端），不是本机
- 本地环境配置（补全/alias）考试里**不提供**，但练习时能大幅提升效率，值得配
- 本地准备的核心价值：① 练熟 kubectl 命令肌肉记忆；② 熟悉考点对应的知识点

## 一、本地练习环境

> 本节描述的是**本机练习集群**（192.168.10.12）的实际状态，不是考试环境。
> 最后复查：**2026-10-18 22:03**（`kubectl get nodes` / `get pods -n kube-system` / `df -h` 实测）

### 1. 当前环境状态

| 项目 | 状态 | 说明 |
|------|------|------|
| kubectl context | ✅ `kubernetes-admin@kubernetes` | KUBECONFIG 已导出 `/etc/kubernetes/admin.conf` |
| 集群版本 | ✅ Client / Server 均 **v1.36.3** | 考试环境 v1.37，差一个小版本，影响很小 |
| 三节点 | ✅ k8s-master / node01 / node02 全部 Ready | 已稳定运行 **70 天** |
| 系统 / 内核 | ✅ Fedora 44 Server / 6.19.10-300 | 三节点一致 |
| 容器运行时 | ✅ containerd 2.3.3 | 三节点一致 |
| CNI | ✅ Calico **v3.29.1** | kube-system 13 个 Pod 全部 Running |
| 命名空间 | 9 个 | 含 Week 3~6 的练习 ns |
| Ingress | ✅ IngressClass `nginx`（ingress-nginx，22 天） | Week 5 实操用 |
| Gateway API | ✅ GatewayClass `nginx`（NGINX Gateway Fabric，13 天，Accepted=True） | Week 5 实操用 |
| 动态存储 | ✅ StorageClass `nfs-client`（nfs-subdir-external-provisioner，`WaitForFirstConsumer`，允许扩容） | Week 6 实操用 |
| bash 补全 | ✅ 已装 `/etc/bash_completion.d/kubectl` | 8 月时还是待配 |
| alias | ⚠️ 只配了 `alias k='kubectl'` | `kg/kd/kl/kaf` 未配，见第 3 小节 |
| jq / vim / yq | ✅ 全部就位（`/usr/sbin/`） | yq 8 月未装，现已装 |
| helm | ❌ 未装 | 架构域（25%）考点，需要时装 |
| metrics-server | ❌ 未装 | `kubectl top` 不可用，HPA 也依赖它 |
| 磁盘 / | ⚠️ 58G 总量，已用 33G，**可用 26G（57%）** | 8 月可用 53G，两个多月用掉 27G |

### 2. 相比 2026-08-09 那次体检

- **三件待办全部完成**：bash 补全、alias k、yq 都已就位
- **练习环境补齐了 Week 5/6 需要的组件**：ingress-nginx、NGINX Gateway Fabric、NFS 动态供给
- **磁盘可用从 53G 降到 26G**：镜像层 + PV 数据累积。建议现在就清掉不用的练习资源（`kubectl delete ns`、`crictl rmi --prune`），别等涨到 85% 再动手
- **两个缺口**：helm（架构域要用）、metrics-server（排障域看资源用量要用）

### 3. 环境配置（留档）

bash 补全 —— 已配：
```bash
kubectl completion bash > /etc/bash_completion.d/kubectl
echo 'source /usr/share/bash-completion/bash_completion' >> ~/.bashrc
```

alias —— 目前只有 `k`，完整别名组（可选，配了更省时间）：
```bash
cat >> ~/.bashrc <<'EOF'
alias kg='kubectl get'
alias kd='kubectl describe'
alias kl='kubectl logs'
alias kaf='kubectl apply -f'
alias kdel='kubectl delete'
EOF
source ~/.bashrc
```

> 注意：补全和 alias 考试环境里**都没有**，练的是自己的肌肉记忆，别依赖 shell 配置答题。

## 二、考试参数与考点权重（2026-10-18 核实官方页面）

### 考试基本参数（官方）

| 项目 | 值 | 说明 |
|------|-----|------|
| 时长 | 2 小时 | 纯实操，终端里做题，无选择题 |
| 通过分 | 66% | CKA/CKAD 同为 66%，CKS 是 67% |
| **Kubernetes 版本** | **v1.37** | 官方承诺新版本发布后 **4–8 周内**对齐最新小版本 |
| 题量 | **官方不公布固定题量** | 报名赠送的 Killer.sh 模拟器每套 17 题，可当强度参考 |
| 费用 | **$445** | 含 12 个月报考窗口 + **2 次考试机会** + 2 次模拟器 |
| 加购价 | $625（+THRIVE-ONE）／$645（+LFS258） | 按需 |
| 模式 | 线上机考，PSI Secure Browser | 浏览器内虚拟集群 + 终端，远程监考 |
| 语言 | 英语 / 简体中文 / 日语 | 任务文本三语，考试中可用语言下拉**随时切换** |
| 有效期 | 2 年 | 2024-04-01 之前取得的 CKA 证书为 3 年 |
| 重考 | 报名自带 **2 次考试机会** | 首考未过用第二次，无需额外付费 |

### 相比 2026-08 的变化（本次核实修正）

| 项 | 8 月笔记写的 | 现在官方口径 | 影响 |
|---|---|---|---|
| 考试环境版本 | v1.35 | **v1.37** | 练习集群 1.36.3 比考试低一个小版本，见下方说明 |
| 题量 | 「17 题 120 分钟」 | 官方不公布题量；17 是**模拟器**每套题量 | 别按 17 题规划时间，按 120 分钟倒排 |
| 重考 | 「含 1 次免费重考」 | 报名含 **2 次考试机会**（12 个月内） | 表述更准确，实际权益更大 |
| 语言 | 「可选中/英文监考官」 | 任务文本 **英/简中/日**，可随时切换 | 不是选监考官语言 |
| 有效期 | 2 年 | 2 年（2024-04-01 前为 3 年） | 补充例外 |
| 领域权重 | 25/15/20/10/30 | **未变** | 权重表继续有效 |

> 一个容易混淆的点：官方页面的 Curriculum 链接仍指向 `CKA_Curriculum_v1.35.pdf`，但考试环境标注已是 **v1.37**。
> 也就是说**权重表还是 v1.35 版的内容，跑的环境已经是 v1.37**。考点范围按权重表准备，具体对象行为以 1.37 文档为准。

### 本地练习环境与考试的版本差

- 本地：v1.36.3 ｜ 考试：v1.37（差一个小版本）
- 对 CKA 影响很小：考的 API 对象（Pod/Deployment/Service/PV/PVC/NetworkPolicy/RBAC）跨版本稳定
- 需要留意的只是**被移除或改默认值的字段**。考前花 1 小时过一遍 1.37 的 release notes 中 `Removed` 和 `Changed` 段即可，不必升级集群
- 如果坚持对齐：`kubeadm upgrade plan` → 逐节点升级，属于 Week 0-1 学过的内容，可以当复习

### 领域权重表（Curriculum v1.35，权重未变）

> 故障排除为最高权重 30%，架构/安装合并为 25%，工作负载 15%。前期笔记里 20%/10% 的旧权重已作废。

| 领域 | 权重 | 核心考点 |
|------|------|---------|
| **集群架构、安装和配置** | **25%** | RBAC、kubeadm 建集群、集群生命周期（升级）、高可用控制平面、Helm/Kustomize、扩展接口（CNI/CSI/CRI）、CRDs 和 Operators |
| **工作负载和调度** | **15%** | Deployment 滚动更新/回滚、ConfigMap/Secret、自动扩缩容、自愈部署、Pod 准入和调度（limits/node affinity/taint） |
| **服务和网络** | **20%** | Pod 连通性、NetworkPolicy、Service（ClusterIP/NodePort/LoadBalancer）+ Endpoints、Gateway API、Ingress 控制器/资源、CoreDNS |
| **存储** | **10%** | StorageClass 动态供给、卷类型/访问模式/回收策略、PV/PVC 管理 |
| **故障排除** | **30%** | 集群/节点排障、组件排障（apiserver/etcd/kubelet）、资源监控、容器输出流、服务与网络排障 |

> **重点**：
> - 故障排除 30% 权重最高，考试会把集群搞坏让你修（Pod CrashLoopBackOff、节点 NotReady、DNS 不解析等）
> - **etcd 备份恢复** 在架构域必考，考试大纲明确列出
> - 新版考纲新增：**Gateway API**、**CRDs/Operators**、**Helm/Kustomize**（注意别漏）

## 二·1、考试 open-book 规则（可查阅内容）

CKA 是**开卷考**，但只能查特定官方资源：

| 可查阅 | 说明 |
|--------|------|
| **kubernetes.io/docs** | 官方文档，最大依仗，Tasks 章节有逐步操作步骤 |
| **kubernetes.io/blog** | 官方博客 |
| **github.com/kubernetes** | K8S 官方 GitHub |

**限制**：
- 只能开 **1 个额外浏览器标签页**，只允许访问上面三个域名
- 搜索引擎、其他博客、AI 工具等一律不可用
- → 平时练好"去 kubernetes.io/docs 查具体步骤"的习惯，考试时这就是救命稻草
- 官方原文清单：`docs.linuxfoundation.org/tc-docs/certification/certification-resources-allowed`（考前以它为准）

## 三、学习路径规划

基于当前进度（集群已就绪），推荐顺序：

1. **配好本地环境**（第一节，10 分钟）
2. **第一个实验**：Deployment + Service + Ingress——打通"应用跑起来→对外暴露"主线
3. **排障练习**：删 coredns 重建、cordon/uncordon 节点、`kubectl describe` 找问题
4. **etcd 备份恢复**（CKA 必考、权重高、唯一集群级操作）
5. **刷模拟题**：CKA 模拟题巩固 + 查漏补缺

## 四、CKA 考试实用技巧

- **官网资源**：kubernetes.io/docs 是考试时唯一可查的官方资料（含任务章节操作步骤），必须熟练使用
- 每题先读题干 → 找到对应 namespace → 用 `kubectl` 快速创建，避免手写长 YAML（多用 `kubectl create/run` 生成骨架再改）
- 时间分配：官方不公布题量，按 120 分钟倒排。**排障题 30% 权重、通常最耗时**，先做完能立刻上手的题，把排障留出足够余量
- 考试界面有 Notepad 和 Calculator 内嵌工具，关键信息（IP、namespace）随手记
- 提交前用 `kubectl get` 验证答案生效

## 五、学习记录

### 2026-08-09
- 集群搭建完成，三节点 Ready
- CKA 基线环境体检：补全/alias/yq 待配置
- 本笔记创建

### 2026-10-18
- **复查本地练习环境**：三节点 70 天稳定运行，Calico v3.29.1、ingress-nginx、NGF、nfs-client 均在线；8 月的三件待办（补全 / alias k / yq）已全部完成
- 新发现两个缺口：helm 未装、metrics-server 未装（`kubectl top` 不可用）
- 磁盘可用 53G → 26G，需要清理练习资源
- 顺带按官方页面核实考试参数，**考试环境版本 v1.35 → v1.37**（K8s 1.37 于 2026-09 发版，官方 4–8 周内对齐）
- 修正三处不准确表述：题量 17 是模拟器题量而非考试题量；「1 次免费重考」实为报名含 2 次考试机会；语言不是选监考官，而是任务文本英/简中/日可切换
- 补充有效期例外：2024-04-01 之前取得的 CKA 证书有效期 3 年
- 权重表经核对未变（25/15/20/10/30），继续有效
