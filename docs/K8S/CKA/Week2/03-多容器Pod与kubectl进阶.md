---
title: "多容器 Pod 与 kubectl 进阶"
sidebar_position: 3
---

# 多容器 Pod 与 kubectl 进阶

> 日期：2026-08-24 ｜ 环境：VMware 虚拟机集群（Tabby 终端）

## 一、今日目标

- [x] 用 dry-run 生成 YAML 骨架再改（CKA 标准工作流）
- [x] 多容器 Pod：验证共享存储、共享网络、独立生命周期
- [x] 容器级排障：describe 按容器分列 + jsonpath 提取 containerStatuses
- [x] kubectl edit 与 Pod spec immutable 限制
- [x] 导出→删除→重建的 YAML 修改流程
- [x] Label 与 selector 语法

## 二、操作记录

### 1. dry-run 生成 YAML 骨架

```bash
kubectl run sidecar-demo --image=busybox:1.36 --dry-run=client -o yaml > sidecar-demo.yaml
```

三个要点：`--dry-run=client`（本地生成不创建）、`-o yaml`（YAML 格式）、`> file`（保存）。
考试 90% 的 YAML 都这么来，没人从空文件开始写。

### 2. 多容器 Pod YAML（sidecar 日志模式）

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sidecar-demo
spec:
  volumes:
    - name: shared
      emptyDir: {}
  containers:
    - name: app
      image: busybox:1.36
      command: ['sh', '-c', 'while true; do date >> /data/log.txt; sleep 5; done']
      volumeMounts:
        - name: shared
          mountPath: /data
    - name: sidecar
      image: busybox:1.36
      command: ['sh', '-c', 'tail -f /data/log.txt']
      volumeMounts:
        - name: shared
          mountPath: /data
```

### 3. 验证三件事

```bash
# 共享存储：两个容器读同一个文件
kubectl exec sidecar-demo -c app -- cat /data/log.txt
kubectl exec sidecar-demo -c sidecar -- cat /data/log.txt

# 共享网络：sidecar 里 localhost 就是 Pod IP
kubectl exec sidecar-demo -c sidecar -- wget -qO- http://localhost:80 2>&1 | head -5

# 独立生命周期：app 改成 exit 1 后重启，用 jsonpath 看各自 restartCount
kubectl get pod sidecar-demo -o jsonpath='{range .status.containerStatuses[*]}{.name}{"\t"}{.restartCount}{"\t"}{.state}{"\n"}{end}'
```

实验结果：app Restart Count = 5（CrashLoopBackOff），sidecar = 0（Running）。
back-off 2m40s = 160s = 第 5 次退避，与 Day1 推算一致。

### 4. kubectl edit 与 immutable

```bash
kubectl edit pod web    # 改 image 成功（触发容器替换，RESTARTS 不涨）
kubectl edit pod web    # 加 command 被拒绝
```

Pod spec 可改字段白名单（报错原文）：
- `spec.containers[*].image`
- `spec.initContainers[*].image`
- `spec.activeDeadlineSeconds`
- `spec.tolerations`（只能新增）
- `spec.terminationGracePeriodSeconds`（负数→1）

### 5. 导出→删除→重建

```bash
kubectl get pod web -o yaml > web.yaml   # 导出
vi web.yaml   # 删系统字段 + 改 command + 改名
kubectl delete pod web
kubectl create -f web.yaml
```

要删的系统字段：uid、resourceVersion、creationTimestamp、managedFields、status 整段、last-applied-configuration annotation。
`kube-api-access-xxxxx` volume（自动注入的 SA 凭证）删不删都行，删了会自动重建。

### 6. Label 操作

```bash
kubectl label pod web-v2 env=dev team=cka    # 加
kubectl get pod web-v2 --show-labels          # 查
kubectl label pod web-v2 env-                 # 删（key 加减号）
kubectl label pod web-v2 env=test --overwrite # 改值（不加 --overwrite 会报错）
```

### 7. Selector 四连

```bash
kubectl get pods -l env=dev --show-labels        # 等于
kubectl get pods -l env!=dev --show-labels       # 不等（含无 label 的 Pod！）
kubectl get pods -l 'env in (dev,test)'          # 集合（要加引号）
kubectl get pods -l env=dev,team=cka             # AND（逗号）
```

## 三、关键概念（用自己的话填）

### emptyDir

- 生命周期：Pod 创建时建在节点上，Pod 删除时数据永久删除。**容器重启不会丢数据**（volume 在 Pod 层）。
- 存储介质：默认磁盘，`medium: Memory` 挂 tmpfs（占容器内存配额，可能触发 OOMKill）。
- 初始为空，两容器都能读写。

### 多容器 Pod 的三个共享 / 一个独立

- 共享：网络（同一 IP，localhost 互通）、存储（volume）、网络命名空间
- 独立：生命周期（容器各自重启，RESTARTS 按容器计）

### create vs apply

（留白，自己总结：何时用 create，何时用 apply，为什么"删了重建"场景用 create 更好）

### Exit Code 三分法

| Exit Code | 含义 | 排查方向 |
|---|---|---|
| 0 | 正常退出（Completed） | ？ |
| 1 等非 0 | 出错退出 | ？ |
| 137 | SIGKILL（常见 OOMKill） | ？ |

（排查方向留白，自己填）

## 四、实验结论与踩坑

### 坑 1：kubectl edit 改 image 不触发 RESTARTS

RESTARTS 只统计"容器终止后按 restartPolicy 重新拉起"。改 image 是**容器替换**（停旧→建新），不是"重启失败容器"。验证方法：`describe` 看 Started 时间变了但 RESTARTS 没涨。

### 坑 2：command: ["nginx"] 导致 Completed

nginx 默认 daemonize，主进程 exit 0 → 容器 Completed。
正确写法：`command: ["nginx", "-g", "daemon off;"]`
关键认知：command 覆盖 ENTRYPOINT，args 对应 CMD。

### 坑 3：selector name 与 selector 互斥

`kubectl get pod web-v2 -l env=dev` 报错 `name cannot be provided when a selector is specified`。
pod 名是"精确点名"，selector 是"批量筛选"，二者不能同时用。

### 坑 4：`!=` 会命中没有该 label 的对象

| selector | 有 env=dev | 有 env=prod | 没 env label |
|---|---|---|---|
| `env=dev` | ✅ | ❌ | ❌ |
| `env!=dev` | ❌ | ✅ | **✅**（实验验证！） |
| `!env` | ❌ | ❌ | ✅ |

（实验数据：`env!=dev` 返回了没有 env label 的 web Pod + env=prod 的 web3）

## 五、排障套路

### 多容器 Pod 排障

1. `kubectl get pod` → READY `1/2` 说明两容器中死了一个
2. `kubectl describe pod` → Containers 段按容器分列，看各自 State 和 Restart Count
3. `kubectl logs <pod> -c <容器名> --previous` → 看崩溃容器上次日志
4. jsonpath 一行提取：`-o jsonpath='{range .status.containerStatuses[*]}{.name}{"\t"}{.restartCount}{"\t"}{.state}{"\n"}{end}'`

### Pod spec 改不了怎么办

1. `kubectl get pod xxx -o yaml > new.yaml`
2. 删系统字段（uid、resourceVersion、creationTimestamp、managedFields、status）
3. 改需要的字段 + 改名
4. `kubectl delete pod xxx && kubectl create -f new.yaml`

### Service 没有后端 endpoint 排障思路

（留白：用 Service 的 selector 当 -l 条件查 Pod，查不到就是 label 对不上）

## 六、今日遗留 / 待深入

（留白：command 覆盖 ENTRYPOINT vs args 覆盖 CMD 的区别，Week4 会深入；kube-api-access volume 的原理，Week8 RBAC 会讲）

## 七、参考文档

- Pod 基础：https://kubernetes.io/zh-cn/docs/concepts/workloads/pods/
- emptyDir：https://kubernetes.io/zh-cn/docs/concepts/storage/volumes/#emptydir
- Labels 与 selector：https://kubernetes.io/zh-cn/docs/concepts/overview/working-with-objects/labels/
- kubectl logs：https://kubernetes.io/zh-cn/docs/reference/kubectl/generated/kubectl_logs/
- `kubectl explain pod.spec.volumes.emptyDir`（离线文档）
