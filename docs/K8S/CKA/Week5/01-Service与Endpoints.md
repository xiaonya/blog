---
title: "Service 与 Endpoints"
sidebar_position: 1
---

# Service 与 Endpoints

> 集群：k8s-master / k8s-node01 / k8s-node02（VMware，Fedora，v1.36.3，CNI=Calico v3.29.1）

## 核心认知：Service 不是"转发器"，是"规则"

```
外部客户端 →  nodePort  →  port  →  targetPort（Pod 容器）
             (节点上开的洞)  (Service逻辑端口)  (容器实际监听)
```

- Service 的 ClusterIP **只是个虚拟 IP**，集群里找不到对应网卡，**没有任何进程在监听**
- **kube-proxy 运行在每个节点**，watch Service/EndpointSlice 变化，在本节点写 iptables DNAT 规则
- Service 真正干的活：把"目标 ClusterIP"改写成"某个真实 Pod IP"

```
你在 master 上 curl 192.168.10.12:31629
  → master 内核被 iptables 拦截（kube-proxy 写的规则）
  → DNAT：目标改写成 10.244.85.218:80（一个真在 node01 上的 Pod）
  → 包跨节点路由过去 → 200
```

**推论：`ss -tlnp | grep <nodePort>` 查不到任何进程** —— 它是 iptables 规则，不是 socket。

## 一、五种 Service 类型

| 类型 | ClusterIP | 谁能访问 | 一句话 | 用途 |
|---|---|---|---|---|
| **ClusterIP**（默认） | 有（VIP） | 仅集群内 | 集群内的门牌号 | 微服务互调 |
| **NodePort** | 有 + 节点端口 | 集群外 `NodeIP:30000-32767` | 每台机器开个后门 | 测试暴露 |
| **LoadBalancer** | 有 + 云 LB | 公网 | 找云厂商要 VIP | 云上生产入口 |
| **ExternalName** | **无 IP**，返回 CNAME | 集群内 | DNS 层跳转，不碰 Pod | 接外部服务 |
| **Headless**（`clusterIP: None`） | **None** | 集群内，DNS 返回**全部 Pod IP** | 不给门牌号，给你名单 | StatefulSet、DB 集群 |

**三条套娃规则（必记）**

1. NodePort **包含** ClusterIP（不是二选一）
2. LoadBalancer **包含** NodePort
3. 前三种都会生成 Endpoints；**ExternalName 完全不生成**（`get endpoints` 返回 NotFound）

## 二、命令式生成 YAML（CKA 主力姿势）

```bash
# 建靶子
kubectl create deployment web-svc --image=nginx:1.25 --replicas=3

# 先 dry-run 看 YAML，再真正创建
kubectl expose deployment web-svc --name=web-cip --port=80 --target-port=80 --type=ClusterIP --dry-run=client -o yaml
kubectl expose deployment web-svc --name=web-cip --port=80 --target-port=80 --type=ClusterIP

# NodePort（不写 --node-port 由系统在 30000-32767 随机分配）
kubectl expose deployment web-svc --name=web-np --port=80 --target-port=80 --type=NodePort

# Headless —— 关键参数 --cluster-ip=None
kubectl expose deployment web-svc --name=web-headless --port=80 --target-port=80 --cluster-ip=None

# 另一种命令式写法：--tcp=<port>:<targetPort>
kubectl create service clusterip web-odd --tcp=9999:80
```

**ExternalName 是唯一必须手写 YAML 的类型**（命令式生成不了）：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-ext
spec:
  type: ExternalName
  externalName: www.baidu.com
```

## 三、实测观察点

```bash
kubectl get svc -o wide
kubectl get endpoints
kubectl get endpointslices -l kubernetes.io/service-name=web-cip -o wide
kubectl get pods -l app=web-svc -o wide
```

实测结果：

```
NAME           TYPE        CLUSTER-IP       EXTERNAL-IP    PORT(S)        SELECTOR
web-cip        ClusterIP   10.111.19.171    <none>         80/TCP         app=web-svc
web-np         NodePort    10.103.225.30    <none>         80:31629/TCP   app=web-svc
web-headless   ClusterIP   None             <none>         80/TCP         app=web-svc
web-ext        ExternalName <none>          www.baidu.com  <none>         <none>

NAME           ENDPOINTS
web-cip        10.244.58.236:80,10.244.85.218:80,10.244.85.220:80
web-headless   10.244.58.236:80,10.244.85.218:80,10.244.85.220:80   ← 和 ClusterIP 完全一样
web-ext        (NotFound —— 对象根本不存在)
```

**三个观察结论**

1. Endpoints 的 IP 列表 = Pod IP 列表，数量与副本数一致
2. `PORT(S)` 列 NodePort 显示 `80:31629/TCP` = **`port:nodePort`**，**targetPort 不在这列显示**
3. **Headless 的 Endpoints 与 ClusterIP 完全相同** → 它的"特殊"不在 Endpoints 层，在 DNS 层

## 四、Endpoints vs EndpointSlice（5 点差异）

不是"新格式省略了重复子对象"，那是顺带的副作用。真实差异：

| 维度 | Endpoints | EndpointSlice |
|---|---|---|
| API 组 | `v1`（core） | `discovery.k8s.io/v1` |
| 分片 | 一个 Service 全部端点挤在**一个对象** | 默认每片最多 **100** 个，自动切片 |
| 状态 | 只有 `addresses` / `notReadyAddresses` **两态** | 每端点三态：`ready` / `serving` / `terminating` |
| 拓扑 | 无 | `nodeName` + `zone` + **`hints`**（支撑 `trafficDistribution: PreferClose`） |
| 地址类型 | 无 | `addressType`: IPv4 / IPv6 / FQDN |
| 端口位置 | 每条 endpoint 带 IP:port | **切片级 `ports` 字段**，全体共用 |

**为什么分片是本质动机**：5000 节点的集群，一次部署让 Endpoints 全量重写，曾把 apiserver 打趴。分片把"全量重推"变成"局部更新"。

**为什么要有 `terminating`**（解决滚动更新 502）：

```
Pod 收到 SIGTERM 要下线
  Endpoints 模型：直接挪进 notReadyAddresses → 流量瞬断 → 正在处理的请求 502
  EndpointSlice 模型：ready=false（不接新流量）
                     terminating=true, serving=true（在飞的请求继续放行）
                   → 优雅收尾
```

⚠️ **v1 Endpoints 在 v1.33+ 已废弃**，本集群 v1.36.3 执行 `kubectl get endpoints` 会输出 Warning。新写 YAML 不要再用 `kind: Endpoints`；写 RBAC 时 `resources` 用 `endpointslices`（组是 `discovery.k8s.io`）。

## 五、Headless 的本质：差异在 DNS 层

同一个 Deployment、同样 3 个 Pod，只换 Service 类型，`nslookup` 结果完全不同：

```
ClusterIP: web-cip
  web-cip.default.svc.cluster.local → 10.111.19.171        ← 1 个，虚拟 IP

Headless: web-headless
  web-headless.default.svc.cluster.local → 10.244.58.236   ← 3 个，真实 Pod IP
  web-headless.default.svc.cluster.local → 10.244.85.218
  web-headless.default.svc.cluster.local → 10.244.85.220
```

| | ClusterIP | Headless |
|---|---|---|
| Endpoints | 3 个 Pod IP | 完全相同 |
| Service IP | VIP | **None** |
| DNS 返回 | **1 个 VIP** | **3 个 Pod IP** |
| 谁负载均衡 | **kube-proxy**（iptables 随机 DNAT） | **客户端自己** |
| 客户端知道连的是谁 | 不知道 | 知道（就是具体某个 Pod） |

**类比**：ClusterIP = 打总机，接线员随手转接；Headless = 拿到分机名册，你自己挑。

### ⚠️ 关键认知：Headless 返回的多个 IP 是**无序集合**，不能在集合里"挑一个"

实测连续 3 次 dig 同一个域名：

```
第 1 次:  10.244.85.218   10.244.85.220   10.244.58.236
第 2 次:  10.244.85.218   10.244.58.236   10.244.85.220
第 3 次:  10.244.58.236   10.244.85.220   10.244.85.218
                ↑ 顺序每次都不一样
```

元凶是 CoreDNS 的 **`loadbalance` 插件**（Corefile 里有一行 `loadbalance`）。它故意打乱 A 记录顺序，防止所有客户端都取第一个 IP 把负载堆到同一台 Pod 上。

**语义对照**：

```
web-headless            → { IP1, IP2, IP3 }   "这 3 个等价，随便挑"
web-0.nginx-svc-headless → IP0                "这就是 web-0，唯一确定"
```

**要精确寻址"某一个"，办法不是在集合里挑，而是换一个唯一的名字**（逐 Pod 域名）。`loadbalance` 只在「一个名字对多个 IP」时起作用，逐 Pod 域名不受影响。

**真实系统怎么知道该找谁？三种做法**：

| 做法 | 适用 | 说明 |
|---|---|---|
| A. 不需要知道 | 无状态、Pod 完全等价 | 轮询/随机即可 —— **这种情况其实应该用 ClusterIP，别用 Headless** |
| B. 逐 Pod 域名精确寻址 | 有状态、角色固定 | 约定 `web-0` 是主库 → 直连 `web-0.<svc>.<ns>.svc.cluster.local`。**角色信息来自名字的约定，不来自 IP** |
| C. 应用层协商 / 专用 Service | 角色会变（主库切换） | ① 连上所有节点问"谁是主"（Redis Sentinel / MySQL MGR / Kafka）；② 给主库打标签，建专用 selector 的 Service |

**这条链解释了 StatefulSet 存在的理由**：

```
需要「精确寻址某一个后端」
  ↓ 必须有「稳定的唯一名字」
     ↓ 必须用 StatefulSet（名字带序号，重建不变）
        ↓ 必须配 Headless Service（DNS 返回 Pod IP 而非 VIP）
           ↓ 返回多个时必须用「逐 Pod 域名」才能定位

反证：Deployment 的 Pod 名是随机 hash（web-svc-5b86d588d7-t4w8p），
      重建就换名 → 压根给不了稳定域名 → 永远做不到精确寻址
```

### 逐 Pod 稳定域名（最实用的能力）

`<pod-name>.<headless-svc>.<ns>.svc.cluster.local`

实测（StatefulSet `web`，serviceName=nginx-svc-headless）：

```
web-0.nginx-svc-headless.default.svc.cluster.local → 10.244.58.219
web-1.nginx-svc-headless.default.svc.cluster.local → 10.244.58.221
web-2.nginx-svc-headless.default.svc.cluster.local → 10.244.58.222
```

前提校验（两个字段都必须对）：

```bash
kubectl get pod web-0 -o jsonpath='{.spec.hostname}'    # web-0
kubectl get pod web-0 -o jsonpath='{.spec.subdomain}'   # nginx-svc-headless
```

**稳定身份 = 身份绑在名字上，不绑 IP**。删掉 web-0 重建，名字仍是 web-0、IP 变了、域名照样指向它。Deployment 的 Pod 名是随机 hash，做不到。

### 应用场景

| 场景 | 用 Headless 的理由 |
|---|---|
| StatefulSet（DB/MQ/ZK/etcd） | 每台角色不同，必须精确寻址（从库必须连主库，不能随机） |
| 需要逐 Pod 稳定域名 | `pod-0.svc.ns.svc.cluster.local` 长期有效 |
| 客户端自建负载均衡 | 拿全量 IP，自己轮询/熔断（gRPC 长连接场景） |
| 感知副本数变化 | DNS 返回的 IP 数量就是真实副本数 |
| **无状态 Web 服务** | ❌ **不要用**，属自找麻烦 |

## 六、ExternalName：唯一不碰 Pod 的类型

```bash
kubectl get svc web-ext -o wide
# NAME      TYPE           CLUSTER-IP   EXTERNAL-IP     PORT(S)   SELECTOR
# web-ext   ExternalName   <none>       www.baidu.com   <none>    <none>

kubectl get endpoints web-ext
# Error from server (NotFound)     ← 对象根本不存在，不是"空"

kubectl describe svc web-ext | grep -E 'Selector|Type'
# Selector:          <none>
# Type:              ExternalName
```

DNS 验证（关键证据）：

```
web-ext.default.svc.cluster.local  canonical name = www.baidu.com
```

**为什么 ExternalName 不需要 Endpoints / selector / ports？**

```
Endpoints 的唯一作用 = 提供「Service IP → Pod IP」转换表，供 kube-proxy 写 DNAT
↓
ExternalName 完全不经过 kube-proxy —— 它是纯 DNS 层的 CNAME 重定向
↓
不写 DNAT  → 不需要 Endpoints
不指向 Pod → 不需要 selector
不做端口映射（直连外部服务真实端口）→ 不需要 ports
```

**和"selector 写错"的本质区别**：selector 写错是 Endpoints **存在但为空**（`<none>`）；ExternalName 是 Endpoints **对象压根不存在**（NotFound）。

## 七、踩坑实录

### 1. `PORT(S)` 列的术语陷阱

`80:31629/TCP` 是 **`port:nodePort`**，**不是**"源端口:目标端口"。三个字段在 `describe` 里才分开显示：

```
Port:        80/TCP        ← port（Service 逻辑端口）
TargetPort:  80/TCP        ← targetPort（Pod 容器端口）
NodePort:    31629/TCP     ← nodePort（节点端口）
```

### 2. targetPort 写错 → 是"秒拒"，不是"超时"

实测打一个无监听的端口：

```
curl http://10.244.85.218:8080
→ curl: (7) Failed to connect ... after 3 ms: Could not connect to server
→ 17ms 就返回
```

因为 Pod 内核**主动回 RST**（IP 活着，只是没进程监听）。

**排障分水岭（CKA 排障核心）**：

| 现象 | 含义 | 排查方向 |
|---|---|---|
| `Connection refused`（秒回） | 网络通，**端口不对** | `describe svc` 看 TargetPort、Pod 里 `ss -tlnp` |
| `Connection timed out`（几十秒） | **网络层不通** | `get endpoints`、NetworkPolicy、CNI |

### 3. NodePort 三节点都能通（即使节点上没 Pod）

实测 master / node01 / node02 三个 IP 全部 200，但 master 上一个 nginx Pod 都没有。原因：**kube-proxy 在每个节点都写规则**，所以任意节点 IP:NodePort 都能被 DNAT 到后端 Pod。

### 4. `port` 可与 `targetPort` 完全解耦 + 一个高频考点

`port` 是纯粹的逻辑端口，**不绑定任何真实进程**：

```yaml
ports:
- port: 9999        # 对外就叫 9999
  targetPort: 80    # 流量落到 Pod 的 80
```

约束三条：① 多端口 Service 每个 port **必须起 name**（硬校验）；② `port` 是对外契约，改了所有调用方要改；③ 真正不自由的是 `targetPort`（须匹配容器监听）和 `nodePort`（须 30000–32767 且集群内唯一）。

🔥 **考点：`targetPort` 省略时默认 = `port` 的值。**

```yaml
ports:
- port: 80
  # 不写 targetPort → 等价于 targetPort: 80
```

**经典坑**：把 `port` 改成 8080 又忘写 `targetPort` → 自动变成 `targetPort: 8080` → 容器实际听 80 → `Connection refused`。

### 5. 短名解析会掉到上游，拿到不可路由的 fake-ip

```
web-0.nginx-svc-headless.default.svc.cluster.local → 10.244.58.219  ✅
web-0.nginx-svc-headless                           → 28.0.0.206      ❌ 假 IP
web-0.nginx-svc-headless.default                   → 28.0.0.209      ❌ 假 IP
```

根因链：

```
1. 写了个 cluster.local 里不存在的名字
2. CoreDNS 有 `forward . /etc/resolv.conf` → 不认识的名字转发上游
3. 宿主机开了代理工具（Clash 类）的 fake-ip 模式
   → 对任何域名都返回 fake-ip，而不是 NXDOMAIN
   （28.0.0.0/8 + IPv6 fdfe:dcba:9876::/64 是 Clash 默认 fake-ip 池）
4. 于是拿到 28.0.0.x 这种不可路由的地址
```

**两条结论**：

1. **跨 namespace 或写域名，一律用完整 FQDN**
2. **DNS 解析"成功" ≠ 网络可达** —— 拿到 `28.0.0.x` 这种不像集群网段的地址，先怀疑 fake-ip/劫持
3. 副产：**ExternalName 指向外部域名在此环境 DNS 层可通、网络层不可达**（fake-ip 只有宿主机代理进程能路由）

Pod 内 DNS 配置：

```
search default.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.96.0.10
options ndots:5
```

`ndots:5` = 域名点数少于 5 个时，**先按 search 列表逐个展开尝试**。同 namespace 短名能用就是这个机制。

## 八、排障三板斧

```bash
# 1. Service 长什么样（重点看 Selector 与 Endpoints 列）
kubectl describe svc <name>

# 2. 一眼定生死 —— 空的/NotFound 就是后端没匹配上
kubectl get endpoints

# 3. 从 Pod 内部真实访问一次（别在外面 curl，要模拟真实调用方）
kubectl run tmp --image=busybox:1.36 --rm -it --restart=Never -- \
  sh -c "wget -qO- --timeout=3 http://web-cip; echo"
```

**Service 的故障全是哑巴故障**：selector 写错不会报错、不会警告，`kubectl get svc` 里 Endpoints 列安静地显示 `<none>`。所以排障第一个动作永远是 `kubectl get endpoints`。

现成案例：集群里 `svc-ready` 的 selector=`app=web`，而 `kubectl get pods -l app=web` 返回 `No resources found` → Endpoints 空。

## 八点五、DNS 验证方法（重要技巧）

**必须指定 CoreDNS 的 ClusterIP 查询**，否则节点会走 `/etc/resolv.conf` → systemd-resolved → 上游 → 拿到 fake-ip 假地址。

```bash
# CoreDNS 的 Service 名叫 kube-dns（历史遗留，虽然跑的是 CoreDNS）
kubectl get svc -n kube-system kube-dns        # ClusterIP 通常 10.96.0.10

# 方法1（最快）：从节点直接指定 CoreDNS 查，master 自带 dig/nslookup/host
dig @10.96.0.10 web-headless.default.svc.cluster.local +short

# 方法2：Pod 内非交互（自动化/留档，Tabby 自动通道里 -it 会挂）
kubectl run dnstest --image=busybox:1.36 --restart=Never --command -- sh -c 'nslookup web-headless'
sleep 6; kubectl logs dnstest; kubectl delete pod dnstest

# 方法3：Pod 内交互（手工实验，可顺便看 resolv.conf / 试短名）
kubectl run tmp --image=busybox:1.36 --rm -it --restart=Never -- sh
```

**`+short` 的取舍**：

- 加 `+short` → 只出 IP，适合并排对比（1 个 VIP vs 全部 Pod IP，差异一眼可见）
- 不加 `+short` → 出完整应答，能看到 **CNAME**（ExternalName 场景必须用这个）

**注意**：`kubectl get endpoints` 虽然也能看到全部 Pod IP，但 **ClusterIP 的 Endpoints 完全一样** —— 它**证明不了 Headless 的特殊性**，差异只在 DNS 应答那一层。

## 九、复习自测题（答案）

1. **EndpointSlice 和 Endpoints 的本质差异？**
   - 不只"省略重复子对象"。核心：分片可扩展性（默认 100/片）、三态条件（ready/serving/terminating，解决滚动更新 502）、拓扑 hints、addressType、API 组不同
   - 端口位置：slice 是**切片级 `ports`**（全体共用），Endpoints 是**每条 endpoint 带 IP:port**

2. **`PORT(S)` 列的 `80:31629` 各是什么？**
   - `port` : `nodePort`。`targetPort` 不在这一列，只在 `describe` 里

3. **targetPort 写错会超时吗？**
   - 不会。Pod 内核主动回 RST → 秒级 `Connection refused`（curl exit 7）
   - 超时意味着网络层不通（无 endpoints / NetworkPolicy / CNI 故障）

4. **NodePort 为什么任意节点 IP 都能访问？**
   - kube-proxy 在**每个节点**都写 DNAT 规则，不是节点上起了代理进程

5. **Headless 与 ClusterIP 的差别在哪一层？**
   - **DNS 层**。两者 Endpoints 完全相同；DNS 应答时 ClusterIP 返回 1 个 VIP，Headless 返回全部 Pod IP，负载均衡责任转移给客户端

6. **Headless 的逐 Pod 域名格式？前提条件？**
   - `<pod>.<headless-svc>.<ns>.svc.cluster.local`
   - 需 Pod 的 `hostname` + `subdomain` 正确，且 StatefulSet 的 `serviceName` 指向该 Headless Service

7. **ExternalName 为什么没有 Endpoints？**
   - 不走 kube-proxy（纯 DNS CNAME），无需 DNAT 规则；不指向 Pod，无需 selector；不做端口映射，无需 ports
   - `kubectl get endpoints` 返回 **NotFound**（对象不存在），区别于 selector 写错导致的"存在但为空"

8. **`port` 可以随便设吗？**
   - 基本可以，它是纯逻辑端口、与容器端口解耦
   - 约束：多端口必须每个起 name；port 是对外契约；targetPort 须匹配容器监听；nodePort 须 30000–32767 且唯一
   - **考点：targetPort 省略时默认 = port 的值**

## 速记卡片

```
# 四类 Service 创建
kubectl expose deploy X --name=A --port=80 --target-port=80                      # ClusterIP
kubectl expose deploy X --name=B --port=80 --target-port=80 --type=NodePort
kubectl expose deploy X --name=C --port=80 --target-port=80 --cluster-ip=None    # Headless
kubectl create service clusterip D --tcp=9999:80
# ExternalName 必须手写 YAML

# 三层映射
nodePort(30000-32767) → port(逻辑端口) → targetPort(容器端口)
targetPort 省略 = port 的值   ← 高频考点

# 排障三板斧
kubectl describe svc X
kubectl get endpoints          # 空/NotFound = 后端没匹配上
kubectl run tmp --image=busybox:1.36 --rm -it --restart=Never -- nslookup X

# DNS 命名
<svc>.<ns>.svc.cluster.local              # Service
<pod>.<svc>.<ns>.svc.cluster.local        # Headless + StatefulSet 逐 Pod
<pod-ip带-.>.<ns>.pod.cluster.local        # Pod 默认

# 分水岭
Connection refused（秒回）→ 网络通，端口不对
Connection timed out      → 网络层不通
```
