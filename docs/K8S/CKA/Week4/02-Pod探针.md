---
title: "Pod 探针：liveness / readiness / startup"
sidebar_position: 2
---

# Pod 探针：liveness / readiness / startup

## 三种探针，管的事不一样

| 探针 | 执行期 | 失败时动作 | 失败后 STATUS | 失败后 READY |
|---|---|---|---|---|
| **livenessProbe** | 运行期，周期性 | kubelet **重启容器** | 退避期 `CrashLoopBackOff`，重启后短暂 Running | **0/1**（容器死了） |
| **readinessProbe** | 运行期，周期性 | **不重启**，从 Service endpoints **摘除** | 一直 `Running` | **0/1**（活着但没资格） |
| **startupProbe** | 启动期，一次性 | 重启容器（给足宽限） | 同 liveness | — |

**记忆锚点（词源法）**：
- **live** = 活。失败 = 活不下去 → 杀掉重来
- **ready** = 就绪。失败 = 还没准备好接客 → 不给流量，但人不杀

一句话：**liveness 管生死，readiness 管上岗资格**。

## 三种探测方式

| 方式 | 原理 | 适合 |
|---|---|---|
| `httpGet` | HTTP 请求，**只有 2xx/3xx 算成功**，4xx/5xx 都失败 | Web 应用 /healthz |
| `tcpSocket` | TCP 端口握手 | 数据库/Redis 等端口型服务 |
| `exec` | 容器内执行命令，exit 0 成功 | 无健康端点、可写脚本验证的应用 |

## 探针参数

```yaml
livenessProbe:
  exec:
    command: ["cat", "/tmp/healthy"]
  initialDelaySeconds: 5    # 首次探测前延迟（等应用起）
  periodSeconds: 5          # 探测间隔
  failureThreshold: 3       # 连续失败 N 次才算真失败
  successThreshold: 1       # 连续成功 N 次恢复（readiness 可 >1，liveness 恒为 1）
  timeoutSeconds: 1         # 单次探测超时
```

真正的"失败延迟" = `periodSeconds × failureThreshold`（5s × 3 = 15s 后才动手）。

## startupProbe：慢应用的免检通行证

- startupProbe 成功**之前**，liveness/readiness **全部禁用**
- 成功之后，startupProbe 退出历史舞台，另外两个接管
- Java 类启动 1-2 分钟的应用，用它给宽限期，否则 liveness 会在启动期反复误杀

## 今日实验实录（Master01 实测）

| Pod | 设计 | 实测结果 |
|---|---|---|
| pod-liveness-exec | 30s 后删 /tmp/healthy，探针 cat | **重启 27 次**，循环杀（Killing 事件 + exit 137） |
| pod-liveness-http | 探 nginx `/` | 0 重启，稳定 Running |
| pod-liveness-http2 | 探 `/noexist`（404） | **CrashLoopBackOff ×23**，`Liveness probe failed: statuscode: 404` ×49 |
| pod-ready | 删 /tmp/ready 观察摘流 | READY 0/1 但 STATUS Running，endpoints 摘空 |
| pod-startup | sleep 40s 模拟慢启动 | startup 宽限期内无重启，成功后 liveness 接管 |

## 关键排障技能：两种"容器重启"的鉴别

| 证据 | 探针杀的 | 进程自己退出 |
|---|---|---|
| `Killing` 事件 | ✅ `failed liveness probe, will be restarted` | ❌ 无 |
| Last State reason | `Error` | `Completed` |
| Exit Code | **137**（=128+9，SIGKILL） | **0**（正常跑完） |
| Unhealthy 事件 | `Liveness probe failed: ...` | 最多 readiness 的 Warning |

实测样本：
- pod-liveness-exec/http2：`exit 137` + Killing 事件 → liveness 杀的
- pod-ready：`Completed exit=0` + 无 Killing → **是 sleep 600 到期自然退出**，restartPolicy: Always 拉起，与 readiness 无关

## 今日踩坑与修正

### 1. 实验进程别用短 sleep
`sleep 600` 的常驻 Pod 10 分钟后自然退出（Completed exit=0），会伪装成"探针故障"。常驻实验用 `sleep 3600` 或 `while true; do sleep 60; done`。

### 2. liveness 失败的 Pod READY 也会 0/1（修正手册原话）
原以为"liveness 失败 READY 一直 1/1"——实测 CrashLoopBackOff 退避期容器是死的，READY 同样 0/1。**与 readiness 的本质区别是"死 vs 活着但不合格"**，不是 READY 数字本身。

### 3. httpGet 的 404 算失败
健康路径配错（返回 404）= 应用活着但探针一直失败 = 反复被杀。排障时先确认探针路径真的返回 2xx。

## 常驻检查命令

```bash
kubectl get pod <pod> -w                                         # 盯 RESTARTS / READY
kubectl describe pod <pod> | grep -B2 -A4 -iE 'killing|unhealthy' # 探针事件
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[0].lastState.terminated.reason} {.status.containerStatuses[0].lastState.terminated.exitCode}'
kubectl get endpoints <svc>                                      # 看流量摘除
```

## 复习题答案

1. **liveness 失败**：STATUS 退避期 CrashLoopBackOff、READY 0/1（容器死了）；**readiness 失败**：STATUS 一直 Running、READY 0/1、endpoints 摘除（活着但没资格）。
2. 失败。httpGet 只有 2xx/3xx 算成功。
3. 137 = 128+9 = **SIGKILL 杀的**，大概率 liveness 失败后 kubelet 杀容器。正常退出是 exit 0（Completed）。❗易混点：进程"运行结束"是 exit 0，137 是被杀。
4. initialDelaySeconds = 首次探测前延迟；periodSeconds = 探测间隔；failureThreshold = 可容忍连续失败次数。
5. 启动 2 分钟 > 失败阈值（3×5s=15s）→ 启动期反复被杀、永远起不来。解决：加 startupProbe 给宽限期（如 failureThreshold=30 × periodSeconds=5 = 150s 余量）。

## 速记卡片

```
探针三兄弟
├─ startup   启动期一次性（成功前别的探针全禁用）
├─ liveness  失败 → 杀（exit 137 / Killing / CrashLoopBackOff）
└─ readiness 失败 → 摘流（Running 不变 / READY 0/1 / endpoints 空）

探测三方式：httpGet(2xx/3xx) / tcpSocket(端口) / exec(exit 0)
参数三件套：initialDelay / period / failureThreshold
失败延迟 = period × failureThreshold
```
