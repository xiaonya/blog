---
title: "StatefulSet 与持久化"
sidebar_position: 4
---

# StatefulSet 与持久化

> 实验环境：三节点 K8s v1.36.3 + Calico，NFS 后端 `192.168.0.241:/K8SStorage`
> SC：`nfs-client`（`WaitForFirstConsumer` + `allowVolumeExpansion: true` + `Delete`）
> 本篇实验输出均来自真实集群。

---

## 核心认知

**StatefulSet 解决的是 Deployment 结构上做不到的三件事。**

| 能力 | Deployment | StatefulSet |
|---|---|---|
| **网络标识** | Pod 名随机、无固定 DNS | **`web-0` / `web-1` / `web-2` 固定不变** |
| **存储** | 所有副本共享或各自临时卷 | **每个副本一个独立 PVC，永久绑定** |
| **启停顺序** | 并行、无序 | **按序号 0→1→2 创建，反向终止** |

**关键在「身份」两个字**：StatefulSet 里的 Pod **不是可互换的**。`web-0` 永远是 `web-0`，永远用 `data-web-0`，永远不会串到别人的数据。

这就是它能承载数据库、消息队列这类有状态应用的原因。

---

## 一、★ 两个"稳定"由两套独立机制提供（最容易混淆的点）

| 稳定什么 | 靠什么字段 | 解决什么问题 |
|---|---|---|
| **网络标识** | `serviceName` + **Headless Service** | 每个 Pod 有**可解析的固定域名** |
| **存储** | `volumeClaimTemplates` | 每个 Pod 有**专属的持久卷** |

**两者互不相干。**

> **记忆**：`serviceName` 给 Pod 发「身份证」，`volumeClaimTemplates` 给 Pod 发「保险柜」。

**常见误解**：以为 `serviceName` 和存储绑定有关。**没有任何关系** —— 它纯粹是 DNS 身份的基础。

---

## 二、volumeClaimTemplates：PVC 的自动生成器

```yaml
spec:
  serviceName: sts-lab-headless     # ★ 必须指向 Headless Service
  replicas: 3
  volumeClaimTemplates:             # ★ 模板，不是单个 PVC
    - metadata:
        name: data                  # ← 模板名
      spec:
        accessModes: [ReadWriteOnce]
        storageClassName: nfs-client
        resources:
          requests:
            storage: 1Gi
```

### PVC 命名规则

```
<模板名>-<StatefulSet 名>-<序号>

data - sts-lab - 0   →  data-sts-lab-0
data - sts-lab - 1   →  data-sts-lab-1
data - sts-lab - 2   →  data-sts-lab-2
```

**三个副本 = 三个独立 PVC，一一对应，不会混。**

### 完整 YAML（Headless Service + StatefulSet）

```yaml
apiVersion: v1
kind: Service
metadata:
  name: sts-lab-headless
spec:
  clusterIP: None            # ★ Headless
  selector:
    app: sts-lab
  ports:
    - name: web
      port: 80
      targetPort: 80
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: sts-lab
spec:
  serviceName: sts-lab-headless
  replicas: 3
  selector:
    matchLabels:
      app: sts-lab
  template:
    metadata:
      labels:
        app: sts-lab
    spec:
      containers:
        - name: app
          image: nginx:alpine
          ports:
            - containerPort: 80
          volumeMounts:
            - name: data
              mountPath: /data
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: [ReadWriteOnce]
        storageClassName: nfs-client
        resources:
          requests:
            storage: 1Gi
```

---

## 三、Headless Service 与逐 Pod DNS

### 为什么必须是 Headless

**只有 Headless Service（`clusterIP: None`）才会生成「逐 Pod 的 DNS 记录」。**

**普通 ClusterIP Service：**

```
DNS: nginx-svc.default.svc  →  10.96.x.x（VIP）
                                ↑ 只有 Service 级别的一条记录
```

**Headless Service：**

```
DNS: nginx-svc-headless.default.svc          →  3 个 Pod IP（集合）
DNS: web-0.nginx-svc-headless.default.svc    →  10.244.x.x（web-0）  ← ★ 逐 Pod
DNS: web-1.nginx-svc-headless.default.svc    →  10.244.y.y（web-1）  ← ★
DNS: web-2.nginx-svc-headless.default.svc    →  10.244.z.z（web-2）  ← ★
```

**如果 `serviceName` 指向普通 ClusterIP Service，`web-0.svc-name` 这个域名根本解析不出来。**

### 实测验证

```bash
DNS=$(kubectl -n kube-system get svc kube-dns -o jsonpath='{.spec.clusterIP}')

# 逐 Pod 记录
for i in 0 1 2; do
  dig @$DNS web-$i.nginx-svc-headless.default.svc.cluster.local +short
done

# Service 级别记录
dig @$DNS nginx-svc-headless.default.svc.cluster.local +short
```

**实测结果**：

```
web-0 → 单个 Pod IP，且与 kubectl get pods 里的 web-0 完全一致
web-1 → 单个 Pod IP，一致
web-2 → 单个 Pod IP，一致
Service → 返回 3 个 IP
```

**两种查询粒度，两种语义**：精确寻址 vs 负载均衡集合。

### 为什么需要它

**有些应用必须能「精确寻址某个实例」，而不是"随便一个副本"：**

```
MySQL 主从：
  mysql-0.mysql-headless  →  主库（接受写入）
  mysql-1.mysql-headless  →  从库 1
  mysql-2.mysql-headless  →  从库 2

客户端要写数据 → 必须精确连到 mysql-0
普通 Service → DNS 返回 VIP，负载均衡到随机副本 → 可能连到从库，写入失败
```

### 机制：记录是怎么生成的

**K8s 并没有"特殊对待 StatefulSet"**，逐 Pod DNS 来自 Pod 自己的两个字段：

```bash
kubectl get pod web-0 -o jsonpath='hostname={.spec.hostname}{"\n"}subdomain={.spec.subdomain}{"\n"}'
```

**预期**：

```
hostname=web-0
subdomain=nginx-svc-headless
```

**机制链条**：

```
StatefulSet 自动给 Pod 设置：
    hostname  = <Pod 名>
    subdomain = <serviceName>
        ↓
CoreDNS 发现 subdomain 指向 Headless Service
        ↓
生成 A 记录：<hostname>.<subdomain>.<ns>.svc.cluster.local
```

**反证**：普通 Deployment 的 Pod，这两个字段都是空的

```bash
kubectl get pod app-a-xxx -o jsonpath='{.spec.hostname} / {.spec.subdomain}{"\n"}'
# → " / "   （都是空）
```

### 逐 Pod DNS 成立的三个条件（缺一不可）

| 条件 | 谁提供 |
|---|---|
| Pod 有 `hostname` + `subdomain` | **StatefulSet 自动设置**（Deployment 不设） |
| 同名 Service 是 **Headless** | 手工创建 |
| `serviceName` 指向它 | `spec.serviceName` |

**三者缺一，DNS 记录就不存在。** 这也是为什么 `serviceName` 写错时，Pod 能跑但域名解析不出来 —— 一个很隐蔽的故障。

---

## 四、有序创建（OrderedReady）

### 实测时间线（★ 铁证）

```
sts-lab-0   Pod 09:42:22  │  PVC 09:42:22  │  PV 09:42:22
sts-lab-1   Pod 09:42:24  │  PVC 09:42:24  │  PV 09:42:24
sts-lab-2   Pod 09:42:26  │  PVC 09:42:26  │  PV 09:42:26
                 ↑ 间隔精确 2 秒，三组时间戳完全同步
```

### 三个细节

**① PVC 与 Pod 的创建时间「完全一致」**

因为 StatefulSet controller 在**同一个 reconcile 循环**里：

```
reconcile(sts-lab):
   1. 检查 sts-lab-0 → 不存在
   2. 创建 PVC data-sts-lab-0      ← 先建 PVC
   3. 创建 Pod sts-lab-0            ← 再建 Pod（引用那个 PVC）
   4. 等 sts-lab-0 Ready
   5. 对序号 1 重复上述流程
```

**"先建 PVC 再建 Pod"是硬性顺序** —— 否则 Pod 会因找不到 PVC 而卡住。

**② 2 秒间隔 = 等前一个 Pod Ready**

```
sts-lab-0 → Running + Ready  →  才创建 sts-lab-1
sts-lab-1 → Running + Ready  →  才创建 sts-lab-2
```

**如果前一个起不来，后面的永远不会创建。** 这是有意设计 —— 数据库从库不该在主库就绪前启动。

**③ PV 也在同一秒创建**

因为 SC 是 `WaitForFirstConsumer`：PVC 创建后先等，Pod 一创建立刻触发供给，两步之间 0 秒。

### `podManagementPolicy`

| 取值 | 行为 | 适用场景 |
|---|---|---|
| `OrderedReady`（默认） | 严格 0→1→2，前一个 Ready 才建下一个 | 主从、有启动依赖 |
| `Parallel` | **同时创建全部** | 对等节点、无启动依赖（如 Cassandra） |

⚠️ **即使设成 `Parallel`，终止时仍然是反向有序的（2→1→0）。**

---

## 五、RWO 为什么没阻止跨节点部署（易误解）

实测 Pod 分布：

```
sts-lab-0  →  k8s-node02
sts-lab-1  →  k8s-node01
sts-lab-2  →  k8s-node02
```

`volumeClaimTemplates` 里写的是 `accessModes: [ReadWriteOnce]`，三个 Pod 却分布在两个节点上。

**因为 RWO 限制的是「单个 PVC」，不是「整个 StatefulSet」**：

```
data-sts-lab-0  → 只被 node02 挂载   ✅
data-sts-lab-1  → 只被 node01 挂载   ✅
data-sts-lab-2  → 只被 node02 挂载   ✅
```

**三个独立 PVC，各自满足 RWO。**

这正是 StatefulSet 的存储模型：**不是共享一个卷，而是每人一个卷。** 如果换成 Deployment 共享一个 RWO 卷，就真的会被限制在同一节点。

---

## 六、★★ 核心考点：PVC 不会被自动删除

### 行为总表

| 操作 | Pod | **PVC** |
|---|---|---|
| `replicas: 3 → 1` | 序号 > 1 的被删 | **保留，状态仍 `Bound`** |
| `replicas: 1 → 3` | 重建 | **复用原 PVC，数据还在** |
| `kubectl delete sts` | 全部删除 | **全部保留，仍 `Bound`** |
| `kubectl delete pvc` | — | 才真的删（按 `reclaimPolicy` 处理） |

### ★ 实测一：缩容 3 → 1

```bash
# 先写入标记
for i in 0 1 2; do
  kubectl exec sts-lab-$i -- sh -c "echo \"I am sts-lab-$i, written at \$(date +%H:%M:%S)\" > /data/marker.txt"
done

kubectl scale sts sts-lab --replicas=1
kubectl get pvc | grep data-sts-lab
```

**结果**：`sts-lab-1/2` 的 Pod 被删，**`data-sts-lab-1/2` 状态仍为 `Bound`，没有被删**。

### ★ 实测二：扩容 1 → 3（复用旧卷）

```bash
kubectl scale sts sts-lab --replicas=3
```

**结果（★ 最硬的证据是 AGE 对比）**：

```
data-sts-lab-0   PVC AGE 8m2s     sts-lab-0  Pod AGE 8m2s
data-sts-lab-1   PVC AGE 8m      sts-lab-1  Pod AGE 101s   ← Pod 重建过
data-sts-lab-2   PVC AGE 7m58s   sts-lab-2  Pod AGE 100s   ← 同上
```

**PVC 比 Pod 老了 6 分多钟。**

```
09:42:22  PVC 创建
09:47:40  写入 marker.txt
09:48:37  Pod 被删除 → 重建（AGE 从 0 开始）
          PVC 没有任何变化（AGE 继续累加）
```

**三条交叉验证**：

| 证据 | 结果 |
|---|---|
| `marker.txt` 内容 | `written at 09:47:40`（写入时刻，不是新时间） |
| PV 名 | `pvc-b6f4e9ad` / `pvc-1759b2ae` / `pvc-4efbfabb` —— **与缩容前完全相同** |
| Pod → PVC 映射 | `sts-lab-2 → data-sts-lab-2`（序号绑死） |

> **判断"卷有没有被重建"最直接的方法：看 PVC 的 `AGE`。** 它是 PVC 自己的寿命，不是 Pod 的。

### ★ 实测三：删除 StatefulSet

```bash
kubectl delete sts sts-lab
```

**结果**：

```
sts-lab            → NotFound（StatefulSet 已删）
Pod                → No resources found
data-sts-lab-0/1/2 → Bound，AGE 12m   ← ★ 全部保留
ownerReferences    → 空                ← ★ 关键证据
```

**`ownerReferences` 为空，说明 PVC 不归 StatefulSet 管 —— 所以不会被级联删除。**

### 为什么设计成这样

**StatefulSet 管的是「有状态」应用，数据的价值高于 Pod 的价值。**

| 设计 | 误删 StatefulSet 的后果 |
|---|---|
| **自动删 PVC**（"整洁"） | **数据库文件随 Pod 一起消失** —— 不可恢复 |
| **保留 PVC**（当前设计） | 留下几个没人用的 PVC —— 事后 `kubectl delete pvc` 即可 |

**一个可修复，一个不可修复。K8s 选择了保守的那一边。**

**代价**：必须有人记住清理

```bash
kubectl delete sts <name>
kubectl get pvc | grep <模板名>-<sts名>-     # 确认残留
kubectl delete pvc <逐个处理>                # 确认后再删
```

> **"宁可留垃圾，不可丢数据"** —— 与 Day1 的 `Retain` → `Released` 是同一个设计哲学。

### 注意：`Edited` ≠ `Released`

| 状态 | 前提 | PVC 对象 | PV 状态 |
|---|---|---|---|
| **删 StatefulSet** | PVC 保留 | **存在** | **`Bound`** |
| **删 PVC + `Retain`** | PVC 被删 | 不存在 | **`Released`** |

**`Released` 的前提是「PVC 对象被删除」。** 只删 StatefulSet 不会走到这一步。

---

## 七、容错链（K8s 存储的整体思路）

```
误删 Pod           → 自动重建，无感
误删 StatefulSet   → PVC 保留，重新 apply 即恢复
误删 PVC           → NFS 上留 archived-*（默认归档），可人工捞回
NFS 归档也被清      → 只能靠备份
```

**每一层都留了后路，直到最后一层才真正丢数据。**

### 重建复用（机制推导，本次未实测）

删掉 StatefulSet 后重新 apply 同一份 YAML，会：

```
创建 Pod sts-lab-0 → 发现 PVC data-sts-lab-0 已存在 → 直接挂载
                   → 不会新建 PVC
```

**因为 PVC 名是「确定性生成」的**（`data-sts-lab-0`），只要 StatefulSet 名和模板名不变，重建时必然命中同一个 PVC。

**这一点决定了：「误删 StatefulSet 可以无损恢复」** —— 但前提是**没有删 PVC**。

---

## 八、其他考点

### ① `partition` —— 分区滚动更新（金丝雀）

```yaml
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 2        # 只更新序号 >= 2 的 Pod
```

```
partition: 2  →  只更新 sts-lab-2，0 和 1 保持旧版本
                ↓ 验证没问题
partition: 0  →  全部更新
```

**用途**：先在序号最大的副本上试新镜像，观察没问题再全量。**StatefulSet 自带的金丝雀能力**，不需要额外的 Deployment。

### ② `--cascade=orphan` —— 删控制器但留 Pod

```bash
kubectl delete sts sts-lab --cascade=orphan
```

**效果**：StatefulSet 对象被删，**Pod 变成「孤儿」继续运行**。

**用途**：把 StatefulSet 从 Helm 切换到手工管理、或迁移到另一个控制器时，避免服务中断。之后再 `kubectl apply` 一个新的 StatefulSet，它会**认领**这些同名 Pod（因为名字完全匹配）。

### ③ `podManagementPolicy: Parallel`

见第四节。**只在「无启动依赖」的场景使用**，否则会破坏 StatefulSet 存在的意义。

---

## 九、踩坑实录

### 坑 1：把 `serviceName` 理解成"存储绑定"

**`serviceName` 只负责 DNS 身份**，和 PV/PVC 毫无关系。存储由 `volumeClaimTemplates` 负责。

### 坑 2：以为删 StatefulSet 会把 PVC 一起带走

**不会。** PVC 没有指向 StatefulSet 的 `ownerReference`，级联删除碰不到它。

### 坑 3：把 `Released` 和"删 StatefulSet"混为一谈

- 删 StatefulSet → PVC 还在 → **`Bound`**
- 删 PVC（Retain）→ **`Released`**

**`Released` 的前提是 PVC 对象被删。**

### 坑 4：以为 `accessModes: RWO` 会限制所有 Pod 到同一节点

**RWO 限制的是单个 PVC，不是整个 StatefulSet。** 每个 Pod 有独立 PVC，可以分布在不同节点。

### 坑 5：`serviceName` 漏写或写错

Pod 能正常跑，但**域名解析不出来** —— 因为 `subdomain` 指向的 Service 不存在或不是 Headless。

**排查**：`kubectl get pod <name> -o jsonpath='{.spec.subdomain}'` 看它指向谁。

### 坑 6：前一个 Pod 起不来，后面的永远不创建

`OrderedReady` 模式下，`sts-lab-0` 卡在 Pending/ImagePullBackOff 时，**`sts-lab-1/2` 压根不会被创建**。

```bash
kubectl get pods -l app=sts-lab
# NAME        READY   STATUS
# sts-lab-0   0/1     Pending        ← 只有这一个
# （1 和 2 根本不存在）
```

**别以为是控制器坏了 —— 这是设计行为。** 排查方向是先解决 0 号 Pod 的问题。

### 坑 7：清理时只删 StatefulSet，忘了 PVC

```bash
kubectl delete sts sts-lab
kubectl get pvc        # ← 发现一堆残留
```

**生产运维文档里必须写明这一步。**

### 坑 8：`podManagementPolicy` 改了会影响已有 StatefulSet 吗

`podManagementPolicy` **可以修改**（不是不可变字段），但只影响后续的创建/删除行为，不会重建已有 Pod。

---

## 十、复习自测题

**概念**

1. StatefulSet 相比 Deployment，提供了哪三项 Deployment 做不到的能力？
2. `serviceName` 和 `volumeClaimTemplates` 分别负责什么？两者有关系吗？
3. 为什么 `serviceName` 必须指向 Headless Service？普通 ClusterIP 会怎样？
4. 逐 Pod DNS 记录成立需要哪三个条件？

**机制**

5. `data-sts-lab-1` 这个名字是怎么生成的？
6. 为什么 PVC 和 Pod 的创建时间"完全一致"？控制器的执行顺序是什么？
7. `OrderedReady` 和 `Parallel` 的区别？终止顺序会因 `Parallel` 改变吗？
8. `accessModes: ReadWriteOnce` 为什么没有阻止三个 Pod 分布在两个节点？

**核心行为**

9. 缩容 `3 → 1` 后，`data-sts-lab-2` 会怎样？状态是什么？
10. 扩容 `1 → 3` 后，`sts-lab-2` 挂的是新 PVC 还是旧的？怎么验证？
11. 删除 StatefulSet 后，PVC 会怎样？为什么不会被级联删除？（提示：`ownerReferences`）
12. 删 StatefulSet 后的 PVC 状态是 `Bound` 还是 `Released`？`Released` 的前提是什么？
13. 「误删 StatefulSet 能无损恢复」的前提条件是什么？

**运维**

14. 如何做 StatefulSet 的金丝雀发布？
15. `--cascade=orphan` 用在什么场景？

---

## 十一、速记卡片

```
【StatefulSet 三件事】
稳定网络标识 / 稳定存储 / 有序启停

【两个"稳定"是两套机制】
serviceName + Headless  →  DNS 身份（身份证）
volumeClaimTemplates    →  专属存储（保险柜）
两者互不相干

【PVC 命名】
<模板名>-<sts名>-<序号>   →  data-sts-lab-0

【逐 Pod DNS 三个条件】
Pod 有 hostname + subdomain（StatefulSet 自动设置）
同名 Service 是 Headless（clusterIP: None）
serviceName 指向它
三者缺一，记录不存在

【有序创建】
OrderedReady（默认）：0 Ready → 建 1 → 1 Ready → 建 2
间隔 ~2 秒；前一个起不来，后面的不创建
PVC 与 Pod 同秒创建（控制器先建 PVC 再建 Pod）

【RWO 的粒度】
限制单个 PVC，不是整个 StatefulSet
每个 Pod 独立 PVC → 可以跨节点

【★ PVC 不自动删】
缩容       → Pod 删，PVC 留（仍 Bound）
扩容回来   → 复用旧 PVC，数据还在
删 StatefulSet → Pod 全删，PVC 全留
原因：PVC 没有指向 StatefulSet 的 ownerReferences

【Released 的前提】
PVC 对象被删 → PV 才 Released
只删 StatefulSet 不会 Released

【判断卷有没有重建】
看 PVC 的 AGE（不是 Pod 的）
AGE 比 Pod 老 = 复用旧卷

【容错链】
删 Pod → 自动重建
删 StatefulSet → PVC 保留，重新 apply 恢复
删 PVC → NFS 留 archived-*
归档也清 → 只能靠备份

【其他】
partition: N        → 只更新序号 >= N 的（金丝雀）
--cascade=orphan    → 删控制器留 Pod
podManagementPolicy → OrderedReady / Parallel（终止仍反向有序）
```

---

## 附：Week6 剩余计划

| Day | 主题 | 状态 |
|---|---|---|
| Day1 | PV / PVC 生命周期 | ✅ |
| Day2 | StorageClass + 动态供给 | ✅ |
| Day3 | 绑定模式 + 扩容 | ✅ |
| Day4 | StatefulSet + volumeClaimTemplates | ✅ 本篇 |
| Day5 | 存储排障 + 收口 | ⬜ |
