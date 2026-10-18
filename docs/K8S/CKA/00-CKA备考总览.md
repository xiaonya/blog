---
title: "CKA 备考总览"
sidebar_position: 1
---

# CKA 备考总览

> 集群状态：✅ 三节点 Ready（2026-08-09 搭建完成，见 `03-集群初始化.md`）
> 目标：CKA 认证（Kubernetes Administrator），官方认证考试，机考实操
> 环境：Master 192.168.10.12（本机练习环境）

## 〇、重要前提：考试环境 ≠ 本地环境

- CKA 考试是**浏览器里的虚拟环境**（官方提供的集群 + 终端），不是本机
- 本地环境配置（补全/alias）考试里**不提供**，但练习时能大幅提升效率，值得配
- 本地准备的核心价值：① 练熟 kubectl 命令肌肉记忆；② 熟悉考点对应的知识点

## 一、本地环境准备（练习用）

### 1. kubectl bash 补全
```bash
kubectl completion bash > /etc/bash_completion.d/kubectl
echo 'source /usr/share/bash-completion/bash_completion' >> ~/.bashrc
```

### 2. alias（CKA 社区通用习惯）
```bash
cat >> ~/.bashrc <<'EOF'
alias k='kubectl'
alias kg='kubectl get'
alias kd='kubectl describe'
alias kl='kubectl logs'
alias kaf='kubectl apply -f'
EOF
source ~/.bashrc
```

### 3. 常用工具
- jq ✅ 已有（`/usr/sbin/jq`）
- vim ✅ 已有
- yq ❌ 未装（操作 YAML 常用，考试环境自带，本地学的时候方便）
```bash
dnf install -y yq
```

### 4. 当前环境状态（2026-08-09 21:56 体检）
| 项目 | 状态 |
|------|------|
| kubectl context | kubernetes-admin@kubernetes ✅ |
| 三节点 | Ready ✅ |
| kubectl v1.36.3 | ✅ |
| jq / vim | ✅ |
| bash 补全 | ❌ 待配 |
| alias k | ❌ 待配 |
| yq | ❌ 待装 |
| 磁盘 / | 58G 可用 53G ✅ |

## 二、CKA 考点权重表（官方大纲 · 2026-08 最新版）

> ⚠️ **2026 版考纲已更新**：故障排除升为最高权重 30%，架构/安装合并为 25%，工作负载降到 15%。
> 之前笔记里 20%/10% 的旧权重作废，以下表为准。

### 考试基本参数（官方）
| 项目 | 值 |
|------|-----|
| 时长 | 2 小时 |
| 通过分 | 66% |
| Kubernetes 版本 | v1.35（随版本季度更新） |
| 费用 | $445（含 1 次免费重考） |
| 模式 | 线上机考、浏览器虚拟环境、命令行实操 |
| 语言 | 可选中/英文监考官 |
| 有效期 | 2 年（CARE 计划可续期） |
| 重考 | 1 次免费 | 

### 领域权重表
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
- 时间分配：17 题 120 分钟，平均每题 7 分钟，排障题（权重 20%）通常耗时
- 考试界面有 Notepad 和 Calculator 内嵌工具，关键信息（IP、namespace）随手记
- 提交前用 `kubectl get` 验证答案生效

## 五、学习记录

### 2026-08-09
- 集群搭建完成，三节点 Ready
- CKA 基线环境体检：补全/alias/yq 待配置
- 本笔记创建
