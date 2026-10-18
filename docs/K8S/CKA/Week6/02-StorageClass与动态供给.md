---
title: "StorageClass 与动态供给"
sidebar_position: 2
---

# StorageClass 与动态供给

> 实验环境：三节点 K8s v1.36.3 + Calico，NFS 后端 `192.168.0.241:/K8SStorage`
> 组件：`nfs-subdir-external-provisioner:v4.0.2`（部署在 `default` ns，跑在 k8s-node01）
> 本篇所有输出均为真实集群实测。

---

## 核心认知

**StorageClass 是"说明书"，Provisioner 是"工人"，两者靠一个字符串名字对上暗号。**

```
StorageClass  = 说明书    ← 里面没有存储实体，只有"谁去干、怎么干"
Provisioner   = 工人      ← 一个 Pod，真的在干活
NFS 共享目录   = 工地      ← 存储空间本身
```

**★ K8s 不会主动"派活"给 provisioner。** 是 provisioner 自己 watch 所有 PVC，比对名字后发现"这个活儿是找我的"，才动手。

**推论**：

- Provisioner 挂了 → 没人拉活 → PVC 一直 Pending（K8s 干着急但没办法）
- Provisioner 重启 → 它会重新 watch，**把之前挂着的活一起补上**

**这也是为什么 provisioner 需要 RBAC 权限** —— 它要能 watch PVC、创建 PV、更新 PVC 状态。它是一个有权限的普通 Pod，不是特权组件。

---

## 一、三个"provisioner"必须分清

**"provisioner" 这个词在三个地方出现，含义完全不同：**

| 出现位置 | 是什么 | 本例中的值 |
|---|---|---|
| **SC 里的 `provisioner` 字段** | 一个**字符串标识符**（"我叫谁"） | `k8s-sigs.io/nfs-subdir-external-provisioner` |
| **Provisioner 程序** | 一段**代码**，打包成容器镜像 | `nfs-subdir-external-provisioner:v4.0.2` |
| **Provisioner Pod** | 部署到集群里跑起来的**工作负载** | 一个 Deployment，1 个 Pod |

**用运维视角类比**：

| K8s 概念 | 机房类比 |
|---|---|
| **StorageClass** | 工单模板：「这类活找哪个承包商，参数怎么填」 |
| **`provisioner` 字段** | 模板上写的**承包商名字** |
| **Provisioner Pod** | 承包商的**驻场人员**（真在机房里干活，有工牌有权限） |
| **NFS 共享目录** | 工地 |
| **PV** | 一块编好号的地（`/K8SStorage/xxx`） |
| **PVC** | 用户的用地申请单 |

**"承包商名字"和"驻场的人"必须对上** —— 名字写错了或者人没来，工单就一直挂着。

---

## 二、StorageClass 字段全表

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-client
provisioner: k8s-sigs.io/nfs-subdir-external-provisioner   # ★ 谁来创建（字符串标识）
parameters:                                                 # ★ 传给插件的参数
  server: 192.168.0.241
  path: /K8SStorage
  archiveOnDelete: "false"
reclaimPolicy: Delete                                       # 动态供给默认 Delete
volumeBindingMode: Immediate                                # 或 WaitForFirstConsumer
allowVolumeExpansion: true                                  # 允许 PVC 扩容
```

| 字段 | 作用 |
|---|---|
| `provisioner` | 用哪个插件创建存储（**必须和 Pod 的 `PROVISIONER_NAME` 一致**） |
| `parameters` | 传给插件的参数，**每个插件自己解释**（这是它的"私有扩展区"） |
| `reclaimPolicy` | 回收策略，**动态创建的 PV 从这里继承** |
| `volumeBindingMode` | 绑定时机（`Immediate` / `WaitForFirstConsumer`） |
| `allowVolumeExpansion` | 能不能扩容 |

### 设成默认 StorageClass

```bash
kubectl patch storageclass nfs-client -p \
  '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

**之后 PVC 不写 `storageClassName` 就会用它。**

### ★ SC 是"改不动"的对象

| 字段 | 可变？ |
|---|---|
| `provisioner` | ❌ |
| **`parameters`** | ❌ |
| `reclaimPolicy` | ❌ |
| `volumeBindingMode` | ❌ |
| `allowedTopologies` | ❌ |
| **`allowVolumeExpansion`** | ✅ **唯一能改的** |

**为什么要这么严**：SC 被历史 PV 引用时，参数是**复制**过去的：

```
SC nfs-client  ← 已被 100 个已存在的 PV 引用（各自复制了当时的参数）
```

允许改 `parameters` 的话：老的 100 个 PV 不会跟着变（值在创建时固化了），**但新 PVC 会用新规则** —— 同一个 SC 名下的存储行为就分裂了，而且没有地方能看出这个分裂。

**代价**：删 SC 会短暂影响新 PVC 的供给（已绑定的 PV/PVC 不受影响）。

---

## 三、Provisioner 部署四件套

| 组件 | 作用 |
|---|---|
| **ServiceAccount** | 给它一个身份 |
| **ClusterRole** | 定义权限（**必须用 ClusterRole，因为 PV 是集群级资源**） |
| **ClusterRoleBinding** | 把身份和权限绑起来 |
| **Deployment** | 本体（单副本） |

### 权限清单（逐条都有理由）

```yaml
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get","list","watch","create","delete"]      # 创建/删除 PV
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get","list","watch","update"]              # watch PVC，写回状态
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get","list","watch"]                       # 读 SC 的参数
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create","update","patch"]                  # 发事件（你在 describe 里看到的）
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get","list","watch","create","update","patch"]  # ★ leader election
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get","list","watch"]                       # 读节点信息（拓扑相关）
```

**注意 `endpoints` 那一条** —— 不是用来转发流量的，是**用来抢 leader 锁的**（见下节）。

### Deployment 的关键配置

```yaml
spec:
  replicas: 1
  strategy:
    type: Recreate              # 避免滚动更新时新旧 Pod 同时在线
  template:
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: m.daocloud.io/registry.k8s.io/sig-storage/nfs-subdir-external-provisioner:v4.0.2
          env:
            - name: PROVISIONER_NAME       # ★ 必须和 SC 里一字不差
              value: k8s-sigs.io/nfs-subdir-external-provisioner
            - name: NFS_SERVER
              value: 192.168.0.241
            - name: NFS_PATH
              value: /K8SStorage
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
      volumes:
        - name: nfs-client-root
          nfs:                          # ★ Pod 自己挂载 NFS
            server: 192.168.0.241
            path: /K8SStorage
```

**两个必须理解的设计**：

1. **`PROVISIONER_NAME` 的值必须和 SC 里的 `provisioner` 字段一字不差** —— 这就是"对上暗号"。写错一个字符，工人就永远等不到活干（**而且不报错，只是沉默**）。
2. **Pod 自己挂载了 NFS**（→ `/persistentvolumes`）—— 因为它要在那个目录里 `mkdir`。**所以 provisioner 跑在哪个节点，那个节点就必须能访问 NFS。**

---

## 四、为什么必须单副本 + Leader Election

**多副本会出问题**：两个 Pod 同时看到同一个 PVC 都要处理 → 一个 mkdir 了 `default-pvc-x-pvc-aaa`，另一个又建一遍 → **卷重复创建、数据错乱**。

代码里有 **leader election** 兜底（只有抢到锁的才干活），官方也建议保持单副本。

**多副本的意义只在"故障快速切换"，不在"并行加速"** —— 它天生是串行工作的角色。

### 实测日志里的 leader election

```
attempting to acquire leader lease default/k8s-sigs.io-nfs-subdir-external-provisioner...
successfully acquired lease default/k8s-sigs.io-nfs-subdir-external-provisioner
```

**锁存在一个同名的 Endpoints 对象上**：

```
Event: Kind:"Endpoints" Name:"k8s-sigs.io-nfs-subdir-external-provisioner"
       reason: 'LeaderElection' ... became leader
```

**这就解释了 RBAC 里为什么必须有 `endpoints` 权限。**

---

## 五、实验一：工人缺席时是什么样

**场景**：SC 已经建好（说明书有了），但集群里没有任何 provisioner。

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-dyn-test
spec:
  accessModes: [ReadWriteMany]
  storageClassName: nfs-client
  resources:
    requests:
      storage: 1Gi
```

### 实测结果：Pending

```
NAME           STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pvc-dyn-test   Pending                                      nfs-client     5s
```

### ★ Events 那句话是排障标准话术

```
Type    Reason                From                          Message
Normal  ExternalProvisioning  persistentvolume-controller   Waiting for a volume to be created
                                                            either by the external provisioner
                                                            'k8s-sigs.io/nfs-subdir-external-provisioner'
                                                            or manually by the system administrator.
                                                            If volume creation is delayed, please verify
                                                            that the provisioner is running and
                                                            correctly registered.
```

**逐句拆**：

| 片段 | 含义 |
|---|---|
| `ExternalProvisioning` | K8s 判定"这是动态供给请求" |
| `either by the external provisioner 'xxx'` | **① 明确点名在等谁** → 去集群找有没有叫这个名字的 Pod |
| `or manually by the system administrator` | **② 第二条出路** → 自己手工建一个匹配的 PV 走静态绑定 |
| `verify that the provisioner is running and correctly registered` | **③ `running` = Pod 活着吗；`correctly registered` = 名字对上暗号了吗** |

**官方原文用的词就是 `registered`** —— 印证了"名字对上暗号"这个机制。

### ★ 两个容易误判的细节

**① 事件类型是 `Normal`，不是 `Warning`**

**`Normal` 不代表"正常"** —— 它只是在陈述一个状态（"我在等"）。很多人 `describe` 时只扫一眼有没有 Warning，看到全是 Normal 就以为没问题，结果卡在 Pending 上找不到方向。

**② `From: persistentvolume-controller`**

**这条事件是 K8s 自己发的，不是 provisioner 发的**（provisioner 压根不在）。是 K8s 在"催"，但没人接。

### 为什么不会回头去找静态 PV

**关键在"判定"这一步**：

```
PVC 找不到现成的静态 PV
+ 它指定了 storageClassName: nfs-client（一个真实存在的 SC）
   ↓
K8s 判定"这是动态供给请求"
   ↓
交给该 SC 声明的 provisioner 处理
   ↓
没有人认领这个名字 → 永远 Pending
```

**K8s 判断走静态还是动态，依据是 PVC 有没有指定 SC**：

- 指定了 SC → 走动态（哪怕只是等）
- SC 是 `""` → 只走静态

---

## 六、实验二：部署 provisioner

**部署后实测**：

```
NAME                                      READY   STATUS    RESTARTS   AGE     IP              NODE
nfs-client-provisioner-7465696b8b-7k698   1/1     Running   0          5h23m   10.244.85.244   k8s-node01
```

**Pod 跑在 k8s-node01** —— 所以那个节点必须能访问 NFS（Pod 自己挂载了它）。

### ★ 它会自动补上之前挂着的活

```
provision "default/pvc-dyn-test" class "nfs-client": started
```

**注意：这个 PVC 是实验一（5 小时前）就建好的，不是新提交的。**

原因就是核心机制：**provisioner 是"自己拉活干"** —— 它 watch 的是**全量 PVC 列表**，不只盯着"新来的"。所以它一上线，就会把之前积压的任务一起处理掉。

---

## 七、实验三：目录凭空出现

### 实测结果

```
PVC      pvc-dyn-test  →  Bound
         绑到 PV: pvc-3534f78e-da86-4052-877f-bfd89d89e2db

PV（自动生成）
NAME                                       CAP   SC           RECLAIM   MODE         NFS_PATH
pvc-3534f78e-da86-4052-877f-bfd89d89e2db   1Gi   nfs-client   Delete    Filesystem   /K8SStorage/default-pvc-dyn-test-pvc-3534f78e-...

NFS 服务器
default-pvc-dyn-test-pvc-3534f78e-da86-4052-877f-bfd89d89e2db   ← ★ 凭空出现
```

**一个 8 行的 PVC，四件事自动发生**：建目录、造 PV、绑定、回收策略继承。

### 命名格式

```
default-pvc-dyn-test-pvc-3534f78e-da86-4052-877f-bfd89d89e2db
   ↑ns      ↑PVC名            ↑PV名
```

**NFS 上的目录名 = `<namespace>-<pvc名>-<pv名>`**（对 NFS 来说，"创建存储"就是 `mkdir` 一个子目录）。

### 日志逐行读

```
attempting to acquire leader lease ...            ← ① 抢 leader
successfully acquired lease                       ← ② 抢到了
Starting provisioner controller ...               ← ③ 开始 watch
provision "default/pvc-dyn-test" class "nfs-client": started    ← ④ 认领这个活
provision ...: volume "pvc-3534f78e-..." provisioned            ← ⑤ 建完目录 + PV
Trying to save persistentvolume "pvc-3534f78e-..."              ← ⑥ 写 PV 对象
persistentvolume "pvc-3534f78e-..." saved
provision ...: succeeded                                        ← ⑦ 收工
```

**⑤ 是核心** —— 它把三件事一起做了：**在 NFS 上 mkdir → 创建 PV 对象 → 写入绑定关系**。

**⑥⑧ 对应事件侧**：`Provisioning` → `ProvisioningSucceeded`（你在 `describe pvc` 里看到的那两条）。

### ★ "创建存储"对不同后端含义完全不同

| 后端 | "创建存储"实际做什么 |
|---|---|
| **NFS** | 在共享目录下 `mkdir` 一个子目录 |
| **云盘**（EBS 等） | 调云厂商 API **创建一块云盘**，再挂到节点 |
| **Ceph RBD** | 在 Ceph 池里**创建一个 image** |
| **local-path**（Rancher） | 在节点本地 `mkdir` 一个目录 |
| **iSCSI** | 创建一个 LUN |

**这正是 provisioner / CSI 抽象的价值** —— 上层（PVC）完全不用知道底层是 mkdir 还是调 API。

---

## 八、回收策略实测：归档 vs 真删

### 第一次实验：SC 未设 `archiveOnDelete`（默认 → 归档）

删掉 PVC 后实测：

```
PVC   → 删除（NotFound）
PV    → 删除
NFS   → 目录还在！但改名为：
        archived-default-pvc-dyn-test-pvc-3534f78e-da86-4052-877f-bfd89d89e2db
        里面的 marker.txt 完好无损
```

### ★ 核心认知：`reclaimPolicy` 只是 K8s 层的语义

```
reclaimPolicy: Delete
   = "这个 PV 对象和它背后的底层卷，一起回收掉"
        ↑ 只是【指令】，不是【动作】
```

**指令发下去之后，具体怎么回收是 provisioner 自己实现的。** 这就是 SC 里 `parameters` 存在的意义 —— **它是"传给插件的参数"，插件自己解释。**

**对不同存储，"Delete 回收"的动作完全不同**：

| 存储 | 实际做什么 |
|---|---|
| NFS | 归档或 `rm -rf` 一个目录 |
| 云盘 | 调云 API 把云盘 detach 并删除（通常有几天回收站缓冲） |
| Ceph RBD | 删掉 image |

> **"`Delete` 就等于数据消失"是个危险的简化认知。**

### ★ 坑：日志在"说谎"

```
delete "pvc-3534f78e-...": started
delete "pvc-3534f78e-...": volume deleted      ← 说删了
delete "pvc-3534f78e-...": persistentvolume deleted
delete "pvc-3534f78e-...": succeeded
```

**三行都说删了，但实际只是改名。** provisioner 内部把"归档"视为"删除"的完成态 —— **对它来说这件事已经做完了，日志不区分。**

**排障启示**：涉及存储的问题，**日志说成功不等于底层真如你所想**，最终还得去看实际存储（`ls` 那一下）。

### 第二次实验：`archiveOnDelete: "false"`（→ 真删）

```bash
kubectl delete sc nfs-client     # parameters 不可变，只能删了重建
# 重建时加 archiveOnDelete: "false"
```

实测结果：

```
NFS 列表：
archived-default-pvc-dyn-test-pvc-3534f78e-...   21:26   ← 第一次实验的归档残留
pv-day1
@Recycle

default-pvc-del-test-pvc-e3f0b10f-...            ← ★ 彻底消失
archived-default-pvc-del-test-...                ← ★ 也没有产生
```

**`archiveOnDelete: "false"` 生效了 —— 真删成功。**

### ★ 怎么辨别归档目录属于谁

**看名字里的 PV 名 + 时间戳，不要只看 `archived-` 前缀。**

| 实验 | PVC | SC 参数 | 删后目录 |
|---|---|---|---|
| 第一次 | `pvc-dyn-test` | 未设（默认归档） | `archived-default-pvc-dyn-test-pvc-`**`3534f78e`**`-...` |
| 第二次 | `pvc-del-test` | `archiveOnDelete: "false"` | **不存在**（真删） |

**`archived-` 后面那串 PV 名是唯一标识。** 看到 NFS 上一堆 `archived-*` 时，靠 PV 名 + 时间戳判断"这是谁的归档"。

### `archiveOnDelete` 参数

| 取值 | 删 PVC 时的动作 |
|---|---|
| **不写** / `"true"`（**默认**） | **把目录重命名为 `archived-<原名>`** |
| `"false"` | **真删（`rm -rf`）** |

**这个默认值是刻意的保守设计**：误删 PVC 时数据不至于永久消失，管理员还能在归档目录里捞回来。

**代价**：**NFS 会越用越满，需要有人定期清理**（自动化的代价是有人得扫垃圾）。

**生产建议**：要么配定时清理，要么把参数设成 `false` 并靠备份兜底。

---

## 九、SC 的不可变性怎么验证（一个踩坑）

**错误的验证方法**：

```bash
kubectl patch sc nfs-client -p '{"parameters":{"archiveOnDelete":"false"}}'
# → storageclass.storage.k8s.io/nfs-client patched (no change)
```

**`patched (no change)` 说明这次的 patch 值**和当前值相同**，kubectl 比较后**根本不发更新请求** → **不会触发不可变性检查。**

**所以"no change"不能用来验证字段可变性。**

**正确的验证方法 —— patch 成一个不同的值**：

```bash
kubectl patch sc nfs-client -p '{"parameters":{"archiveOnDelete":"true"}}'
# → The StorageClass "nfs-client" is invalid:
#   parameters: Forbidden: updates to parameters are forbidden
```

**✅ 实测确认：SC 的 `parameters` 确实不可变。**

---

## 十、NFS 方案的局限（务必知道）

### ① 协议无配额机制

PV 上的 `capacity` 只是**账面声明**（用于绑定匹配和调度判断），**实际目录不限制大小**。

**后果**：两个 PVC 各申请 20G，NFS 只有 30G —— K8s 会认为都够（各自看各自的声明），实际写起来就爆。

### ② `root_squash`

provisioner 以 root 在 NFS 上 `mkdir`，**被服务端 squash 成 `nobody`（uid 65534）**：

```
drwxrwxrwx 2 65534 65534 4096 ... default-pvc-dyn-test-pvc-3534f78e-...
```

**所有动态创建的目录天生是 nobody 属主。** 如果哪天遇到"Pod 写不进 PVC"，先看目录权限是不是被手工改成 755 了。

### ③ NAS 侧零适配

**NFS 是几十年的标准协议，Linux 内核原生支持。** kubelet 挂载时执行的就是一条普通命令：

```bash
mount -t nfs 192.168.0.241:/K8SStorage/xxx /var/lib/kubelet/pods/.../volumes/...
```

**跟手工在服务器上敲的 mount 一模一样。NAS 侧完全不需要知道"K8s"这个词。**

**三层看"零适配"**：

| 层次 | NAS 侧 | K8s 侧 |
|---|---|---|
| ① 挂载（Pod 用 NFS 卷） | **零适配** | 零配置（kubelet 内建支持） |
| ② 动态供给（PVC 自动建目录） | **零适配** | **要装 provisioner** |
| ③ 高级能力（快照/配额/克隆） | — | ❌ **NFS 方案给不了** |

**第 ③ 层是 NFS 的天花板**：快照需要 CSI driver，NFS provisioner 不支持；配额协议层面没有；卷克隆不支持。

**所以 NFS 在 K8s 里的定位是"够用就好"** —— 简单、零适配、能跑，但缺企业级能力。

---

## 十一、踩坑实录

### 坑 1：`PROVISIONER_NAME` 和 SC 的名字不一致（静默失败）

**不报错，只是永远 Pending。** 唯一的线索是 Events 里那句 `correctly registered`。

### 坑 2：provisioner Pod 所在节点必须能访问 NFS

因为 Pod 自己挂载了 NFS。**换节点调度时如果那个节点访问不到 NFS，供给就失败。**

### 坑 3：`kubectl patch sc` 返回 `patched (no change)`

**这不代表字段可变** —— 只是值相同，kubectl 没发更新请求。要验证不可变性必须改成**不同的值**。

### 坑 4：`reclaimPolicy: Delete` 不等于数据被真删

对 NFS provisioner 来说，默认行为是**归档**（改名 `archived-*`）。要真删必须显式设 `archiveOnDelete: "false"`。

### 坑 5：日志说 `volume deleted`，实际可能只是改名

**排障不能只看日志字面**，涉及存储一定要 `ls` 实际存储确认。

### 坑 6：归档目录会累积

`archiveOnDelete` 默认 `true` → 每次删 PVC 都留一份归档 → **NFS 越用越满**。需要定期清理或改参数。

### 坑 7：SC 删了重建会短暂影响新 PVC

已绑定的 PV/PVC 不受影响，但**正在等待供给的 PVC 可能会失败或延迟**。改 SC 前先确认没有 Pending 的 PVC。

### 坑 8：provisioner 日志时间戳是 UTC

```
I1011 05:38:42    ← UTC
本地实际：13:38    ← 差 8 小时
```

排障对时间线时容易困惑。

### 坑 9：NFS 目录缓存导致"看不到刚创建的目录"

NFS 客户端会缓存目录属性。**服务端刚创建的目录，客户端 `ls` 可能看不到**（默认缓存几秒到几十秒）。写脚本时要加 `sleep` 或用精确路径确认，别急着判定"失败"。

### 坑 10：通配符写文件路径的陷阱

```bash
echo "data" > /mnt/nfs/default-pvc-*/marker.txt
```

**如果通配符没匹配到任何目录**，bash 会原样保留这个字符串，报 `No such file or directory` —— **这个报错和"目录不存在"看起来一样，但原因完全不同。**

**改用精确路径**：

```bash
PVNAME=$(kubectl get pvc <name> -o jsonpath='{.spec.volumeName}')
DIR="/mnt/nfs/default-<pvc名>-$PVNAME"
```

---

## 十二、复习自测题

**概念**

1. SC 里的 `provisioner` 字段、Provisioner 程序、Provisioner Pod，三者分别是什么？
2. 为什么 K8s 不主动"派活"给 provisioner？这个设计导致"provisioner 挂了"时会发生什么？
3. 为什么 provisioner 的 Pod 自己也要挂载 NFS？
4. 为什么建议保持单副本？多副本会出什么问题？leader election 的锁存在哪？

**动态供给**

5. PVC 有 `storageClassName` 但静态找不到 PV 时，K8s 会做什么？（这是个"反直觉"的点）
6. provisioner 起来后，会不会处理"它启动之前"建的 PVC？为什么？
7. 动态创建的 PV 上，`persistentVolumeReclaimPolicy` 的值是从哪来的？
8. NFS 上生成的目录名格式是什么？由哪三部分构成？

**回收**

9. `archiveOnDelete` 缺省时是什么行为？设成 `"false"` 呢？
10. 日志显示 `volume deleted`，能不能确认底层数据已被删除？怎么确认真相？
11. `kubectl patch sc` 返回 `patched (no change)`，能得出什么结论？怎么正确验证 SC 的不可变性？

---

## 十三、速记卡片

```
【三个 provisioner】
SC 里的 provisioner 字段 = 名字（字符串）
Provisioner 程序        = 代码（镜像）
Provisioner Pod         = 实体（干活的）

【对上暗号】
SC.provisioner  ==  Pod 的 PROVISIONER_NAME
不一致 → 永远 Pending，不报错

【工人自己拉活】
K8s 不派活，provisioner watch 全量 PVC
所以它上线后会补上之前积压的任务

【Events 那句话】
either by external provisioner 'xxx'  → 找出路 1：把人请来
or manually by administrator          → 出路 2：手工建 PV
verify running and correctly registered → 检查活着没 + 名字对不对
注意是 Normal 不是 Warning

【目录命名】
<namespace>-<pvc名>-<pv名>

【reclaimPolicy 只是指令】
Delete ≠ 数据消失
实际动作由 provisioner 实现
NFS 默认：改名 archived-*
NFS 真删：archiveOnDelete: "false"

【日志不区分】
volume deleted 可能只是改名 → 必须 ls 实际存储

【SC 不可变】
parameters / provisioner / reclaimPolicy / volumeBindingMode 全不可改
只有 allowVolumeExpansion 能改
patch 出 "no change" 不代表可变（值没变就没发请求）

【NFS 局限】
无配额 → capacity 只是账面数字
root_squash → 目录属主 nobody（ls -ln 看 65534）
无快照/克隆 → 要企业能力得换 CSI
```

---

## 附：Week6 剩余计划

| Day | 主题 | 状态 |
|---|---|---|
| Day1 | PV / PVC 生命周期 | ✅ 本篇 |
| Day2 | StorageClass + 动态供给 | ✅ 本篇 |
| Day3 | 绑定模式（`WaitForFirstConsumer`）+ 扩容 | ⬜ |
| Day4 | StatefulSet + `volumeClaimTemplates` 深化 | ⬜ |
| Day5 | 存储排障 + 收口 | ⬜ |
