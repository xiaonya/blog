---
title: "PV / PVC 生命周期"
sidebar_position: 1
---

# PV / PVC 生命周期

> 实验环境：三节点 K8s v1.36.3 + Calico，存储后端 NFS `192.168.0.241:/K8SStorage`
> 本篇的实验输出全部来自真实集群，可直接复现。

---

## 核心认知

**K8s 存储是四层抽象，每一层都在解决上一层留下的问题。**

```
① 容器可写层      → 容器实例重建，数据就没了
② Volume          → 数据挪到 Pod 层，但 Pod 删了还是没
③ PV / PVC        → 数据挪到集群层，跨 Pod 生命周期；但 PV 要管理员手工建
④ StorageClass    → 让存储自动分配，不用手工建 PV
```

**记住这条主线：每一层抽象，都是把数据往「更长寿」的地方挪。**

而 PV / PVC 这一层的设计意图是**供需解耦**：

```
管理员视角：我有什么存储        → 池化成 PV（集群级资源）
开发者视角：我要多大、怎么用     → 提申请 PVC（命名空间级资源）
                    ↓
              K8s 负责把两者【匹配】绑定
```

---

## 一、Volume：Pod 怎么用存储

**核心认知：Volume 的生命周期与 Pod 绑定**（不是与容器绑定）。

同一个 Pod 里的多个容器可以共享 Volume；容器重建，Volume 里的数据还在；**但 Pod 被删除，Volume 也就没了**。

### 常用 Volume 类型

| 类型 | 生命周期 | 典型用途 |
|---|---|---|
| `emptyDir` | **跟 Pod**（Pod 删则删） | 容器间共享临时数据、缓存、`/tmp` |
| `hostPath` | 跟节点（Pod 删了数据还在） | ⚠️ 生产慎用（绑定具体节点，跨节点调度就找不到） |
| `configMap` | 跟 ConfigMap 对象 | 注入配置文件 |
| `secret` | 跟 Secret 对象 | 注入凭据（内存文件系统 tmpfs） |
| `downwardAPI` | 跟 Pod | 把 Pod 自身元数据（IP、标签、limits）暴露成文件 |
| `nfs` | 跟 NFS 服务器（**独立于 Pod 和节点**） | 共享存储 |
| **`persistentVolumeClaim`** | **跟 PV**（可跨 Pod 生命周期） | ★ **生产标准做法** |
| `projected` | 多种来源合并 | 把多个 secret / configMap 汇到一个目录 |

### 两个考点细节

- `emptyDir` 可以指定 `medium: Memory` → 变成 tmpfs（内存盘），但**会计入容器的内存 limit**
- `hostPath` 有个 `type` 字段（`DirectoryOrCreate` / `FileOrCreate` / `Socket` 等），**不写 type 时目录不存在会直接失败**

### ★ 关键澄清：为什么"容器重启"会丢数据

**这里有个容易说错的点**，必须精确：

| 数据放在哪 | 容器重建 | **Pod 被删除** |
|---|---|---|
| **容器可写层**（直接写文件） | ❌ 丢失 | ❌ 丢失 |
| **`emptyDir` 卷** | ✅ **保留** | ❌ 丢失 |
| **PVC** | ✅ 保留 | ✅ **保留** |

**准确的说法不是"容器重启数据就没了"，而是「容器实例被销毁重建时，可写层的数据才丢」。**

关键在于**"重启"到底会不会销毁容器实例** —— 这里 Docker 和 K8s 行为不同：

| 场景 | 容器实例 | 可写层数据 |
|---|---|---|
| **Docker：`docker restart`** | **原地重启同一个实例** | ✅ **保留** |
| **Docker：`docker rm` 后重新 run** | 新实例 | ❌ 丢失 |
| **K8s：容器崩溃后 kubelet 重启它** | **新建一个容器实例** | ❌ **丢失** |

**K8s 的"重启"不是 Docker 那种原地重启，而是"杀掉旧的、建个新的"。**

验证方法：看 `containerID` 会变

```bash
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[0].containerID}'
# 重启前：docker://abc123...
# 重启后：docker://def456...   ← 变了 = 新容器
```

**为什么 K8s 要这么做**：这是「不可变基础设施」的体现 —— 容器应该无状态、可随时替换。**要保留状态？那就显式声明卷。** 状态必须是显式声明的，不能是隐式积累的。

---

## 二、PV / PVC：供需解耦

| | PV | PVC |
|---|---|---|
| 谁创建 | 管理员（或 provisioner 自动） | 开发者 |
| 作用域 | **集群级**（不属于任何 ns） | **命名空间级** |
| 描述什么 | "我有什么存储" | "我要什么存储" |
| 类比 LVM | 已存在的 LV | 挂载需求申请单 |

**PV 是集群级的，PVC 是命名空间级的** —— 所以不同 ns 不能共享同一个 PVC（但可以各自建 PVC 绑到同一个 PV，前提是 accessModes 允许）。

---

## 三、绑定：四条硬匹配

PVC 找 PV 时**逐条比对**，全部满足才能绑定：

| 条件 | 规则 | 严格程度 |
|---|---|---|
| **① `storageClassName`** | **必须完全相等**（空 ≠ manual） | ★ **精确匹配** |
| ② `accessModes` | PV 必须支持 PVC 要求的**所有**模式 | 宽松（PV 可以更强） |
| ③ `capacity` | PV 容量 **≥** PVC 的 requests | 宽松 |
| ④ `selector` | 若 PVC 写了标签选择器，PV 标签要匹配 | 可选 |

**注意：只有 ① 是"严格相等"，②③④ 都是"PV 可以比要求的好"。**

**匹配失败 = PVC 一直 Pending，不报错，只是沉默。** 这是最常见的"存储哑巴故障"。

---

## 四、`storageClassName` 的三个状态（最易混）

| PVC 的写法 | 有默认 SC 的集群 | **没有默认 SC 的集群** |
|---|---|---|
| **不写** 该字段 | 自动填入默认 SC 的名字 | 保持 `""` |
| **写 `""`**（空字符串） | **保持 `""`** —— 明确拒绝默认 SC | 保持 `""` |
| 写 `manual` | `manual` | `manual` |

### 机制：默认化（defaulting）

**PVC 的 `storageClassName` 是"两段式"的 —— 先默认化，再匹配。**

```
不写 storageClassName 时：
    ├─ 集群里【有】默认 SC → 控制器自动把默认 SC 的名字填进去
    └─ 集群里【没有】默认 SC → 保持为空 ""
```

**所以"不写"不是一个固定结果，它取决于集群状态。**

**"不写"和"写 `""`"是完全不同的意思**：

- **不写** = 交给默认化，有默认 SC 就用
- **`""`** = **明确拒绝**动态供给，只走静态绑定（找 `storageClassName` 同为空的 PV）

### ★ 一个必须纠正的直觉：有默认 SC 时不会 Pending

**很多人（包括我）第一反应是**："PVC 不写 SC，集群有默认 SC，那它会被填成 `nfs-client`，找不到现成的 PV → Pending。"

**错。** 正确结果分三种：

| 集群状态 | PVC 的 SC 最终值 | 实际结果 |
|---|---|---|
| 无默认 SC | `""` | **Pending**（找不到 SC 为空的 PV） |
| **有默认 SC + provisioner 正常** | `nfs-client` | **Bound** —— 但绑的是**动态新建的 PV** |
| 有默认 SC + provisioner 挂了 | `nfs-client` | **Pending**（在等 provisioner） |

**漏掉的关键一步是「静态找不到就动态创建」**：

```
① 默认化 → storageClassName = "nfs-client"
② 找不到现成的静态 PV
③ ★ 但 PVC 有 storageClassName，且这个 SC 存在
   → 走【动态供给】流程，通知 provisioner
④ provisioner 创建一个新 PV（storageClassName: nfs-client）
⑤ 新 PV 与 PVC 绑定 → Bound
```

> **PVC 的最终归宿不是"必须绑到某个静态 PV"，而是"先找静态的，找不到就让 provisioner 造一个"。**

### 由此产生的经典坑

```
管理员：我建了个 pv-day1（manual），1Gi，等着人用
开发者：我写个 PVC 要 500Mi，没写 storageClassName
集群：有个默认 SC nfs-client
        ↓
结果：PVC 绑到了动态创建的新 PV
     你的 pv-day1 一直 Available 地闲置着，永远没人用
```

**"我建的 PV 为什么没人用"** —— 根因就是默认 SC 把 PVC "抢"走了。

**解法两种**：

```yaml
spec:
  storageClassName: ""        # 方式 A：明确拒绝默认 SC，只走静态
  # storageClassName: manual  # 方式 B：和手工 PV 对上名字
```

---

## 五、三个关键属性（考纲明确要求）

### ① `accessModes` —— 是"能力声明"，不是"访问控制"

| 模式 | 含义 |
|---|---|
| `ReadWriteOnce` (RWO) | 只能被**一个节点**读写 |
| `ReadOnlyMany` (ROX) | 可以被**多个节点**只读 |
| `ReadWriteMany` (RWX) | 可以被**多个节点**读写 |
| `ReadWriteOncePod` (RWOP) | 只能被**一个 Pod** 读写（1.22+） |

**两个关键细节**：

1. **它是给调度器看的，不是给文件系统看的**
   写了 RWO，**K8s 做的事是"确保所有用这个卷的 Pod 调度到同一个节点"** —— 靠调度约束满足，不是靠拒绝挂载。
   底层存储真的支持多挂（比如 NFS），即使声明 RWO，技术上多节点也能挂 —— **但调度器会拦着不让 Pod 分到多节点。**

2. **RWO vs RWOP 的区别**：`RWO` 限制在**节点**级（同节点多 Pod 可共享）；`RWOP` 限制在 **Pod** 级（同节点不同 Pod 也不行）。

### ② `reclaimPolicy` —— 删了 PVC 之后 PV 会怎样

| 策略 | 删掉 PVC 后 | 谁是默认 |
|---|---|---|
| `Retain` | **PV 保留，状态变 `Released`**，数据还在但无法被新 PVC 绑定 | **静态 PV 默认** |
| `Delete` | PV 对象和底层存储**一起删掉** | **动态供给默认** |
| `Recycle` | **已废弃**（1.26 起不推荐） | — |

**`Retain` 的坑**：`Released` 状态的 PV **不能直接给新 PVC 用**，必须手工清掉 `claimRef`：

```bash
kubectl patch pv <pv-name> -p '{"spec":{"claimRef": null}}'
```

### ③ `volumeMode`

| 取值 | 含义 |
|---|---|
| `Filesystem`（默认） | 挂载成目录，Pod 里看到的是文件系统 |
| `Block` | 把裸块设备给 Pod，Pod 里看到的是 `/dev/xxx` |

---

## 六、实验一：绑定匹配（★ 先预测再实测）

### 准备 NFS 目录

```bash
mkdir -p /mnt/nfs && mount -t nfs 192.168.0.241:/K8SStorage /mnt/nfs
mkdir -p /mnt/nfs/pv-day1 && chmod 777 /mnt/nfs/pv-day1
umount /mnt/nfs && echo '✅ NFS 目录就绪'
```

### 提交 PV（`storageClassName: manual`）+ PVC（**不写**该字段）

```yaml
# PV
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-day1
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  nfs:
    server: 192.168.0.241
    path: /K8SStorage/pv-day1
---
# PVC —— 注意：故意不写 storageClassName
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-day1
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 500Mi
```

### 实测结果：**Pending**

```
NAME       STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pvc-day1   Pending                                      manual         8s
```

**原因**：集群里**没有默认 SC**，所以 PVC 的 `storageClassName` 最终解析为 `""`。而 PV 是 `manual` —— `""` ≠ `"manual"`，**第一道门槛就卡住了**。

**核心结论：绑定是精确字符串匹配，`""` 和 `"manual"` 是两个不同的值，没有"通配"的意思。**

验证 PVC 实际生效的值（不是你写的，是默认化之后的）：

```bash
kubectl get pvc -o custom-columns='NAME:.metadata.name,SC:.spec.storageClassName'
```

---

## 七、实验二：修正后绑定成功

⚠️ **PVC 的 `storageClassName` 是「不可变字段」，不能 patch，必须删了重建。**

```bash
kubectl delete pvc pvc-day1
# 重建时加上 storageClassName: manual
```

### 实测结果：**Bound**

```bash
kubectl get pv pv-day1
# NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM              STORAGECLASS
# pv-day1   1Gi        RWX            Retain           Bound    default/pvc-day1   manual
```

**对比实验一，差的只有 PVC 里那一行。**

### 验证它真的能用（整链路铁证）

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-day1
spec:
  containers:
    - name: app
      image: nginx:alpine
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: pvc-day1
```

```bash
kubectl exec pod-day1 -- sh -c 'echo "hello from k8s" > /data/test.txt'

# ★ 去 NFS 服务器上验证
mount -t nfs 192.168.0.241:/K8SStorage /mnt/nfs
cat /mnt/nfs/pv-day1/test.txt      # → hello from k8s
ls -ln /mnt/nfs/pv-day1/test.txt   # 属主 nobody（65534）
umount /mnt/nfs
```

**容器里写的文件出现在 NFS 服务器上** —— 这是整条链路（Pod → PVC → PV → NFS）打通的直接证据。

**同时暴露了 `root_squash`**：容器里以 root（uid 0）写的文件，在 NFS 上属主变成 `nobody`（uid 65534）。

---

## 八、实验三：`Retain` 的后果

```bash
kubectl delete pod pod-day1     # 顺序不能反：先删 Pod，再删 PVC
kubectl delete pvc pvc-day1
```

### 实测结果：PV 变 `Released`，数据完好

```
NAME      CAPACITY   RECLAIM POLICY   STATUS     CLAIM   STORAGECLASS
pv-day1   1Gi        Retain           Released           manual
```

**`Released` 的含义**：PVC 没了，但 PV 里还留着绑定关系的"回执"：

```yaml
spec:
  claimRef:
    kind: PersistentVolumeClaim
    name: pvc-day1
    namespace: default
    uid: 8b7a3c1d-...    # ★ 重点在这一行
```

### 为什么不让它自动变回 `Available`（安全闸）

设想如果一删 PVC 就立刻 `Available`：

```
你删掉 PVC（可能只是想重建、或者误删）
   ↓ PV 立刻 Available
别人的 PVC 恰好要存储 → 秒绑上
   ↓ 新 Pod 起来
看到的是你的旧数据，甚至还往上写
```

**你的数据被覆盖了，而且没有任何提示。**

所以 K8s 的选择是：**回收必须由管理员显式确认。**

### ★ 一个延伸问题：直接建同名 PVC 能绑上吗

**不能。** 因为 `claimRef.uid` 记的是**旧 PVC 的 uid**，删了重建的 PVC uid 必然不同。**名字一样没用 —— K8s 认 uid，不认名字。**

---

## 九、实验四：让 `Released` 的 PV 重获新生

```bash
kubectl patch pv pv-day1 -p '{"spec":{"claimRef": null}}'
kubectl get pv pv-day1     # → Available
```

然后新建 PVC 绑定它：

```bash
# pvc-day1-reuse，storageClassName: manual
```

### 实测结果：绑定成功，且**旧数据还在**

```
NAME             STATUS   VOLUME    CAPACITY   ACCESS MODES   STORAGECLASS
pvc-day1-reuse   Bound    pv-day1   1Gi        RWX            manual

$ cat /mnt/nfs/pv-day1/test.txt
hello from k8s          ← 数据完好
```

**这就是 `Retain` 的意义：PV 回收了，数据没丢。**

### 三种回收方案对照

| 方案 | PV 对象 | 底层存储 | 要不要人工介入 |
|---|---|---|---|
| **`Retain`** | 保留，`Released` | 原样保留 | ✅ 要清 `claimRef` |
| **`Delete` + 归档** | 删除 | **改名 `archived-*`** | ✅ 要清归档目录 |
| **`Delete` + 真删** | 删除 | **真删** | ❌ 全自动 |

一句话记忆：**`Retain` 把"回收"留给管理员；`Delete` 交给 K8s 全自动。前者安全但麻烦，后者省事但危险。**

---

## 十、踩坑实录

### 坑 1：PVC 的 `storageClassName` 不可变

想改 SC 名字只能 `delete` + 重建 PVC。`kubectl edit` 会被拒绝。

### 坑 2：绑定失败**不报错**

PVC 一直 `Pending`，`kubectl get` 只显示状态，**没有任何提示告诉你是哪一项没对上**。必须 `describe` 看 Events。

### 坑 3：`""` 和"不写"是两回事

- 不写 → 交给默认化（有默认 SC 就填）
- 写 `""` → 明确拒绝默认 SC

在**有默认 SC 的集群**里，这两者的结果完全不同。

### 坑 4：`Released` 状态不能直接复用

很多人删了 PVC 想重新绑同一个 PV，发现绑不上还以为是"PV 坏了"。实际是 `claimRef` 还占着，要手工清。

### 坑 5：`accessModes` 是"声明"不是"控制"

NFS 明明支持 RWX，你写 RWO 也不会报错 —— **K8s 只是用调度约束来"遵守"你的声明**。

### 坑 6：NFS 的 `root_squash`

容器里以 root 写的文件，在 NFS 上属主变 `nobody:nobody`（uid 65534）。

**现在没踩坑是因为目录是 777。** 如果手工建了 755 的子目录，容器里的 root 就会被拒（因为 NFS 把它当 nobody）。

**排查这类问题的关键动作**：`ls -ln`（用 `-n` 看**数字 uid**）。看到 65534 就知道是 root_squash 干的，不是 K8s 的问题。

### 坑 7：NFS 没有配额

PV 上写的 `capacity: 1Gi` 只是**账面声明**（用于绑定匹配和调度判断），**实际那个目录不限制大小** —— 申请 1Gi 写 10Gi 也不会被拦，能一直写到 NFS 盘满。

要真实配额得换 Ceph / 云盘 / 带 quota 的 NAS。

---

## 十一、复习自测题

**基础**

1. PV 和 PVC 的作用域分别是什么？为什么 PV 是集群级而 PVC 是命名空间级？
2. `emptyDir` 和 PVC 在"容器重建 / Pod 删除"两种场景下分别表现如何？

**绑定（最容易错的一节）**

3. PVC 绑定 PV 要满足哪四个条件？哪一个是"精确相等"、其他是"宽松"？
4. PVC **不写** `storageClassName` 和写 `storageClassName: ""`，在**有默认 SC** 的集群里结果一样吗？
5. 集群有默认 SC `nfs-client`，你手工建了个 `storageClassName: manual` 的 PV。一个新 PVC 不写 SC 会绑到哪？你的 PV 会怎样？
6. `storageClassName` 能改吗？要改怎么办？

**回收**

7. `Retain` 和 `Delete` 分别在什么时候是默认值？
8. 删了 PVC 后 PV 变 `Released`，能直接给新 PVC 用吗？要先做什么？
9. 为什么 K8s 不让 `Released` 的 PV 自动变回 `Available`？
10. `accessModes: ReadWriteOnce` 限制的粒度是节点还是 Pod？和 `ReadWriteOncePod` 差在哪？

---

## 十二、速记卡片

```
【四层抽象】
容器可写层 → Volume → PV/PVC → StorageClass
每一层都把数据往更长寿的地方挪

【绑定四条件】
storageClassName 严格相等 ← 唯一"精确匹配"
accessModes / capacity / selector ← PV 可以更强

【SC 三态】
不写 → 默认化（有默认 SC 就填，没有就空）
""   → 明确拒绝动态供给，只走静态
名字 → 精确匹配

【关键是"静态找不到就动态创建"】
PVC 不是"必须绑到某个静态 PV"

【回收三态】
Retain → Released，数据留，要清 claimRef
Delete + archiveOnDelete 默认 → 改名 archived-*
Delete + archiveOnDelete=false → 真删

【认 uid 不认名字】
claimRef.uid 决定能不能被认领

【accessModes 是给调度器看的】
K8s 靠调度约束遵守声明，不是靠拒绝挂载

【NFS 两个坑】
root_squash → 属主变 nobody（用 ls -ln 看数字 uid）
无配额 → capacity 只是账面数字
```
