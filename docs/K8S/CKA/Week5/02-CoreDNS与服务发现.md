---
title: "CoreDNS 与服务发现"
sidebar_position: 2
---

# CoreDNS 与服务发现

> 集群：k8s-master / k8s-node01 / k8s-node02（v1.36.3，Calico v3.29.1）

## 核心认知：CoreDNS 是集群的"电话簿"

```
客户端代码里写的是：http://web-cip:80
                    ↓ 谁把它变成 IP？
              CoreDNS（10.96.0.10）
                    ↓
              Service / Pod 的真实 IP
```

Day1 亲眼看到：同一批 Pod 两天内 IP 全换了（`10.244.58.236` → `10.244.58.241` …）。**Pod IP 不可依赖，所以必须有"名字 → IP"的动态翻译机制。**

**CoreDNS 不是 Kubernetes 的组成部分，而是跑在集群里的一个普通 Deployment。**

## 一、部署形态（注意命名陷阱）

```bash
kubectl -n kube-system get deploy coredns          # 2/2
kubectl -n kube-system get svc kube-dns -o wide    # 10.96.0.10
kubectl -n kube-system get cm coredns -o yaml      # Corefile 在这
kubectl get pods -n kube-system -l k8s-app=kube-dns
```

| 对象 | 名字 |
|---|---|
| Deployment | `coredns` |
| **Service** | **`kube-dns`** ← 不是 coredns！ |
| ConfigMap | `coredns` |
| Pod label | `k8s-app=kube-dns` |

**⚠️ Service 叫 `kube-dns` 是历史遗留**（早期用 kube-dns 组件，换 CoreDNS 后名字没改）。考试找不到 CoreDNS 时，`get svc coredns` 是没有的。

**端口明细（名字比端口号更值得记）：**

```
dns      : UDP/53   → targetPort 53
dns-tcp  : TCP/53   → targetPort 53     ← 与 dns 同一容器端口，只是协议不同
metrics  : TCP/9153 → targetPort 9153
```

### 为什么 DNS 要同时开 UDP 和 TCP 53

**UDP 是常态路径，TCP 是"大到装不下时的兜底通道"。**

**DNS 响应有 512 字节的历史限制（RFC 1035）。** 超出时：

```
1. DNS 服务器把响应【截断】到 512 字节内
2. 报文头里把 TC（Truncated）标志位置 1        ← 关键信号
3. 客户端看到 TC=1 → 知道答案不完整
4. 客户端【改用 TCP】重发同一查询
5. TCP 无 512 限制，拿到完整答案
```

**实测换算：**

```
web-headless 有 3 个 Pod → 响应 MSG SIZE = 241 字节
每条 A 记录约 16 字节（名字压缩后）
→ (512 - 241) ÷ 16 ≈ 17
→ 约 20 个端点就会突破 512 字节
```

**只开 UDP 的后果**：Service 端点一多 → 客户端收到截断响应 → 尝试 TCP 重试 → 53/TCP 没监听 → 连接被拒 → **表现为"Pod 少的 Service 能解析、Pod 多的解析不了"，极难排查**。

| 53/TCP 的用途 | K8s 会遇到吗 |
|---|---|
| ① 响应截断后的 TCP 重试 | ✅ **就是它** |
| ② 区域传送（zone transfer） | ❌ 不涉及 |
| ③ DNSSEC 等大响应 | ❌ 一般不涉及 |

**取巧方法失效记录**：`dig +bufsize=200` **无法**人为触发截断——CoreDNS（符合 RFC 的实现）会把小于 512 的 EDNS0 buffer 值当作 **512** 处理。要看到 TC 标志必须让真实响应超过 512 字节。

## 二、Corefile 逐行拆解

```
.:53 {                                    ← 监听所有域名、53 端口
    errors                                ← 错误写日志
    health { lameduck 5s }                ← 健康端点；lameduck=关闭前延迟 5s 摘流量
    ready                                 ← readiness 探针端点

    kubernetes cluster.local in-addr.arpa ip6.arpa {   ← ★ 核心插件
       pods insecure                      ← ★ 决定 Pod A 记录是否生成
       fallthrough in-addr.arpa ip6.arpa  ← 反查失败时继续往下走
       ttl 30                             ← dig 看到的 TTL=30 出自这里
    }

    prometheus :9153                      ← Prometheus 指标

    forward . /etc/resolv.conf {          ← ★ 非 cluster.local 的名字全丢给上游
       max_concurrent 1000
    }

    cache 30 {
       disable success cluster.local      ← ★ cluster.local 成功结果【不缓存】
       disable denial cluster.local       ← ★ 失败结果【也不缓存】
    }

    loop                                  ← 防转发死循环
    reload                                ← Corefile 改了自动重载
    loadbalance                           ← ★ 打乱 A 记录顺序
}
```

**三个必记插件：**

| 插件 | 作用 | 为什么重要 |
|---|---|---|
| `kubernetes` | 应答 `cluster.local` | 服务发现本体；`pods insecure` 控制 Pod 域名 |
| `forward` | 不认识的名字转发上游 | **fake-ip 的入口**（Day1 悬案） |
| `loadbalance` | 打乱 A 记录顺序 | Day1"IP 顺序每次都变"的元凶 |

**两个设计取舍：**

- **为什么 `cache` 对 cluster.local 关缓存？** Service 端点在随时变。缓存了就会让客户端拿到已死掉的 IP。宁可牺牲性能也要保证服务发现实时性。
- **为什么 `pods insecure` 是 `insecure`？** 因为 Pod 的 A 记录**不校验 Pod 是否存在**就返回，属于"不安全但快"的模式。

## 三、DNS 命名规则全表

| 对象 | 域名格式 | 实例 |
|---|---|---|
| 普通 Service | `<svc>.<ns>.svc.cluster.local` | `web-cip.default.svc.cluster.local` |
| Headless + StatefulSet 逐 Pod | `<pod>.<svc>.<ns>.svc.cluster.local` | `web-0.nginx-svc-headless.default.svc.cluster.local` |
| Pod（按 IP） | `<IP 的点换横杠>.<ns>.pod.cluster.local` | `10-244-85-239.default.pod.cluster.local` |
| 反向（ClusterIP） | `<IP 倒序>.in-addr.arpa` → PTR | `-x 10.111.19.171` → `web-cip.default.svc.cluster.local` |
| 反向（Pod IP） | 同上 → PTR（**Service 视角**） | 见第五节 |
| SRV | `_<port-name>._<proto>.<svc>.<ns>.svc.cluster.local` | `_http._tcp.web-named.default...` |

## 四、★ 短名能不能用：取决于「解析器有没有 search 域」

**这张表是 Day2 最容易搞错的地方。**

| 写法 | 谁能用 | 原理 |
|---|---|---|
| `<svc>.<ns>.svc.cluster.local` | **所有人**（dig / Pod / 节点） | 完整 FQDN，不依赖 search |
| `<svc>.<ns>` | **只有走 libc 解析器的 Pod** | 靠 search 域 `svc.cluster.local` 拼接 |
| `<svc>` | **只有同 namespace 的 Pod** | 靠 search 域 `<ns>.svc.cluster.local` 拼接 |
| **节点上敲 `dig`** | **必须用完整 FQDN** | 节点 resolv.conf 是 `search .`，无 cluster.local search 域 |

**Pod 的 `/etc/resolv.conf`：**

```
search default.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.96.0.10
options ndots:5
```

`ndots:5` = **域名里点数少于 5 个时，先按 search 列表逐个展开尝试**。

**铁证**（在 Pod 内用 glibc 的 `getent` 验证）：

```
$ kubectl exec <pod> -- getent hosts nginx-tns.test-ns
10.108.178.246  nginx-tns.test-ns.svc.cluster.local
                ↑ 解析后的完整名字被打印出来 —— 直接看到 svc.cluster.local 被拼接
```

```
nginx-tns.test-ns  +  search 域 svc.cluster.local  =  nginx-tns.test-ns.svc.cluster.local
```

**⚠️ 两个"不走 search 域"的工具，验证短名时会误导你：**

| 工具 | 走 search 域吗 | 后果 |
|---|---|---|
| `dig`（在节点上） | ❌ | 短名一律失败 |
| busybox `nslookup` | ❌ | 短名失败，裸名直接查询 |
| `getent hosts`（libc） | ✅ | **验证短名的正确工具** |

## 五、SRV 记录：只为【命名端口】生成

```bash
# web-cip 的端口 name 是空的 → 没有 SRV
kubectl get svc web-cip -o jsonpath='{range .spec.ports[*]}name=[{.name}] port={.port}{"\n"}{end}'
# name=[] port=80

dig @10.96.0.10 _http._tcp.web-cip.default.svc.cluster.local SRV
# status: NXDOMAIN        ← 什么都没有
```

对照实验（`name: http`）：

```yaml
ports:
- name: http          # ← 有名字才有 SRV
  port: 80
  targetPort: 80
```

```
$ dig @10.96.0.10 _http._tcp.web-named.default.svc.cluster.local SRV
ANSWER SECTION:
_http._tcp.web-named.default.svc.cluster.local. 30 IN SRV 0 100 80 web-named.default.svc.cluster.local.
                                                        ↑  ↑   ↑   ↑
                                                    优先级 权重 端口  目标
```

**SRV 四段含义：**

| 字段 | 值 | 含义 |
|---|---|---|
| priority | 0 | 优先级，越小越优先 |
| weight | 100 | 权重，同优先级内按比例分配 |
| **port** | **80** | **★ A 记录给不了的信息** |
| target | `web-named...` | 提供该端口的主机名 |

**Day1 那道"多端口 Service 必须给每个 port 起 name"的题，现在有了第二层答案**：

```
ports[].name  →  决定 SRV 记录的键：_<name>._<proto>.<svc>.<ns>.svc.cluster.local
                                        ↑↑↑↑
```

**没名字 → 没有 SRV → 依赖 SRV 做服务发现的客户端（gRPC / SRV 感知的 LB / Headless 场景）就找不到你。**

## 六、dnsPolicy 四取值

| 取值 | `/etc/resolv.conf` 指向 | 场景 |
|---|---|---|
| **`ClusterFirst`**（默认） | **CoreDNS**（10.96.0.10） | 绝大多数情况 |
| **`Default`** | **继承节点**的 resolv.conf | 用宿主机 DNS |
| **`ClusterFirstWithHostNet`** | CoreDNS | **`hostNetwork: true` 的 Pod 必用** |
| **`None`** | 不填，**必须**手工配 `dnsConfig` | 完全自定义 |

### 🔥 hostNetwork 的坑（三组对照实测）

| Pod | 字段 `dnsPolicy` | `hostNetwork` | 实际 resolv.conf nameserver | 能解析集群 Service |
|---|---|---|---|---|
| `hostnet` | `ClusterFirst` | true | **192.168.10.2** | ❌ |
| 普通 Pod | `ClusterFirst` | false | **10.96.0.10** | ✅ |
| `hostnet-fixed` | **`ClusterFirstWithHostNet`** | true | **10.96.0.10** | ✅ |

**机制：**

```
hostNetwork: true → 共享节点网络命名空间
  → kubelet 看到 hostNetwork + ClusterFirst
  → 把 DNS 策略【降级为 Default】    ← 官方原文："fallback to the behavior of the Default policy"
  → 用节点的 DNS（192.168.10.2，VMware NAT 网关）
  → 完全不认识 cluster.local → 集群内服务名全部解析失败
```

**⚠️ 最重要的一课：字段值 ≠ 生效值**

```bash
kubectl get pod hostnet -o jsonpath='{.spec.dnsPolicy}'    # ClusterFirst  ← 字段这么说
kubectl exec hostnet -- cat /etc/resolv.conf               # 192.168.10.2  ← 实际这么干
```

**唯一可靠的判断方法**：进 Pod 看 `/etc/resolv.conf`，或直接 `nslookup` 试。

**CKA 意义**：题目给一个 hostNetwork Pod 说"无法解析集群服务"，正确动作是**加 `dnsPolicy: ClusterFirstWithHostNet`**。只重复写 `ClusterFirst` 会发现**改了没有任何变化**。

**一个细节：为什么是 `192.168.10.2` 而不是 `127.0.0.53`？**

```
节点的 /etc/resolv.conf          → nameserver 127.0.0.53（systemd-resolved 的本地 stub）
节点的 /run/systemd/resolve/resolv.conf → nameserver 192.168.10.2（真实上游）
kubelet 为 Pod 生成 resolv.conf 时会【剥掉回环地址】（127.0.0.0/8），改用真实上游
```

**`None` 的写法**（不配 `dnsConfig` 则 Pod 起不来）：

```yaml
spec:
  dnsPolicy: None
  dnsConfig:
    nameservers: ["10.96.0.10"]
    searches: ["default.svc.cluster.local"]
    options:
      - name: ndots
        value: "2"
```

## 七、Pod 域名与反向解析

### 正向（依赖 `pods insecure`）

```
10-244-85-239.default.pod.cluster.local → 10.244.85.239  ✅
```

若把 `pods insecure` 改成 `pods disabled`，Pod A 记录就完全不生成。

### 反向：Pod IP 反查返回的是「Service 视角」的名字 ⭐

```
$ dig @10.96.0.10 -x 10.244.85.239
239.85.244.10.in-addr.arpa. 30 IN PTR 10-244-85-239.web-cip.default.svc.cluster.local.
239.85.244.10.in-addr.arpa. 30 IN PTR 10-244-85-239.web-headless.default.svc.cluster.local.
239.85.244.10.in-addr.arpa. 30 IN PTR 10-244-85-239.web-np.default.svc.cluster.local.
239.85.244.10.in-addr.arpa. 30 IN PTR 10-244-85-239.web-named.default.svc.cluster.local.
```

- **格式是 `<IP 换横杠>.<svc>.<ns>.svc.cluster.local`，不是 Pod 自己的名字**
- **PTR 条数 = 有几个 Service 选中了这个 Pod**（此处 4 个）
- **用途**：反查 Pod IP 实际是在问「**这个 IP 属于哪些 Service**」

### 反向：ClusterIP 也能反查

```
$ dig @10.96.0.10 -x 10.111.19.171
171.19.111.10.in-addr.arpa. 30 IN PTR web-cip.default.svc.cluster.local.
```

**小结**：两种反查都返回 `svc.cluster.local` 作用域的名字 —— CoreDNS 的 kubernetes 插件**以 Service/Endpoints 为模型**组织数据，Pod IP 只有在"被某个 Service 选中"时才反查得出来。这也解释了为什么 Pod 记录要单独用 `pods insecure` 开。

## 八、⭐ CoreDNS 故障演练 + 排障三分表

```bash
# 1. 基线
dig @10.96.0.10 web-cip.default.svc.cluster.local +short

# 2. 备份 + 缩到 0
kubectl -n kube-system get deploy coredns -o yaml > /tmp/coredns-backup.yaml
kubectl -n kube-system scale deploy coredns --replicas=0

# 3. 观察瘫痪
kubectl -n kube-system get pods -l k8s-app=kube-dns
kubectl get endpoints -n kube-system kube-dns          # → <none>
dig @10.96.0.10 web-cip.default.svc.cluster.local      # → 失败

# 4. ★ 关键对比：IP 直连
kubectl exec <pod> -- wget -qO- --timeout=3 http://<podIP>   # → 仍然通！

# 5. 恢复
kubectl -n kube-system scale deploy coredns --replicas=2
kubectl -n kube-system rollout status deploy coredns
```

### 三个结论

**① Endpoints 变空** —— Service 还在、ClusterIP 还在，但 `kube-dns` 的 Endpoints 变成 `<none>`。

**② IP 直连仍然通** —— 这是最重要的产出：

```
DNS 挂了  →  翻译服务停摆  →  "web-cip" 这个输入没有输出了
IP 直连   →  压根不需要翻译  →  完全绕开 DNS

因为：kube-proxy 的 iptables 规则没变 / Pod 网络没变 / 容器进程没变
     CoreDNS 只是集群里一个【普通 Deployment】，它死了不影响别人的网络
```

**③ 不需要重启客户端** —— 因为 Corefile 里 `cache` 对 `cluster.local` 关掉了缓存：

```
CoreDNS 恢复 → 下一次查询直接问到 CoreDNS → 立刻拿到正确答案
              ↑ 没有任何缓存要等它过期

（若 cache 对 cluster.local 生效，恢复后还会有最长 30 秒窗口吃旧结果）
```

### 🎯 排障三分表（Day2 最重要的产物）

| 现象 | 含义 | 下一步查什么 |
|---|---|---|
| **秒拒（icmp-port-unreachable）** | Service 有 ClusterIP 但**无 endpoints** | `kubectl get endpoints` / selector 对不对 |
| **秒拒（RST / Connection refused）** | endpoints 有，但 **Pod 端口不对** | `targetPort` / 容器实际监听 |
| **超时** | **网络层不通** | CNI / NetworkPolicy / 路由表 |

**依据**（`svc-ready` 样本，ClusterIP `10.109.201.137`，Endpoints 空）：

```
$ sudo iptables -S | grep 10.109.201.137
-A KUBE-SERVICES -d 10.109.201.137/32 -p tcp \
   -m comment --comment "default/svc-ready has no endpoints" \
   -m tcp --dport 80 -j REJECT --reject-with icmp-port-unreachable
                     ↑ kube-proxy 自己写的注释，明说了原因

$ curl http://10.109.201.137:80
curl: (7) ... after 0 ms: Could not connect to server     ← 17 ms 秒回，不是超时
```

**规则落在哪张表：**

```
Service 无 endpoints 时，kube-proxy 分两个表干活：
  nat 表    → 不写（没有 DNAT 目标可写）
  filter 表 → 写一条 REJECT (icmp-port-unreachable)
            → 客户端立刻收到 ICMP "端口不可达"
```

## 九、DNS 排障七步

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns          # 1 CoreDNS 活着吗
kubectl get svc -n kube-system kube-dns                      # 2 Service 在吗
kubectl get endpoints -n kube-system kube-dns                # 3 ★ 有后端吗（最常出问题）
kubectl exec <pod> -- cat /etc/resolv.conf                   # 4 Pod 用的哪套 DNS
kubectl exec <pod> -- nslookup kubernetes.default            # 5 从 Pod 内能解析吗
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=20    # 6 CoreDNS 自己的日志
kubectl get cm coredns -n kube-system -o yaml                # 7 Corefile 被改坏了吗
```

**第 3 步是重点**：CoreDNS 的 Service 没有 endpoints，DNS 直接失效——而 `get svc` 看起来一切正常。

## 十、踩坑实录

### 1. Service 叫 `kube-dns`，不叫 `coredns`

考试找不到 CoreDNS 时，第一反应通常是 `get svc coredns` —— 没有。Deployment 才叫 `coredns`。

### 2. `dig` 不走 search 域 → 短名验证必须用 `getent`

在节点上 `dig @10.96.0.10 nginx-tns.test-ns` 失败**不代表名字错了**，只代表 dig 不做后缀拼接。节点的 `search .` 什么都没配。**验证短名要进 Pod 用 `getent hosts`。**

### 3. busybox 的 `nslookup` 也不走 search 域

比 dig 稍微好一点（会试几个后缀），但行为不可靠。**别用它验证短名。**

### 4. SRV 只为命名端口生成

端口没 `name` → `_http._tcp.<svc>` 查不到。这是"多端口必须起 name"的第二层原因。

### 5. `hostNetwork` + `ClusterFirst` = 实际 `Default`

字段值骗人。**只有 `/etc/resolv.conf` 会说真话。**

### 6. 无 endpoints 的 Service，REJECT 规则在 **filter 表**不在 nat 表

排查 iptables 时只搜 `-t nat` 会得出"完全没有规则"的错误结论。**两张表都要看。**

### 7. 反查 Pod IP 返回的是 Service 视角的名字

不是 Pod 自己的名字。PTR 条数 = 引用它的 Service 数量。

### 8. `dig +bufsize=<512` 无法触发截断

CoreDNS 会把小于 512 的 EDNS0 buffer 值当 512 处理。要触发 TC 必须让真实响应超 512 字节。

## 十一、复习自测题（答案）

1. **CoreDNS 的 Service 叫什么？暴露哪几个端口？为什么 DNS 要 TCP 和 UDP 两个 53？**
   - Service 叫 `kube-dns`（Deployment 才叫 coredns）
   - `53/UDP`（dns）+ `53/TCP`（dns-tcp）+ `9153/TCP`（metrics）
   - UDP 是常态路径；UDP DNS 有 512 字节限制，超出会置 TC 标志 → 客户端改用 TCP 重试 → **53/TCP 是兜底通道**

2. **约多少个后端会让 DNS 响应超过 512 字节？**
   - 实测 3 个后端 = 241 字节，每条约 16 字节 → **约 20 个**

3. **跨 namespace 访问，短名 `<svc>.<ns>` 能用吗？**
   - **只有走 libc 解析器的 Pod 能**（靠 search 域 `svc.cluster.local` 拼接）
   - 节点上的 `dig`、busybox `nslookup` 都不能
   - **跨 ns 一律写完整 FQDN 最稳**

4. **SRV 记录什么时候生成？格式是什么？**
   - 只在**端口有 `name`** 时生成
   - `_<name>._<proto>.<svc>.<ns>.svc.cluster.local`，返回 `优先级 权重 端口 目标`
   - 它补的是 A 记录给不了的**端口信息**

5. **`hostNetwork: true` 的 Pod 该用哪个 dnsPolicy？为什么？**
   - `ClusterFirstWithHostNet`
   - 因为 `hostNetwork` + `ClusterFirst` 会被 kubelet **降级为 `Default`**，用节点 DNS（不认识 cluster.local）

6. **`kubectl get pod` 显示的 dnsPolicy 能信吗？**
   - **不能**。hostNetwork 场景下字段显示 `ClusterFirst` 但实际生效 `Default`。要看 `/etc/resolv.conf`

7. **反查 Pod IP 得到什么？**
   - 该 IP 对应的**所有 Service 视角的名字**（`<IP 换横杠>.<svc>.<ns>.svc.cluster.local`）
   - 条数 = 引用它的 Service 数量

8. **CoreDNS 挂掉后，IP 直连为什么还能通？**
   - DNS 只做"名字→IP"翻译。IP 直连不需要翻译，且 kube-proxy 规则 / Pod 网络 / 容器进程都没变

9. **服务访问"秒拒"和"超时"分别说明什么？**
   - **秒拒（icmp-port-unreachable）** → Service 无 endpoints
   - **秒拒（RST）** → endpoints 有但 Pod 端口不对
   - **超时** → 网络层不通（CNI / NetworkPolicy / 路由）

10. **CoreDNS 恢复后客户端要做什么吗？**
    - 不需要。`cache` 对 `cluster.local` 关掉了缓存（`disable success/denial cluster.local`），恢复即生效

## 速记卡片

```bash
# ── 家底
kubectl -n kube-system get deploy coredns          # 2 副本
kubectl -n kube-system get svc kube-dns            # 叫 kube-dns！
kubectl -n kube-system get cm coredns -o jsonpath='{.data.Corefile}'

# ── 查 DNS（必须指定 CoreDNS IP；节点上 dig 必须用 FQDN）
dig @10.96.0.10 <svc>.<ns>.svc.cluster.local +short
dig @10.96.0.10 <svc>.<ns>.svc.cluster.local +tcp +short      # 强制 TCP
dig @10.96.0.10 -x <IP> +short                                # 反向
dig @10.96.0.10 _<port-name>._tcp.<svc>.<ns>.svc.cluster.local SRV

# ── Pod 内验证（短名只能用 getent，不能用 busybox nslookup / dig）
kubectl exec <pod> -- getent hosts <svc>
kubectl exec <pod> -- cat /etc/resolv.conf

# ── 命名规则
<svc>.<ns>.svc.cluster.local                       # Service
<pod>.<svc>.<ns>.svc.cluster.local                 # Headless + StatefulSet
<IP换横杠>.<ns>.pod.cluster.local                  # Pod
_<name>._<tcp|udp>.<svc>.<ns>.svc.cluster.local    # SRV（需命名端口）

# ── dnsPolicy
ClusterFirst（默认） / Default / ClusterFirstWithHostNet（hostNetwork 用）/ None
hostNetwork + ClusterFirst → 实际退化为 Default   ← 字段值骗人

# ── 排障三分表
秒拒 icmp-port-unreachable → 无 endpoints
秒拒 RST                   → 端口不对
超时                       → 网络层不通

# ── DNS 挂了
IP 直连仍通 → 问题在解析层，不在网络层
恢复 CoreDNS 后无需重启客户端（cluster.local 不缓存）
```
