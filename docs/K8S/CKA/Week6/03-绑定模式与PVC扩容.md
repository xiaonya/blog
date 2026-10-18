---
title: "绑定模式与 PVC 扩容"
sidebar_position: 3
---

# 绑定模式与 PVC 扩容

> 实验环境：三节点 K8s v1.36.3 + Calico，NFS 后端 `192.168.0.241:/K8SStorage`
> 本篇实验输出全部来自真实集群。

---

## 核心认知

**`volumeBindingMode` 决定的是「什么时候」绑定 PV 和 PVC。**

```
Immediate（默认）      PVC 一创建 ──立刻──> 绑定 + 创建底层存储
WaitForFirstConsumer   PVC 一创建 ──挂起──> 等有 Pod 要用它 ──> 才绑定
```

**一个字段的差别，带来两种完全不同的行为。**

| | `Immediate` | `WaitForFirstConsumer` |
|---|---|---|
| PVC 创建后 | **立刻 Bound** | **一直 Pending** |
| 底层存储何时创建 | 立刻 | **等 Pod 调度后** |
| provisioner 何时被触发 | PVC 创建时 | **Pod 创建时** |
| 适用场景 | 存储无拓扑限制（NFS、本地盘） | **存储有拓扑限制**（云盘、可用区） |

---

## 一、为什么需要 WFFC —— 拓扑限制

**关键在于「存储只能建在某个位置」这件事。**

云环境里，云盘是**可用区绑定**的：

```
集群横跨 us-east-1a 和 us-east-1b

Immediate 模式：
  PVC 一创建 → 立刻在 1a 建了块云盘
  Pod 偏偏被调度到 1b 的节点
  → 挂不上那块盘 → Pod 永远 Pending  ❌

WaitForFirstConsumer 模式：
  先让调度器决定 Pod 去哪（比如 1b）
  → 再在 1b 建盘
  → 盘和 Pod 同区，一定能挂上  ✅
```

> **WFFC 的价值是：让存储的创建位置「跟随」Pod 的调度决定。**

顺带一个成本视角：**没 Pod 用就不建盘** —— 在云上这是真金白银的节省。

⚠️ **本环境的局限**：NFS 没有拓扑限制（任何节点都能访问同一个共享目录），所以 **WFFC 的实际好处在 NFS 上体现不出来**。但**行为差异完全能验证**，而且那个现象本身极其重要 —— 见下节。

---

## 二、★ WFFC 最容易被误判为故障的地方

切到 WFFC 之后，**建完 PVC 会发现它一直 Pending**：

```
NAME       STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS
pvc-wffc   Pending                                      nfs-client
```

**第一反应往往是"PVC 坏了"** —— 然后去查 provisioner、查 SC、查权限，全是白费功夫。

**正确动作是看 Events**：

```
Events:
  Type    Reason                Message
  Normal  WaitForFirstConsumer  waiting for first consumer to be created before binding
```

**`waiting for first consumer`（在等第一个消费者）** —— 这不是故障，是设计行为。把 Pod 建起来，它立刻 Bound。

**另一个细节**：这条事件会**反复出现**（实测 `x42 over 29m`，约每 40 秒一次）。那种刷屏感极容易让人以为出事了。

---

## 三、实验一：切换到 WFFC

### ① 直接改 —— 被拒（不可变字段）

```bash
kubectl patch sc nfs-client -p '{"volumeBindingMode":"WaitForFirstConsumer"}'
```

**报错**：

```
The StorageClass "nfs-client" is invalid:
volumeBindingMode: Forbidden: updates to volumeBindingMode are forbidden
```

**`volumeBindingMode` 也在 SC 的不可变字段清单里** —— 和 `provisioner` / `parameters` / `reclaimPolicy` 一样，**只能删了重建**。

### ② 删了重建

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-client
provisioner: k8s-sigs.io/nfs-subdir-external-provisioner
parameters:
  server: 192.168.0.241
  path: /K8SStorage
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer      # ★ 唯一改动
allowVolumeExpansion: true
```

### ③ 只建 PVC（不建 Pod）

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-wffc
spec:
  accessModes: [ReadWriteMany]
  storageClassName: nfs-client
  resources:
    requests:
      storage: 1Gi
```

**实测结果**：PVC `Pending`，Events 如上节所述，**NFS 上没有任何新目录**，**provisioner 日志里没有任何记录**。

### ④ 再建 Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-wffc
spec:
  containers:
    - name: app
      image: nginx:alpine
      volumeMounts:
        - name: d
          mountPath: /data
  volumes:
    - name: d
      persistentVolumeClaim:
        claimName: pvc-wffc
```

**实测结果**：PVC 立刻 `Bound`，PV 自动生成，**NFS 目录同时出现**。

---

## 四、★ 铁证：三处时间戳互相印证

```
PVC pvc-wffc                    AGE 29m      ← 14:24 前后创建
PV  pvc-c107c6fd-...            AGE 15m      ← 15:37 才产生
NFS 目录 default-pvc-wffc-...   时间戳 15:37  ← 与 PV 同一时刻
Pod pod-wffc                    AGE 15m      ← 15:37 创建
```

**PVC 建了 29 分钟，但 PV 只存在了 15 分钟。**

**中间那 14 分钟**：

- PVC 状态 `Pending`
- **NFS 上什么都没有**
- **provisioner 日志里没有任何 provision 记录** ← 这条最关键

provisioner 日志里第一个 `pvc-wffc` 相关记录是 `07:37:33 UTC`（= 15:37:33 本地）。**在建 Pod 之前，provisioner 压根没被触发过一次。**

### 两种模式的事件流对比

**Immediate（Day2 那次）**

```
PVC 创建
   ↓
K8s 判定"走动态供给" → 通知 provisioner
   ↓
provisioner 立刻 mkdir + 造 PV + 绑定
```

**WaitForFirstConsumer（本次）**

```
PVC 创建
   ↓
K8s 说"等等，还没消费者" → 【不通知 provisioner】
   ↓
（PVC 一直 Pending，反复发 WaitForFirstConsumer 事件 ×42 次）
   ↓
Pod 创建 → 调度器决定 Pod 去哪 → 告诉 K8s"消费者来了"
   ↓
这时才通知 provisioner → mkdir + 造 PV + 绑定
```

> **WFFC 省掉的不只是"绑定时机"，还有「存储的创建」本身。**

---

## 五、PVC Pending 的四种可能（排障分诊）

```
PVC Pending
   ├─ Events: waiting for first consumer ...        → 【正常】在等 Pod
   ├─ Events: waiting for a volume to be created,
   │          either by external provisioner 'xxx'  → 【异常】provisioner 不在 / 名字不对
   ├─ Events: 什么都没有                             → 【异常】没有默认 SC，也没指定 SC
   └─ Events: provision failed ...                  → 【异常】provisioner 报错
```

**排障动作**：拿到 Pending 的 PVC，**第一步永远是 `describe` 看 Events 的 Reason**，不是去重启 provisioner。

---

## 六、实验二：PVC 扩容 —— 失败，但失败得很有价值

### 操作

```bash
kubectl patch pvc pvc-wffc -p '{"spec":{"resources":{"requests":{"storage":"2Gi"}}}}'
```

### 实测：三层状态对不上

| 层 | 值 | 说明 |
|---|---|---|
| PVC `spec.resources.requests.storage` | **2Gi** | ✅ **patch 成功，请求改上去了** |
| PVC `status.capacity.storage` | **1Gi** | ❌ 实际容量没变 |
| PV `spec.capacity.storage` | **1Gi** | ❌ 没变 |

### 决定性线索

```
Type    Reason             Age     From            Message
Normal  ExternalExpanding  6m59s   volume_expand   waiting for an external controller to expand this PVC
```

**`waiting for an external controller to expand this PVC`** —— K8s 在等 provisioner 动手，**但 provisioner 没有任何反应**。

### 根因

**`nfs-subdir-external-provisioner` 没有实现 `ExpandVolume`（ResizeProvisioner）接口** → 收到扩容请求后**直接忽略** → PVC 永远卡在 `ExternalExpanding`。

**这不是配置错误，是组件能力缺失。**

---

## 七、★ `allowVolumeExpansion: true` 的真相

**它只是必要条件，不是充分条件。**

```
allowVolumeExpansion: true
   → 只是让 apiserver 【接受】你改大 requests 的请求
   → 不设的话，改 requests 会被 apiserver 直接拒绝
   → 但【真正执行扩容】是 provisioner 的活
```

**这和 Day2 的 `reclaimPolicy` 是同一个模式**：

| 字段 | 是什么 | 谁真正执行 |
|---|---|---|
| `reclaimPolicy: Delete` | K8s 层的**指令** | provisioner（决定归档还是真删） |
| `allowVolumeExpansion: true` | K8s 层的**许可** | provisioner（决定能否扩、怎么扩） |

---

## 八、扩容的完整链路

```
① 用户改 PVC 的 requests          ← 只能改大，不能改小
② apiserver 检查 allowVolumeExpansion  → 允许
③ K8s 找到该 PVC 的 provisioner，请求扩容
④ ★ provisioner 执行扩容          ← 大多数外部/非 CSI provisioner 卡在这一步
⑤ 更新 PVC 的 status.capacity
```

**CSI 驱动**还有更细的分段：

```
④a ControllerExpansion  扩大底层卷（云 API / 存储后端）
④b NodeExpansion        扩大文件系统（kubelet 参与，可能需要重启 Pod）
```

---

## 九、两个容易混的状态

| 状态 / 事件 | 含义 | 该做什么 |
|---|---|---|
| `ExternalExpanding` | **在等 provisioner 动手**（什么都没开始） | 检查 provisioner 是否支持扩容 |
| `FileSystemResizePending` | **底层卷已扩，文件系统还没扩** | **重启 Pod** 完成文件系统扩容 |

**一个是"没人干"，一个是"干了一半"。** 排查方向完全不同。

---

## 十、踩坑实录

### 坑 1：`volumeBindingMode` 不可变

和 `parameters` 一样，只能删了重建。SC 的不可变字段清单（Day1 已列）：

```
provisioner / parameters / reclaimPolicy / volumeBindingMode / allowedTopologies  → 全部不可变
allowVolumeExpansion                                                            → 唯一能改的
```

### 坑 2：WFFC 下 PVC Pending 是**正常**的

**这是本 Day 最值得记的一条。** 看到 Pending 先看 Events，别急着"修"。

### 坑 3：WFFC 的事件会刷屏

`waiting for first consumer to be created before binding` 每 40 秒重复一次，短时间内累积几十条。**刷屏 ≠ 故障。**

### 坑 4：`allowVolumeExpansion: true` 不代表能扩容

**生产上如果存储不支持扩容，就不要设这个字段**（或显式设 `false`）：

| 配置 | 你改 requests 时 |
|---|---|
| `true`（但 provisioner 不支持） | **静默卡住**，PVC 状态看起来还正常 ← 最坑 |
| `false` / 不设 | apiserver **立刻拒绝**，明确报错 ← 诚实 |

> **宁可明确失败，不要静默成功一半。**

### 坑 5：容量只能往上改

改大了就**回不去**（缩容被 apiserver 拒绝，`Forbidden: field can not be less than previous value`）。

**后果**：如果扩容卡住了（像本次这样），这个 PVC 只能**删掉重建** —— 没有退路。

### 坑 6：扩容卡住时 PVC 状态显示 `Bound`

**又一个哑巴故障** —— 表面看完全正常，真相只在 Events 里。

和 Day2 那个「日志说 `volume deleted` 其实只是改名」是同一类问题：**表面指标正常，真相在细节里。**

### 坑 7：`umount /mnt/nfs` 报 `device is busy`

NFS 挂载点被占用（通常是某个 shell 的当前目录在里面）。处理：

```bash
cd /                 # 先离开挂载点
umount /mnt/nfs
# 还不行就 lazy unmount
umount -l /mnt/nfs
```

**验证**：`mount | grep nfs` 应该没有输出。

**为什么要在意**：挂载残留 + NFS 属性缓存 → 下次 `ls` 拿到旧数据，做出错误判断。

---

## 十一、复习自测题

**绑定模式**

1. `Immediate` 和 `WaitForFirstConsumer` 的区别是什么？后者解决什么问题？
2. 为什么云盘场景**必须**用 `WaitForFirstConsumer`？举一个会出问题的具体场景。
3. WFFC 模式下，PVC 建好后一直 Pending，Events 里应该看到什么？这是故障吗？
4. WFFC 下，底层存储是在什么时候被创建的？由谁触发？
5. `volumeBindingMode` 能改吗？要改怎么办？
6. 为什么 WFFC 在云上能省钱？

**扩容**

7. `allowVolumeExpansion: true` 的含义是什么？设了它就一定能扩容吗？
8. PVC 扩容的完整链路有哪几步？哪一步最容易成为瓶颈？
9. `ExternalExpanding` 和 `FileSystemResizePending` 分别是什么意思？各自的下一步动作是什么？
10. PVC 的容量能改小吗？为什么？
11. 如果存储不支持扩容，`allowVolumeExpansion` 应该设成什么？为什么？

---

## 十二、速记卡片

```
【两种绑定模式】
Immediate            → PVC 一建就绑 + 建存储
WaitForFirstConsumer → 等 Pod 调度才绑 + 才建存储

【WFFC 的价值】
存储的创建位置跟随 Pod 的调度决定
云盘/可用区场景必须用；没 Pod 用就不建盘 → 省钱

【WFFC 的 Pending 是正常的】
Events: waiting for first consumer to be created before binding
这条事件每 ~40 秒重复一次，会刷屏，不是故障

【PVC Pending 分诊】
waiting for first consumer  → 正常，等 Pod
waiting for external provisioner 'xxx' → provisioner 不在/名字不对
什么都没有                  → 没有默认 SC 且未指定 SC
provision failed            → provisioner 报错

【SC 不可变字段】
provisioner / parameters / reclaimPolicy / volumeBindingMode / allowedTopologies
只有 allowVolumeExpansion 能改 → 改不了就删了重建

【扩容是"许可 + 能力"两件事】
allowVolumeExpansion: true  → apiserver 允许（许可）
provisioner 实现 ExpandVolume → 真能扩（能力）
只有许可没有能力 → 静默卡在 ExternalExpanding

【两个扩容状态】
ExternalExpanding         → 没人动手
FileSystemResizePending   → 底层扩了，文件系统没扩 → 重启 Pod

【容量只能往上】
缩容被拒：Forbidden: field can not be less than previous value
扩容卡住 → 只能删 PVC 重建，没有退路

【反直觉】
扩容卡住时 PVC 状态仍是 Bound（哑巴故障）
→ 拆解：表面正常时，答案在 Events 里
```

---

## 附：Week6 剩余计划

| Day | 主题 | 状态 |
|---|---|---|
| Day1 | PV / PVC 生命周期 | ✅ |
| Day2 | StorageClass + 动态供给 | ✅ |
| Day3 | 绑定模式 + 扩容 | ✅ 本篇 |
| Day4 | StatefulSet + `volumeClaimTemplates` | ⬜ |
| Day5 | 存储排障 + 收口 | ⬜ |
