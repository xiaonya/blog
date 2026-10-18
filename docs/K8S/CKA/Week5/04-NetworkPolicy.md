---
title: "NetworkPolicy 网络策略"
sidebar_position: 4
---

# NetworkPolicy 网络策略

> 集群：k8s-master / k8s-node01 / k8s-node02（v1.36.3）
> CNI：**Calico v3.29.1**（必须支持 NetworkPolicy 才能练，Flannel 原生不支持）

## 核心认知：Pod 级的防火墙，管的是东西向流量

```
Service / Ingress   →  管【南北向】：外部 ↔ 集群
NetworkPolicy       →  管【东西向】：Pod ↔ Pod（也含 Pod ↔ 外部）
```

它作用在 **L3/L4**（IP + 端口 + 协议），**不认识域名**。

## 一、三条前提（缺一个都白搭）

| 前提 | 说明 |
|---|---|
| **CNI 必须支持** | Calico ✅ / Cilium ✅ / Flannel ❌。CNI 不支持时，NetworkPolicy 对象能被创建，但**完全不生效**（最坑的哑巴故障） |
| **命名空间级资源** | 写在哪个 ns 里，`podSelector` 就**只在该 ns 内**选 Pod（`namespaceSelector` 才是跨 ns 的入口） |
| **默认全通** | 没有任何策略时，Pod 之间随便通 |

### 官方对"被选中"的定义

> Pod 通过被 NetworkPolicy 选中而**实现隔离**（isolated）。一旦命名空间中存在任何选中特定 Pod 的 NetworkPolicy，该 Pod 将**拒绝任何未被任何 NetworkPolicy 允许的连接**。（命名空间中未被任何 NetworkPolicy 选中的其他 Pod 将**继续接受所有流量**。）

两个关键信息：
1. **"被选中" = 进入隔离态**，隔离态里是 **default-deny + 白名单**
2. **没被选中的 Pod 照旧全通** —— 同一个 ns 里有的 Pod 在隔离态、有的在全通态，是正常的

## 二、三条铁律

### 铁律 1：按方向隔离

```
Pod 没有任何 NetworkPolicy 选中它      →  所有方向全通
Pod 被只含 Ingress 的策略选中          →  入站变白名单，【出站完全不受影响】
```

只在 `policyTypes` 里写了哪个方向，那个方向才被隔离。

### 铁律 2：`podSelector: {}` 是「选中全部」

空选择器 = 匹配**本命名空间所有 Pod**，不是"不选"。语义和直觉相反，考试爱考。

```yaml
podSelector: {}          # 本 ns 全部 Pod
podSelector: {matchLabels: {}}   # 同上
```

### 铁律 3：同一 `-` 项内 = AND，另起 `-` = OR

```yaml
ingress:
  - from:
      - namespaceSelector: {...}   # ┐
        podSelector:       {...}   # ┘ 同一个列表项 → AND（且）
      - podSelector:       {...}   #   新的列表项   → OR （或）
```

**看 YAML 时数连字符的层级** —— 缩进对齐在同一 `-` 下面的多个选择器是 AND，另起一个 `-` 是 OR。

## 三、字段全表

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: app-b-policy
  namespace: default              # ★ 作用域
spec:
  podSelector:                    # ★ 选中哪些 Pod（本 ns 内）
    matchLabels:
      app: app-b
  policyTypes:                    # ★ 隔离哪个方向
    - Ingress
    - Egress
  ingress:                        # 入站白名单
    - from:
        - podSelector: {...}          # 本 ns 的 Pod
        - namespaceSelector: {...}    # 某个 ns 的 Pod
        - ipBlock: {...}              # 外部 CIDR
      ports:
        - protocol: TCP
          port: 80
  egress:                         # 出站白名单（结构同上，用 to 而非 from）
    - to:
        - podSelector: {...}
      ports:
        - protocol: UDP
          port: 53
```

### policyTypes 的默认规则

| 你写的 | 实际生效 |
|---|---|
| 有 `ingress` 段，没写 policyTypes | 自动加上 `Ingress` |
| 有 `egress` 段，没写 policyTypes | 自动加上 `Egress` |
| 写了 `policyTypes: [Egress]` 但 `egress` 段为空 | **拒绝所有出站** ← 坑 |
| 写了 `policyTypes: [Ingress, Egress]` 但只有一个方向有规则 | 另一个方向全拒 |

### 命名空间的标准选法

K8s 1.21+ 会自动给每个 ns 打上 `kubernetes.io/metadata.name` 标签，值就是 ns 名：

```bash
kubectl get ns kube-system --show-labels
```

所以按 ns 选目标的标准写法是：

```yaml
namespaceSelector:
  matchLabels:
    kubernetes.io/metadata.name: kube-system
```

## 四、实验 1：基线（默认全通）

```bash
kubectl exec deploy/app-a -- wget -qO- --timeout=3 http://app-b
# → <<< I am APP-B >>>
```

**没有策略时，Pod 之间随便通。** 这就是基线。

## 五、实验 2：deny-all（实测 timeout）

```yaml
spec:
  podSelector:
    matchLabels:
      app: app-b
  policyTypes:
    - Ingress
  # 故意不写 ingress 规则
```

```bash
kubectl exec deploy/app-a -- wget -qO- --timeout=3 http://app-b
# → wget: download timed out      ★ 超时，不是 refused
```

### ★ 关键认知：NetworkPolicy 的拒绝是 DROP，不是 REJECT

**被拒流量是静默丢包**，客户端只能等自己的超时。不会收到 `RST` 或 `ICMP port unreachable`。

这一点和 Day2 的排障三分表接上了：

| 现象 | 含义 |
|---|---|
| **超时** | 网络层不通 → **CNI / NetworkPolicy / 路由表** |

现在你知道这一格里的 NetworkPolicy 具体长什么样了。

## 六、实验 3：精确放行

```yaml
spec:
  podSelector: {matchLabels: {app: app-b}}
  policyTypes: [Ingress]
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: app-a
```

```bash
kubectl exec deploy/app-a  -- wget -qO- --timeout=3 http://app-b   # ✅ 通
kubectl exec deploy/web-svc -- wget -qO- --timeout=3 http://app-b   # ❌ 不通
```

**同一个目标，来源不同结果不同** —— 这就是 NetworkPolicy 的全部意义。

## 七、★ Egress 的头号坑：DNS 忘了放行

### 出问题的配置

```yaml
policyTypes: [Ingress, Egress]
ingress: [...]      # 有规则
# egress 段：空的！
```

### 实测现象

```
app-b -> app-a     →  wget: bad address 'app-a'
app-b -> web-cip   →  wget: bad address 'web-cip'
app-b 的 nslookup  →  ;; connection timed out; no servers could be reached
```

**注意报错是 `bad address`，不是 `connection timed out`。**

### 机制

```
app-b 想访问 app-a
  → 先要解析 "app-a" 这个域名
  → 发 DNS 查询到 CoreDNS (10.96.0.10:53)
  → 这个查询本身也是"出站流量"，被 Egress 策略 DROP ✗
  → 域名解析失败 → wget 报 "bad address"
  → 业务流量根本没机会发出去
```

**`policyTypes` 含 `Egress` 而 `egress` 段为空 = 拒绝所有出站，包括 DNS。Pod 直接变成瞎子。**

生产上很多人配完 Egress 策略发现"服务全挂了"，查半天网络，最后发现是 DNS 被自己的策略掐了。

### 修复

```yaml
  egress:
    # ① 放行 DNS —— 少了这条整台 Pod 变瞎子
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
    # ② 放行到 app-a 的 80
    - to:
        - podSelector:
            matchLabels:
              app: app-a
      ports:
        - protocol: TCP
          port: 80
```

修复后实测：

```
app-b -> app-a（域名）      →  <<< I am APP-A >>>          ✅
app-b -> app-a ClusterIP    →  <<< I am APP-A >>>          ✅
app-b -> app-a PodIP        →  <<< I am APP-A >>>          ✅
app-b -> web-cip            →  wget: download timed out    ⏱
nslookup app-a              →  Address: 10.104.13.56       ✅ 恢复
```

**注意 `web-cip` 的报错从 `bad address` 变成了 `download timed out`** —— 说明两阶段：

```
阶段 ① DNS：53/UDP 已放行 → 解析成功，拿到 ClusterIP
阶段 ② 连接：DNAT 后目标是 web-cip 的 Pod，不在放行列表 → DROP → 超时
```

**同样是"不通"，一个卡在解析、一个卡在连接，排查方向完全不同。**

## 八、★ AND vs OR（实测对照表）

### 两种写法

**写法 A —— 同一个 `from` 项里两个选择器（AND）**

```yaml
ingress:
  - from:
      - namespaceSelector:
          matchLabels:
            kubernetes.io/metadata.name: net-test
        podSelector:
          matchLabels:
            app: app-a
```

**写法 B —— 拆成两个 `from` 项（OR）**

```yaml
ingress:
  - from:
      - namespaceSelector:
          matchLabels:
            kubernetes.io/metadata.name: net-test
      - podSelector:
          matchLabels:
            app: app-a
```

### 测试环境

| Pod | 标签 | 命名空间 |
|---|---|---|
| `app-a` | `app=app-a` | default |
| `probe` | `app=app-a` | **net-test** |
| `probe-out` | `app=outsider` | default |
| `web-svc` | `app=web-svc` | default |

`app-b` ClusterIP = `10.103.36.92`

### ★ 实测结果

| 来源 | 标签 | 命名空间 | **写法 A**（AND） | **写法 B**（OR） |
|---|---|---|---|---|
| `app-a` | `app=app-a` | **default** | ❌ **timeout** | ✅ 通 |
| `probe` | `app=app-a` | **net-test** | ✅ 通 | ✅ 通 |
| `probe-out` | `app=outsider` | default | ❌ timeout | ❌ timeout |
| `web-svc` | `app=web-svc` | default | ❌ 不通 | ❌ 不通 |

**唯一的差异点在第一行：`default` 命名空间里的 `app-a`。**

### 两个反直觉点

**① 写法 A 反而更严**

`default/app-a` 的标签明明是 `app=app-a`，但在写法 A 下**不通**了：

```
写法 A 的语义：
   在 net-test 命名空间里  AND  标签是 app-a
   → default 里的 app-a 虽然标签对，但【命名空间不对】→ 出局
```

**加 `namespaceSelector` 不是"放宽范围"，而是"额外加了一道约束"。** 很多人以为多写一个选择器是"或者"，结果把原本能通的流量一刀切断 —— 生产上这类事故不少。

**② `podSelector` 不跨命名空间**

`web-svc`（`app=web-svc`，default ns）在写法 B 下也**不通**。因为写法 B 里的 `podSelector: {app: app-a}` 只在**策略所在的 default ns** 内匹配，找不到 `web-svc`。

### 自然语言读法

| 写法 | 读作 |
|---|---|
| 写法 A | net-test 里的 **并且** 是 app-a 的 Pod |
| 写法 B | （net-test 里的所有 Pod）**或者**（**本 ns** 里的 app-a） |

## 九、工作层次：DNAT 与策略检查的顺序

### 实测发现

策略里写的是 `podSelector: {app: app-a}`（按**标签**匹配），但 app-b 用 **ClusterIP** 甚至 **Pod IP** 访问都能通：

```
app-b -> app-a ClusterIP (10.104.13.56)  →  ✅ 通
app-b -> app-a PodIP     (10.244.58.249) →  ✅ 通
```

**为什么用 ClusterIP 也能被"Pod IP 集合"匹配上？**

### 机制

```
app-b Pod 发出包（dst = ClusterIP 10.104.13.56:80）
      ↓ 进入宿主机网络栈
   ① nat/PREROUTING —— kube-proxy 的 DNAT 规则
      dst 被改写成 app-a 的真实 Pod IP：10.244.58.249:80
      ↓
   ② filter/FORWARD —— Calico 的 NetworkPolicy 策略检查
      此时目标已经是 Pod IP ← 在 app-a 的 IP 集合里 ✅ 放行
```

**DNAT 在前，策略检查在后。**

### 三个推论

1. **"用 IP 绕过 NetworkPolicy"不可能** —— 只要目标还是那个 Pod，策略必然生效
2. **写策略时不用关心调用方用什么方式访问** —— 只写目标 Pod 的标签就够了
3. **NetworkPolicy 工作在 L3/L4**，它不认识域名

> ⚠️ 这个顺序是 "Calico + iptables + kube-proxy" 组合下的实测结果。其他 CNI（如 eBPF 模式）实现可能不同。

## 十、ipBlock

### 官方定义（原文）

> `ipBlock`：此选择器将选择特定的 IP CIDR 范围以用作入站流量来源或出站流量目的地。**这些应该是集群外部 IP**，因为 Pod IP 存在时间短暂且随机产生。集群的入站和出站机制通常需要重写数据包的源 IP 或目标 IP。**在发生这种情况时，不确定在 NetworkPolicy 处理之前还是之后发生**……对入站流量而言，这意味着在某些情况下，你可以根据实际的原始源 IP 过滤传入的数据包，**而在其他情况下 NetworkPolicy 所作用的源 IP 则可能是 LoadBalancer 或 Pod 的节点**。

### 和 podSelector 的命运对比

| 选择器 | 匹配依据 | 受 IP 改写影响吗 |
|---|---|---|
| `podSelector` / `namespaceSelector` | **逻辑身份**（Calico 翻译成 Pod IP 集合，在 DNAT **之后**检查） | **不影响** |
| `ipBlock` | **IP 字面量** | **影响** —— 源 IP 可能被 SNAT 成节点 IP |

### 为什么源 IP 会变

```
外部客户端真实 IP 1.2.3.4
   → 到达节点
   → kube-proxy 做 SNAT（默认 externalTrafficPolicy: Cluster）
   → 源 IP 变成【节点 IP】
   → 包到达 Pod
   → NetworkPolicy 检查源 IP，看到的可能不是 1.2.3.4
```

**这正是 `externalTrafficPolicy: Local` 存在的理由之一** —— 保留客户端真实源 IP（代价是本节点没有后端 Pod 时流量直接丢）。

### 正确用法

| 场景 | 写法 |
|---|---|
| 只允许公司内网网段访问 | `ipBlock: {cidr: 10.0.0.0/8}` |
| 放行某个外部数据库出口 | `to: [{ipBlock: {cidr: 203.0.113.5/32}}]` |
| 排除特定 IP | `ipBlock: {cidr: 0.0.0.0/0, except: [203.0.113.5/32]}` |

**集群内部的 Pod 间通信，一律用 `podSelector` / `namespaceSelector`，不要用 ipBlock。**

### hostNetwork 的例外

**`hostNetwork: true` 的 Pod 不受 NetworkPolicy 约束** —— 它没有独立的 veth 网卡，NetworkPolicy 挂不上去。要限制与这类 Pod 的通信，只能从别的 Pod 侧用 `ipBlock` 写节点 IP。

## 十一、失败现象三分表

| 报错 | 含义 | 排查方向 |
|---|---|---|
| **`bad address`** / `no servers could be reached` | **DNS 被拒** | Egress 策略有没有放行 53/UDP |
| **`connection timed out`** | 网络层 DROP | **NetworkPolicy / 路由表 / CNI** |
| **`connection refused`** | 目标端口没监听 | 后端应用 / targetPort |

**这张表建议背熟** —— 三种"不通"的排查方向完全不同，认错方向会浪费大量时间。

## 十二、踩坑实录

### 1. ★ Egress 策略忘了放行 DNS → Pod 变瞎子

`policyTypes` 含 `Egress` 但 `egress` 段为空 → 所有出站被拒，**包括 DNS 查询**。表现是 `bad address` 而不是 `connection timed out`。

**加 Egress 策略时，第一条永远是放行 DNS。**

### 2. ★ 加 `namespaceSelector` 是收紧不是放宽

`namespaceSelector` + `podSelector` 在同一项里是 AND。原本只写 `podSelector` 能通的对象，加了 `namespaceSelector` 后可能**反而被切断**（因为命名空间不匹配）。

### 3. ★ `podSelector` 不跨命名空间

它只在**策略所在的 ns** 内匹配。跨 ns 必须用 `namespaceSelector`。

### 4. `podSelector: {}` 是选中全部，不是不选

空对象 = 本 ns 所有 Pod。

### 5. NetworkPolicy 拒绝 = DROP，表现是 timeout

不是 `refused`。想看到拒绝包是没有的，只能等超时。

### 6. 写了策略但完全不生效 → 先查 CNI

Flannel 等 CNI 不支持 NetworkPolicy。策略对象能建，但没有任何效果 —— **典型的哑巴故障**。

```bash
kubectl get pods -n kube-system | grep -E 'calico|cilium'    # 确认 CNI
```

### 7. `kubectl exec deploy/xxx` 在多副本时随机挑 Pod

测试结果不稳定时，指定具体 Pod 名。

### 8. `nginx:1.25`（Debian 基础）里没有 `wget`

只有 `nginx:alpine` 才有 busybox 的 wget。测试前先确认工具存在，否则会误判成"网络不通"。

### 9. `ipBlock` 匹配不到你以为的源 IP

SNAT 会把外部客户端源 IP 改成节点 IP。

## 十三、复习自测题（答案）

1. **NetworkPolicy 在什么情况下生效？**
   - 必须 CNI 支持（Calico / Cilium）。CNI 不支持时策略对象能建但**完全不生效**

2. **一个 Pod 完全没有 NetworkPolicy 选中它，它的流量怎样？**
   - **所有方向全通**。被选中后才进入"隔离态"= default-deny + 白名单

3. **NetworkPolicy 拒绝流量时，客户端看到什么？**
   - **超时**（DROP 静默丢包），不是 `connection refused`（那是 REJECT）

4. **为什么加了 Egress 策略后 Pod 解析不了域名？**
   - 因为 DNS 查询本身是出站流量，被 Egress 策略拒了
   - 表现是 `bad address` / `no servers could be reached`
   - 修复：放行到 kube-system 的 53/UDP + 53/TCP

5. **`podSelector: {}` 是什么意思？**
   - 选中**本命名空间所有 Pod**

6. **下面两种写法有什么区别？**
   ```yaml
   # A
   from:
     - namespaceSelector: {matchLabels: {kubernetes.io/metadata.name: ns1}}
       podSelector: {matchLabels: {app: x}}
   # B
   from:
     - namespaceSelector: {matchLabels: {kubernetes.io/metadata.name: ns1}}
     - podSelector: {matchLabels: {app: x}}
   ```
   - **A（AND）**：ns1 **里**标签为 x 的 Pod
   - **B（OR）**：ns1 **里的所有 Pod**，**或者**本 ns 里标签为 x 的 Pod
   - 关键：同一 `-` 项内是 AND，另起 `-` 是 OR

7. **`podSelector` 能匹配到其他命名空间的 Pod 吗？**
   - **不能**。它只在策略所在的 ns 内匹配。跨 ns 要用 `namespaceSelector`

8. **为什么用 ClusterIP 或 Pod IP 访问都绕不过 NetworkPolicy？**
   - 因为 **DNAT 发生在策略检查之前**
   - 包里的目标地址在策略检查时已经被统一成 Pod IP，所以按 Pod 标签写的规则必然命中

9. **`policyTypes: [Ingress, Egress]` 但 `egress` 段为空，会怎样？**
   - **拒绝所有出站流量**（包括 DNS）

10. **`ipBlock` 为什么官方说"应该是集群外部 IP"？**
    - 因为集群内的 IP 会被 SNAT/DNAT 改写，而官方明确说**不保证改写发生在策略处理之前还是之后**
    - 集群内部的通信应该用 `podSelector` / `namespaceSelector`

11. **`hostNetwork: true` 的 Pod 受 NetworkPolicy 约束吗？**
    - **不受**。它没有独立的 veth 网卡

12. **怎么判断"不通"是 DNS 问题还是网络问题？**
    - `bad address` / `no servers could be reached` → DNS 被拒
    - `connection timed out` → 网络层 DROP
    - `connection refused` → 端口没监听

## 速记卡片

```bash
# ── 前提检查
kubectl get pods -n kube-system | grep -E 'calico|cilium'   # CNI 必须支持
kubectl get networkpolicy -A                                 # 现有策略
kubectl describe networkpolicy <name>                        # 解读规则

# ── 排障（三种"不通"）
bad address / no servers could be reached  → DNS 被 Egress 拒了
connection timed out                       → 网络层 DROP（NetworkPolicy/路由/CNI）
connection refused                         → 端口没监听

# ── 测试来源（用 IP 避免 DNS 干扰）
kubectl exec deploy/app-a -- wget -qO- --timeout=3 http://<目标IP>

# ── 命名空间标签（K8s 1.21+ 自动打）
kubectl get ns kube-system --show-labels
# → kubernetes.io/metadata.name=kube-system

# ── 三条铁律
① 被选中 = 该方向 default-deny + 白名单（没被选中的 Pod 照旧全通）
② podSelector: {} = 选中全部（不是不选）
③ 同一 "-" 项内多选择器 = AND ；另起 "-" = OR

# ── YAML 读法
from:
  - namespaceSelector: {...}   ┐
    podSelector:       {...}   ┘ 缩进对齐 → AND
  - podSelector:       {...}     另起一项 → OR

# ── Egress 必备（否则 Pod 变瞎子）
egress:
  - to:
      - namespaceSelector:
          matchLabels:
            kubernetes.io/metadata.name: kube-system
    ports:
      - {protocol: UDP, port: 53}
      - {protocol: TCP, port: 53}

# ── 关键认知
拒绝是 DROP → 表现 timeout，不是 refused
DNAT 在策略检查之前 → 用 IP 绕不过 podSelector
podSelector 不跨 ns → 跨 ns 必须 namespaceSelector
加 namespaceSelector 是【收紧】不是放宽
ipBlock 只该用于集群外部 IP（受 SNAT/DNAT 影响）
hostNetwork Pod 不受 NetworkPolicy 约束
```
