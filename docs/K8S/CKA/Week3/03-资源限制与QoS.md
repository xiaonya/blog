---
title: "资源限制与 QoS"
sidebar_position: 3
---

# 资源限制与 QoS

> Week 3 · 配置与密钥 · 2026-09-05

## 一、requests vs limits

| | requests | limits |
|---|---|---|
| 含义 | 保证能拿到的最小资源 | 最多能用的硬上限 |
| 影响调度 | ✅ 是 | ❌ 否 |
| 资源不足时 | 调度器优先保证 | 超过即被限制/杀掉 |

单位：
- CPU：`1` = 1 核，`100m` = 0.1 核。**没有小数写法**，0.5 核写 `500m`
- 内存：`Mi`（1024²）、`Gi`（1024³）规范；`M`/`G` 是十进制，混用容易出错

## 二、CPU vs 内存超限行为

| | 超 requests | 超 limits |
|---|---|---|
| CPU | throttle（节流），不死 | throttle，不死（**可压缩资源**） |
| 内存 | 继续跑，但节点紧张时被 OOM 杀 | **必被 OOMKilled**，exit code 137（**不可压缩**） |

排查 OOMKilled：
```bash
kubectl get pod <name> -o jsonpath='{.status.containerStatuses[0].lastState.terminated.reason}'
# 输出 "OOMKilled"
kubectl describe pod <name> | grep -E 'Reason|Exit Code'
```

## 三、QoS 三类判定（核心表）

| QoS | 判定条件 |
|---|---|
| **Guaranteed** | 每个容器的 cpu 和 memory 都**显式**设了 requests 和 limits，且对应相等 |
| **Burstable** | 不满足 Guaranteed，但至少一个容器设了 requests 或 limits |
| **BestEffort** | 所有容器都没设 requests 也没设 limits |

判定优先级：Guaranteed → Burstable → BestEffort。

### 易错点
1. 只设 limits 不设 requests → **Burstable**（CKA 答案）
2. requests 和 limits 相等但只设了 cpu（mem 未设） → **Burstable**
3. requests 设了但没设 limits → **Burstable**
4. **Guaranteed 要求 cpu 和 memory 都满足条件，缺一不可**

## 四、OOM 驱逐顺序

节点内存耗尽时：
```
BestEffort → Burstable → Guaranteed
（先被杀）             （最后被杀）
```

## 五、requests 如何影响调度

requests 决定调度器选哪个节点，与节点当前实际使用无关。
Pod `Pending` + `Events`:`Insufficient memory` → 请求超出节点可分配资源。

注意 Events 里可能还有别的因素：`1 node(s) had untolerated taint(s)`——这种是节点被打了 taint，需要 toleration 才能调度，跟资源无关。两者结合判断。

## 六、踩坑实录

1. **拼写错误**：`pod-guaranted`（少写 `e`）。Pod 创建能成功，但如果用 selector 或 label 选它就要按错的拼写去找。考试手写 YAML 务必拼对。
2. **YAML 单位必须是字符串**：`cpu: "100m"`（带引号）才不会因解析变成 `100` 当作数字。同理 `memory: "64Mi"`。
3. **混淆 units**：`M` 是十进制，`Mi` 是二进制。考题常挖这个，用规范写法。
4. **OOM 排查时只看 `describe` 不够**：OOMKilled 信息在 `lastState.terminated`，要 jsonpath 或 `kubectl describe` 看 Containers 段。

## 七、复习题（答案）

1. requests 和 limits 哪个影响调度？哪个是硬上限？—— requests 影响调度，limits 是硬上限。
2. CPU/内存超 requests/limits 分别会怎样？—— CPU 超了都是 throttle 不死；内存超 requests 不死但紧张时被杀，超 limits 必 OOMKilled。
3. 只设 limits（不设 requests），QoS 是什么？—— **Burstable**（CKA 标准答案）。
4. 100m 等于多少 CPU？0.5 核怎么写？—— 0.1 核；0.5 核写 `500m`。
5. 节点 OOM 时三类 QoS 的驱逐顺序？—— BestEffort → Burstable → Guaranteed。
