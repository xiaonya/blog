---
title: "Gateway API"
sidebar_position: 5
---

# Gateway API

> 集群：k8s-master / k8s-node01 / k8s-node02（v1.36.3）  
> 实现：**NGINX Gateway Fabric 2.7.2**（控制面 + 数据面分离架构）  
> CRD：Gateway API **v1.6.2**（standard channel）

## 核心认知：Ingress 的继任者

Gateway API 不是为了多几个字段，而是为了解决 Ingress 的**结构性缺陷**。考纲 v1.35 明确要求（Services & Networking 第 4 条：`Use the Gateway API to manage Ingress traffic`），而 ingress-nginx 已于 2026-03 退役 —— 它是官方指定的迁移方向。

## 一、Ingress 的三个结构性缺陷

| 缺陷                                            | 后果                           |
| --------------------------------------------- | ---------------------------- |
| **高级功能全靠注解**（`nginx.ingress.kubernetes.io/*`） | 厂商私有，**不可移植**；同一个值不同实现行为可能不同 |
| **一个对象混了"基础设施配置"和"应用路由"**                     | 网管和开发者抢同一个资源，**权限无法切分**      |
| **只有 host + path**                            | 做不了 header 匹配、流量切分、跨 ns 路由   |

第 2 条最致命 —— Day3 讲过的"50 个应用、10 个团队编辑同一个 nginx.conf 是灾难"，Gateway API 就是官方对这个问题的回答。

## 二、★ 对象模型：四层 + 三种角色

```
GatewayClass      「集群支持哪种网关实现」      cluster 级   ← controller 装完自带
      ↓ 被引用
Gateway           「开哪个端口、什么协议、什么证书」  ← 运维 / 网管
      ↓ 被引用
HTTPRoute         「什么请求去哪个后端」              ← 应用开发者
      ↓ 指向
Service → Pod
```

**三个对象 = 三种角色 = 三种 RBAC 权限。** 这是 Gateway API 的设计精髓。

### 与 nginx.conf 的对照（理解拆分的关键）

| Gateway API                        | 对应 nginx.conf                      |
| ---------------------------------- | ---------------------------------- |
| **GatewayClass**                   | "用的是 nginx 还是 envoy"（选择实现）         |
| **Gateway `listeners[].port`**     | **`listen 80;`**                   |
| **Gateway `listeners[].protocol`** | 协议（HTTP / HTTPS / TLS / TCP / UDP） |
| **Gateway `listeners[].tls`**      | `ssl_certificate` + key            |
| **HTTPRoute `hostnames`**          | **`server_name demo.local;`**      |
| **HTTPRoute `matches[].path`**     | **`location /a { }`**              |
| **HTTPRoute `backendRefs`**        | **`upstream { server ...; }`**     |
| **HTTPRoute `filters`**            | `rewrite` / `proxy_set_header` 等   |
| HTTPRoute `parentRefs`             | 「这份 location 属于哪个 server 块」        |

> **一句话概括**：Ingress 把 `server {}` 和 `location {}` 混在一个对象里；Gateway API 把 `server {}` 那半边独立成 Gateway，`location {}` 那半边留在 HTTPRoute。

```
nginx.conf                        Gateway API
─────────────                     ──────────────
server {                          Gateway
    listen 80;                      └─ listeners: [{name: http, port: 80, protocol: HTTP}]

    location /a { }               HTTPRoute
        proxy_pass http://app-a;    └─ rules:
}                                        - matches:  [{path: {type: PathPrefix, value: /a}}]
                                           backendRefs: [{name: app-a, port: 80}]
```

## 三、与 Ingress 的完整对照

|           | Ingress                  | Gateway API                              |
| --------- | ------------------------ | ---------------------------------------- |
| API 类型    | **内置**（开箱可用）             | **CRD**（要装）                              |
| 基础设施 + 路由 | 混在**一个对象**               | 拆成 **Gateway + HTTPRoute**               |
| 监听端口      | 隐含（controller 统一 80/443） | **listener 显式定义**                        |
| 谁能挂路由     | 无限制                      | **`allowedRoutes` 控制**                   |
| 高级功能      | annotations（**厂商私有**）    | **原生字段**（filters）                        |
| 流量切分      | 靠注解                      | **`weight` 原生**                          |
| 跨命名空间路由   | 不支持                      | 支持（配 ReferenceGrant）                     |
| 数据面       | 全局一套（所有人共享）              | **每个 Gateway 一套**                        |
| 角色权限      | 无法切分                     | **可按对象切分 RBAC**                          |
| 状态字段      | 无（只能看 controller 日志）     | **Accepted / Programmed / ResolvedRefs** |
| 成熟度       | 已冻结 / controller 已退役     | 活跃开发，官方推荐                                |

### `pathType` 命名差异（★ 考试易错）

| Ingress                            | Gateway API                    | 含义    |
| ---------------------------------- | ------------------------------ | ----- |
| `pathType: Prefix`                 | **`path.type: PathPrefix`**    | 路径段前缀 |
| `pathType: Exact`                  | `path.type: Exact`             | 完全相等  |
| `pathType: ImplementationSpecific` | `path.type: RegularExpression` | 实现相关  |

**Ingress 的 `ImplementationSpecific`** 是给自己留的"逃生舱"：官方定义是"匹配方法取决于 IngressClass，实现可以把它当独立类型处理或按 Prefix/Exact 处理"。在 ingress-nginx 下：**不加注解时等价于 Prefix；加了 `nginx.ingress.kubernetes.io/use-regex: "true"` 时按正则匹配。**

**同一个值在不同 controller 下行为不同 —— 这就是"不可移植"的具体含义。** Gateway API 改名叫 `RegularExpression`，至少说清了意图。

**写错 `path.type` 的值会被 CEL 校验直接拒绝**（Gateway API 的 CRD 内建 CEL 校验，不需要 validating webhook）。

## 四、安装（★ 两层 CRD，本集群已踩过坑）

### 正确步骤

```bash
# ① 官方 Gateway API CRD（standard channel）
kubectl apply -f \
  https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.6.2/standard-install.yaml

# ② ★ 实现自己的 CRD —— 这一步最容易漏！
kubectl apply --server-side -f \
  https://raw.githubusercontent.com/nginx/nginx-gateway-fabric/v2.7.2/deploy/crds.yaml

# ③ controller manifest —— 裸金属必须把数据面 Service 改成 NodePort
curl -sLo /tmp/ngf.yaml \
  https://raw.githubusercontent.com/nginx/nginx-gateway-fabric/v2.7.2/deploy/default/deploy.yaml
sed -i 's/type: LoadBalancer/type: NodePort/' /tmp/ngf.yaml
kubectl apply -f /tmp/ngf.yaml

# ④ 验证
kubectl -n nginx-gateway get pods          # 1/1 Running
kubectl get gatewayclass                   # ACCEPTED: True
```

### 为什么是两层 CRD

| 层               | 内容                                                                                               | 来源                            |
| --------------- | ------------------------------------------------------------------------------------------------ | ----------------------------- |
| **① 官方标准 CRD**  | GatewayClass / Gateway / HTTPRoute / GRPCRoute / ReferenceGrant / TCPRoute / UDPRoute / TLSRoute | `kubernetes-sigs/gateway-api` |
| **② 实现自己的 CRD** | `NginxGateway` / `NginxProxy`（`gateway.nginx.org` 组）                                             | NGF 的 `deploy/crds.yaml`      |

NGF 用 `NginxProxy` 配置数据面的镜像、副本数、Service 类型 —— manifest 里用了这个对象，但**定义在单独的 crds.yaml 里**。只装第一层 → controller 找不到初始配置 → 直接崩。

### 镜像

| 组件  | 镜像                                               |
| --- | ------------------------------------------------ |
| 控制面 | `ghcr.io/nginx/nginx-gateway-fabric:2.7.2`       |
| 数据面 | `ghcr.io/nginx/nginx-gateway-fabric/nginx:2.7.2` |

**`ghcr.io` 直连可用**（实测）。注意 DaoCloud 的 ghcr 代理会拒绝（"这镜像不在白名单"），`docker.1ms.run` 只代理 docker.io。

## 五、Gateway 的 listener（Ingress 没有的概念）

Ingress 里**从来不用写"监听哪个端口"** —— controller 统一监听 80/443，所有 Ingress 共享。

Gateway 必须显式声明：

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: my-gw
  namespace: default
spec:
  gatewayClassName: nginx
  listeners:
    - name: http                    # listener 名，HTTPRoute 可用 sectionName 指定挂到哪个
      protocol: HTTP                # HTTP / HTTPS / TLS / TCP / UDP
      port: 80
      hostname: "*.example.com"     # 可选：限制这个 listener 接受哪些域名
      tls:                          # HTTPS 时用
        certificateRefs:
          - name: my-cert
      allowedRoutes:                # ★ 谁能往这个 listener 上挂路由
        namespaces:
          from: Same                # Same（只本 ns）/ All / Selector
```

**两个 Ingress 没有的关键点**：

1. **一个 Gateway 可以有多个 listener**（80 + 443 + 自定义端口），各自独立
2. **`allowedRoutes` 是显式的安全边界** —— 决定哪些命名空间的 HTTPRoute 被允许挂上来

第 2 条是杀手特性：运维可以给某团队开一个 listener，只允许他们的 ns 挂路由。Ingress 做不到。

## 六、HTTPRoute 字段全表

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-route
  namespace: default
spec:
  parentRefs:                    # ★ 挂到哪个 Gateway
    - name: my-gw
      namespace: default         # 跨 ns 时需要（且要在 Gateway 的 allowedRoutes 里被许可）
      sectionName: http          # 挂到哪个 listener（可选）
  hostnames:                     # 相当于 server_name
    - "demo.local"
  rules:                         # ★ 按声明顺序匹配，第一个命中的规则生效
    - matches:
        - path:
            type: PathPrefix     # ★ PathPrefix / Exact / RegularExpression
            value: /a
        - headers:               # 原生能力，Ingress 只能靠注解
            - name: X-Canary
              value: "true"
      filters:                   # ★ 过滤器，Ingress 里叫注解
        - type: URLRewrite
          urlRewrite:
            path:
              type: ReplacePrefixMatch
              replacePrefixMatch: /
      backendRefs:               # ★ 复数，数组
        - name: app-a
          port: 80
          weight: 80             # ★ 流量切分，Ingress 做不到
        - name: app-b
          port: 80
          weight: 20
```

### 常用 filters

| filter 类型                    | 作用           | Ingress 对应                 |
| ---------------------------- | ------------ | -------------------------- |
| `URLRewrite`                 | 重写路径         | `rewrite-target` 注解        |
| `RequestHeaderModification`  | 改请求头         | `configuration-snippet` 注解 |
| `ResponseHeaderModification` | 改响应头         | 同上                         |
| `RequestRedirect`            | 重定向（301/302） | `permanent-redirect` 注解    |
| `RequestMirror`              | 流量镜像         | `mirror-target` 注解         |
| `ExtensionRef`               | 引用实现特有的 CRD  | 无                          |

**全部是标准字段** —— 这是可移植性的来源。

## 七、★ 流量链路（最容易误解的地方）

```
① 外部客户端
   curl -H "Host: demo.local" http://192.168.10.13:32401/
                  ↓
② 数据面 Service（NodePort 32401）   ← 控制面自动创建，不是你建的
                  ↓
③ 数据面 Pod 里的 nginx              ← ★ 七层决策在这里
   读 Host: demo.local + path，匹配 HTTPRoute 的规则
   配置从哪来？→ 控制面通过 gRPC 推过来的
                  ↓
④ 按匹配结果转发到 upstream：app-a 的 Service
                  ↓
⑤ 后端 Pod
```

### ★ 最关键的一句话：流量不经过控制面

```
        ┌──────────────────────────────┐
        │  控制面（nginx-gateway）      │  ← 只在"配置变了"时工作
        │  watch API → 生成配置          │     不转发任何数据包
        └──────────────┬───────────────┘
                       │ gRPC 推配置
                       ↓
  客户端 ──→ 数据面（my-gw-nginx 里的 nginx）──→ 后端 Pod
                       ↑
              实际流量走这条线
```

控制面像**配网工程师**：他在机房外面算好配置，通过带外通道（gRPC）推给交换机（数据面），然后去喝茶 —— 包还是交换机在转。

（Day3 的 ingress-nginx 同理：**数据包从来不经过 Go controller 进程**，是 nginx 进程在转。区别只是那时两者在同一个 Pod 里。）

### 控制面 / 数据面是网络设备的通用分层

|         | 干什么         | 碰不碰真实流量 |
| ------- | ----------- | ------- |
| **控制面** | 做决策、算配置、下指令 | **不碰**  |
| **数据面** | 按指令干活、转发流量  | **碰**   |

**路由器类比**：OSPF/BGP 进程算路由表 = 控制面；转发芯片按表转发 = 数据面。控制面挂了，已有路由表还在，数据面照样转发。

### NGF 2.x 的架构（与 ingress-nginx 对比）

|               | ingress-nginx                      | NGF 2.x                     |
| ------------- | ---------------------------------- | --------------------------- |
| 结构            | 一个 Pod，两个进程（Go controller + nginx） | **两个 Deployment**（跨命名空间）    |
| 数据面数量         | 全局一套（所有 Ingress 共享）                | **每个 Gateway 一套**           |
| 配置传递          | 同 Pod 内写文件 + reload                | **跨 Pod gRPC**（NGINX Agent） |
| 控制面挂了         | 整个 Pod 受影响                         | **已有流量照常转发**                |
| 多团队隔离         | 无                                  | **每个 Gateway 独立数据面**        |
| 数据面需要 K8s 权限吗 | 需要（同一 Pod 内 controller 要 watch）    | **不需要**（只接收 gRPC）           |

**拆开的四个好处**：故障隔离 / 按 Gateway 隔离 / 独立伸缩 / **安全（数据面无需 API 权限）**。

## 八、控制面原理

```
控制面（nginx-gateway Pod）
  1. WATCH   ── GatewayClass / Gateway / HTTPRoute / Service / EndpointSlice / Secret
  2. 校验    ── 引用是否有效（Gateway 存在吗、Service 存在吗、listener 允许吗）
  3. 组装    ── Gateway 的 listener + 所有关联 HTTPRoute 的 rules 合并成一份配置
  4. 生成    ── 渲染成 nginx 配置
  5. 下发    ── gRPC 推给数据面 Pod 里的 NGINX Agent
  6. reload  ── Agent 写文件 + 触发 nginx reload
  7. ★ 回写 status
```

数据面日志里的证据：

```
msg="Sending data plane response message" message="Config apply successful" status=COMMAND_STATUS_OK
```

### ★ status 字段（排障第一手资料）

| 字段                       | 含义                       | 什么时候是坏的                     |
| ------------------------ | ------------------------ | --------------------------- |
| `GatewayClass.ACCEPTED`  | controller 认领了这个 class 吗 | controller 没起来 → `Unknown`  |
| `Gateway.PROGRAMMED`     | 数据面真的配好了吗                | 配置下发失败 → `False`            |
| `HTTPRoute.Accepted`     | 路由被网关接受了吗                | 挂错 Gateway / 端口冲突 → `False` |
| `HTTPRoute.ResolvedRefs` | 引用的 Service 存在吗          | **Service 名写错 → `False`**   |

**`ResolvedRefs: False` 是考试和工作里最常见的坑** —— Service 名打错时，HTTPRoute 语法完全合法、`apply` 也成功，但就是不生效，**只有这个字段会告诉你**。

> 这些 status 由 **controller 填写**（不是 apiserver），和 Service 的 Endpoints、Ingress 的 ADDRESS 同一模式。

## 九、实验 A：path 路由 + URLRewrite

```yaml
spec:
  parentRefs: [{name: my-gw}]
  hostnames: ["lab.local"]
  rules:
    - matches: [{path: {type: PathPrefix, value: /a}}]
      filters:
        - type: URLRewrite
          urlRewrite:
            path: {type: ReplacePrefixMatch, replacePrefixMatch: /}
      backendRefs: [{name: app-a, port: 80}]
    - matches: [{path: {type: PathPrefix, value: /b}}]
      filters:
        - type: URLRewrite
          urlRewrite:
            path: {type: ReplacePrefixMatch, replacePrefixMatch: /}
      backendRefs: [{name: app-b, port: 80}]
```


### 实测结果

| 路径        | 结果                   | 含义                                    |
| --------- | -------------------- | ------------------------------------- |
| `/a`      | `<<< I am APP-A >>>` | ✅ Prefix 匹配 + 重写生效                    |
| `/a/xx`   | 404                  | 匹配上，但重写后是 `/xx` → **后端 nginx 找不到文件**  |
| **`/ab`** | **404**              | ★ **没匹配上** —— `PathPrefix` 按**路径段**匹配 |
| `/b`      | `<<< I am APP-B >>>` | ✅                                     |
| `/b/yy`   | 404                  | 同 `/a/xx`                             |
| `/c`      | 404                  | 无规则命中                                 |

**`/ab` 返回 404 是核心证据**：`PathPrefix` 的语义和 Ingress 的 `Prefix` **完全一致**（element-wise，按 `/` 分段），不是字符串前缀。规则没变，只是换了名字。

### `ReplacePrefixMatch` 的精确语义

**只替换匹配到的那一段前缀**：

```
/a/xx  →  匹配到的前缀是 /a
       →  替换成 /
       →  发给后端的是 /xx        ← 不是 /
```

### ★ Gateway API 不会自动去掉 path 前缀

如果删掉 `filters` 那一段，访问 `/a` 会返回 **404** —— 因为请求原样 `/a` 发给后端，后端 nginx 在根目录找 `a` 这个**文件**，找不到。

**想重写必须显式写 `URLRewrite` filter。**（这和 Day3 的 ingress-nginx 行为一致，那个坑叫"后端应用的 404 与路由未命中的 404 要区分"。）

## 十、实验 B：weight 流量切分

```yaml
  rules:
    - matches: [{path: {type: PathPrefix, value: /}}]
      backendRefs:
        - {name: app-a, port: 80, weight: 80}
        - {name: app-b, port: 80, weight: 20}
```

统计（★ curl 后必须补 `echo`，否则输出无换行、`uniq -c` 会失效）：

```bash
for i in $(seq 1 60); do
  curl -s -H "Host: demo.local" http://192.168.10.13:32401/
  echo
done | sort | uniq -c
```

**实测：60 次请求 → APP-A 50 次 / APP-B 10 次 ≈ 83% : 17%**，与配置的 80:20 吻合 ✅

**同一个 Host、同一个路径，按权重分流** —— 这在 Ingress 里只能靠 `canary` 注解（厂商私有），Gateway API 是标准字段。

## 十一、信息查询地图

| 想知道什么                | 查哪里                                                                |
| -------------------- | ------------------------------------------------------------------ |
| 集群支持哪些网关实现           | `kubectl get gatewayclass`                                         |
| 网关列表 + 是否就绪          | `kubectl get gateway -A`（看 `PROGRAMMED`）                           |
| 网关**声明**监听什么         | `kubectl get gateway my-gw -o yaml` → `spec.listeners`             |
| **实际访问端口（nodePort）** | **数据面 Service** ← ★ 不在 Gateway 上                                   |
| 网关挂了几条路由             | `kubectl describe gateway my-gw` → `Attached Routes`               |
| 路由健康状态               | `kubectl describe httproute my-route` → `Status.Parents`           |
| 数据面 Pod 在哪           | `kubectl get pods -l gateway.networking.k8s.io/gateway-name=my-gw` |

### ★ NodePort 不在 Gateway 上

```
Gateway my-gw 的 ADDRESS 列 = 10.101.121.63   ← ClusterIP，不是访问端口！
真正的 nodePort = 32401                        ← 在【数据面 Service】上
```

**Gateway 声明的是"意图"（监听 80），不是"实现"（怎么暴露出去）。**

```
Gateway 对象              ← 声明「监听 80 端口、协议 HTTP」   （逻辑，你要写的）
      ↓ 控制面翻译
数据面 Deployment + Service ← 实现产物「NodePort 32401」      （物理，自动生成的）
```

同一份 Gateway YAML，在云上变成 LoadBalancer（公网 IP），在裸金属变成 NodePort（随机端口），在 hostNetwork 模式下直接占节点 80。**这是它比 Ingress 更可移植的原因。**

### 常用命令

```bash
# 网关全景
kubectl get gatewayclass,gateway,httproute -A

# ★ 一行拿 nodePort
NP=$(kubectl get svc -l gateway.networking.k8s.io/gateway-name=my-gw \
     -o jsonpath='{.items[0].spec.ports[0].nodePort}')

# 按标签捞出某个网关的全部数据面资源
kubectl get all -l gateway.networking.k8s.io/gateway-name=my-gw

# 路由状态（排障必看）
kubectl describe httproute my-route | grep -A20 'Status:'
```

## 十二、踩坑实录

### 1. ★ 漏装实现自己的 CRD（两层 CRD）

只装官方 `standard-install.yaml` → controller 崩溃：

```
no matches for kind "NginxGateway" in version "gateway.nginx.org/v1alpha1"
```

Gateway API 是**两层 CRD**：官方标准 CRD + **实现自己的 CRD**。NGF 用 `NginxGateway` / `NginxProxy` 配置自己。

### 2. ★ apply 顺序错了导致对象静默漏建

CRD 补装后仍失败：

```
NginxGateway nginx-gateway/nginx-gateway-config not found: context deadline exceeded
```

原因：**第一次 apply manifest 时那个 CRD 还不存在**，`NginxGateway` / `NginxProxy` 两个对象 apply 失败，**被静默漏掉**。

**`kubectl apply` 提交多对象时，个别失败不影响其他对象** —— 命令返回非零，但已成功的部分照常生效。

**修法**：补装 CRD 后**重新 apply 一次 manifest**。

### 3. ★ 错误信息的判别法（排障利器）

```
no matches for kind X     → CRD 层：这个资源类型不存在
X "name" not found        → 对象层：类型有，但实例没建
+ context deadline exceeded → controller 在等它，等到超时
```

### 4. 数据面 Service 默认 LoadBalancer，裸金属下永远 pending

必须 `sed -i 's/type: LoadBalancer/type: NodePort/'` 改 manifest。

### 5. NGF 的 CRD 必须用 `--server-side` apply

`deploy/crds.yaml` 有 1MB+，客户端 apply 会因 annotation 超过 256KB 报错。

### 6. 镜像源

`ghcr.io` **直连可用**（本集群实测）。**DaoCloud 的 ghcr 代理会拒绝**（不在白名单）；`docker.1ms.run` 只代理 docker.io。

### 7. Gateway 的 ADDRESS 是 ClusterIP，不是 nodePort

NodePort 模式下没有单一入口 IP，NGF 填的是数据面 Service 的 ClusterIP。真正的端口在**数据面 Service** 上。

### 8. ★ `pathType` 改名了

`Prefix` → **`PathPrefix`**。写错会被 CEL 校验直接拒绝。

### 9. ★ Gateway API 不会自动去 path 前缀

想重写路径必须显式写 `URLRewrite` filter，否则后端会收到完整原路径。

### 10. GatewayClass 的 `ACCEPTED: Unknown` 不是失败，是"没人认领"

controller 崩了 → 没人填写 status → 一直 `Unknown`。修好后自动变 `True`。

### 11. curl 输出无换行导致统计失效

`<<< I am APP-A >>>` 没有换行符，60 次输出挤成一行，`sort | uniq -c` 完全失效。**循环里 curl 后补 `echo`。**

### 12. 多条 HTTPRoute 可以挂在同一个 Gateway 上

用不同 `hostnames` 区分即可。**它们共享同一个数据面**（这也是与 Ingress 多 Ingress 共享 controller 的相似点）。

## 十三、复习自测题（答案）

1. **Gateway API 的三个核心对象是什么？分别对应什么角色？**
   - **GatewayClass**（基础设施提供者）/ **Gateway**（集群运营者）/ **HTTPRoute**（应用开发者）
   - 三个对象分别可切分 RBAC 权限
2. **Gateway API 是内置 API 还是 CRD？**
   - **CRD**。而且分两层：官方标准 CRD + **实现自己的 CRD**
3. **Ingress 的 `pathType: Prefix` 在 HTTPRoute 里对应什么？**
   - **`path.type: PathPrefix`** —— 名字不同，写错会被 CEL 拒绝
4. **Gateway 里的"监听端口"在哪定义？**
   - **Gateway 的 `spec.listeners[].port`**。Ingress 里没有这个概念（controller 统一监听 80/443）
5. **`allowedRoutes` 是干什么的？**
   - 控制**哪些命名空间的 HTTPRoute 允许挂到这个 listener 上** —— Ingress 没有的安全边界
6. **NodePort 端口在哪查？为什么不在 Gateway 对象上？**
   - 在**数据面 Service** 上
   - Gateway 只声明"监听 80"这个**意图**，怎么暴露（LB / NodePort / hostNetwork）是 controller 生成的**实现**
7. **数据包会经过控制面吗？**
   - **不会**。控制面只在配置变更时工作，通过 gRPC 把配置推给数据面；**实际流量是 客户端 → 数据面 → 后端**
8. **HTTPRoute 的 `ResolvedRefs: False` 说明什么？**
   - 引用的 Service 不存在（通常名字打错）。**语法合法、apply 成功，但不生效**，只有这个字段会提示
9. **怎么让 `/a` 开头的请求到后端时变成 `/`？**
   - 加 `URLRewrite` filter，`path.type: ReplacePrefixMatch`，`replacePrefixMatch: /`
   - **Gateway API 不会自动去前缀**
10. **`PathPrefix` 是字符串前缀还是按路径段？**
    - **按路径段**（element-wise）。实测 `/ab` **不**匹配 `/a` —— 和 Ingress 的 `Prefix` 行为一致
11. **weight 流量切分的写法？Ingress 能做到吗？**
    - `backendRefs` 里多个条目各写 `weight`
    - **Ingress 做不到**（只能靠 canary 注解，厂商私有）
12. **Ingress 的 `ImplementationSpecific` 是什么？Gateway API 的对应物？**
    - "匹配方法取决于 IngressClass"的**逃生舱**
    - ingress-nginx 下：不加注解 ≈ Prefix；加 `use-regex` 注解 → 正则匹配
    - Gateway API 对应 **`RegularExpression`**（命名更直白）
13. **每个 Gateway 会创建哪些数据面资源？**
    - Deployment + Service + ServiceAccount + Secret(agent-tls) + 2×ConfigMap
    - 全部由控制面动态生成，带标签 `gateway.networking.k8s.io/gateway-name=<网关名>`

## 速记卡片

```bash
# ── 安装（三层顺序不能换）
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.6.2/standard-install.yaml
kubectl apply --server-side -f https://raw.githubusercontent.com/nginx/nginx-gateway-fabric/v2.7.2/deploy/crds.yaml
curl -sLo /tmp/ngf.yaml https://raw.githubusercontent.com/nginx/nginx-gateway-fabric/v2.7.2/deploy/default/deploy.yaml
sed -i 's/type: LoadBalancer/type: NodePort/' /tmp/ngf.yaml   # 裸金属必须
kubectl apply -f /tmp/ngf.yaml

# ── 查信息
kubectl get gatewayclass,gateway,httproute -A
kubectl describe gateway my-gw | grep -A10 'Listeners:'        # 挂了几条路由
kubectl describe httproute my-route | grep -A20 'Status:'      # ★ 排障必看
kubectl get all -l gateway.networking.k8s.io/gateway-name=my-gw # 数据面全景
NP=$(kubectl get svc -l gateway.networking.k8s.io/gateway-name=my-gw \
     -o jsonpath='{.items[0].spec.ports[0].nodePort}')          # ★ 真正的入口端口

# ── pathType 对照（★ 易错）
Ingress: pathType: Prefix            →  Gateway: path.type: PathPrefix
Ingress: pathType: Exact             →  Gateway: path.type: Exact
Ingress: pathType: ImplementationSpecific → Gateway: path.type: RegularExpression

# ── 对象对照
IngressClass  →  GatewayClass
Ingress       →  Gateway（listener）+ HTTPRoute（rules）
annotations   →  filters（标准字段）

# ── 关键认知
流量不经过控制面：客户端 → 数据面 → 后端
控制面只做：watch → 校验 → 组装 → 生成 → gRPC 下发 → 回写 status
Gateway 声明意图（监听 80），数据面 Service 才是实现（NodePort 32401）
每个 Gateway 一套独立数据面（Ingress 是全局共享一套）
Gateway API 不会自动去 path 前缀 → 要用 URLRewrite filter
weight 是标准字段 → Ingress 做不到

# ── 排障错误信息判别
no matches for kind X        → CRD 层不存在
X "name" not found           → 资源类型有，实例没建
+ context deadline exceeded  → controller 在等它，超时

# ── status 字段
GatewayClass.ACCEPTED     controller 认领了吗（Unknown = controller 没起来）
Gateway.PROGRAMMED        数据面配好了吗
HTTPRoute.Accepted        路由被接受了吗
HTTPRoute.ResolvedRefs    ★ 引用的 Service 存在吗（False = 名字写错）
```

---

## Week 5 全景小结（Services & Networking 域，20%）

| Day  | 主题                             | 核心产出                                                              |
| ---- | ------------------------------ | ----------------------------------------------------------------- |
| Day1 | Service / Endpoints / Headless | 三层端口映射、五种 Service 类型、Headless 的 DNS 本质、Endpoints vs EndpointSlice |
| Day2 | CoreDNS / 服务发现                 | Corefile 逐行、短名判定表、SRV、dnsPolicy、**排障三分表**                         |
| Day3 | Ingress / Ingress Controller   | 规则 vs 执行者、控制循环、pathType 八路径实测、404 三层来源、TLS + Fake Certificate     |
| Day4 | NetworkPolicy                  | 三铁律、**Egress DNS 坑**、AND/OR 实测对照、DNAT 与策略检查顺序、ipBlock             |
| Day5 | Gateway API                    | 三角色模型、两层 CRD、listener、URLRewrite、weight 切分、status 验证              |

**Week5 贯穿的一条主线**：**每一层都要问"谁在做决策、谁在执行"**

```
Service       决策：selector 选谁   |  执行：内核 DNAT
Ingress       决策：Ingress 规则    |  执行：controller 的 nginx
NetworkPolicy 决策：策略规则        |  执行：CNI（Calico）的内核规则
Gateway API   决策：Gateway+HTTPRoute | 执行：数据面的 nginx（配置由控制面推）
```

**外加一条排障主线**（三种"不通"的判别）：

```
bad address / no servers could be reached  → DNS 被拒（Egress 策略漏放行 53）
connection timed out                       → 网络层 DROP（NetworkPolicy / 路由 / CNI）
connection refused                         → 目标端口没监听（应用 / targetPort）
```
