---
title: "Pod 创建与生命周期"
sidebar_position: 1
---

# Pod 创建与生命周期

> 日期：2026-08-22 ｜ 环境：VMware 虚拟机集群（Tabby 终端）

## 一、今日目标

- [x] 创建第一个 Pod，观察 Pending → ContainerCreating → Running
- [x] 制造 ImagePullBackOff 与 CrashLoopBackOff
- [x] 掌握 `describe` 的 Events / Last State 段
- [x] 验证 kubelet 重启的指数退避
- [x] 理解 `logs --previous` 的真实作用

## 二、操作记录（自己补全命令与现象）

```bash
# 1. 创建 nginx Pod 并观察状态变化
kubectl run nginx --image=nginx:1.27
kubectl get pod nginx -w

# 2. 镜像拉取失败
kubectl run broken --image=nginx:1.99

# 3. 容器启动即崩溃
kubectl run crasher --image=busybox:1.36 -- sh -c 'echo "I am starting"; sleep 5; exit 1'

# 4. 清理
kubectl delete pod crasher t2 nginx broken
```

## 三、关键概念（用自己的话写一遍）

### 1. Pod 生命周期
<!-- Pending / ContainerCreating / Running / Error / Completed 各是什么意思？谁在推动状态变化？ -->

### 2. 重启策略
<!-- restartPolicy 默认值？可选值？作用于什么级别（Pod 还是容器）？ -->

### 3. 指数退避
<!-- 首次失败是否退避？退避序列是什么？封顶多久？ -->

## 四、实验结论

### 指数退避实测数据（来自 crasher Pod）

| 第几次崩溃 | Error 时刻 | 下次 Running 时刻 | 实际等待 | 理论退避值 |
|---|---|---|---|---|
| 1 | 23s | 33s | 10s | 10s |
| 2 | 39s | 66s | 27s | 20s |
| 3 | 72s | 118s | 46s | 40s |
| 4 | 123s | 209s | 86s | 80s |

结论：等待时长 10→20→40→80 严格翻倍；实际值比理论值多 6~7s（容器运行时开销）；首次失败不退避。

### 两种故障状态对比

| | ImagePullBackOff | CrashLoopBackOff |
|---|---|---|
| 失败阶段 | 镜像拉取，容器从未创建 | 容器已启动并运行过，Exit Code ≠ 0 退出 |
| 第一个查的地方 | `describe` 的 Events 段 | `logs --previous` + `describe` 的 Last State |

分界口诀：**有 Exit Code = 容器跑过（查应用层）；无 Exit Code = 容器没跑过（查镜像/调度层）**

### logs --previous 的坑
<!-- 为什么实验中加不加 --previous 输出相同？它真正有用的场景是什么？ -->

## 五、排障套路（Day1 沉淀）

1. Pod 异常 → 先 `kubectl get pod -o wide`（看 Node 和 RESTARTS）
2. `kubectl describe pod` → Events 段看流水账
3. 容器反复重启 → `kubectl logs <pod> --previous` 看死前日志
4. 常用参数：`logs -f`（跟随）、`logs --tail=N`（限定行数）

## 六、遗留问题 / 待深入

<!-- 记录今天没想明白的点，第二天回来看 -->

## 七、参考文档

- https://kubernetes.io/zh-cn/docs/concepts/workloads/pods/pod-lifecycle/
- https://kubernetes.io/zh-cn/docs/reference/kubectl/generated/kubectl_logs/
