---
title: "Ingress 与 Ingress Controller"
sidebar_position: 3
---

# Ingress 与 Ingress Controller

> 集群：k8s-master / k8s-node01 / k8s-node02（v1.36.3，Calico v3.29.1）
> controller：ingress-nginx **v1.15.1**（该项目已于 2026-03 退役，1.15.1 是最后一批版本之一）

## 核心认知：Ingress 是「规则」，Controller 是「执行者」

```
客户端 → Ingress Controller（Pod，读规则、跑 nginx）→ Ingress 规则 → Service → Pod
              ↑ 要单独部署                ↑ 你写的 YAML 在这
```

**Ingress 资源本身什么都不做。** 没有 controller 在集群里读它，它就是一份躺在 etcd 里没人看的文档。

### 用「总机 vs 前台」理解

| 现实 | K8s | 说明 |
|---|---|---|
| 总机（只认号码） | **Service** | 四层，只认 IP:port，听不懂"我找市场部" |
| 前台（听得懂你说什么） | **Ingress Controller** | 七层，看 Host / path 决定去哪 |
| 前台的 SOP 手册 | **Ingress 资源** | 手册本身不会走路 |

### 用 nginx 视角理解（同一件事的两种写法）

| 你以前手写的 | 现在拆给三个 K8s 对象 |
|---|---|
| nginx 进程本身（监听 80/443） | `ingress-nginx-controller`（Deployment/Pod） |
| nginx.conf 的 `server` / `location` | **Ingress 资源** |
| nginx.conf 的 `upstream` 块 | Service + EndpointSlice |

## 一、Ingress 与 Service 的层次分工

| | Service | Ingress |
|---|---|---|
| OSI 层 | **L4**（TCP/UDP） | **L7**（HTTP/HTTPS） |
| 在 nginx.conf 里对应 | `upstream {}` | `server {}` + `location {}` |
| 路由依据 | 只有 IP:port | **Host / path / header** |
| 执行者 | **内核**（iptables/nftables DNAT） | **用户态 nginx 进程** |
| 执行者部署位置 | 每个节点的 kube-proxy | 一个普通 Pod（需自己装） |
| 一个对象承载几个后端 | 1 组（selector 选出的） | **N 组**（rules 里任意多条） |
| TLS 终止 | 不支持 | 支持 |
| 后端能不能写 Pod IP | 能（selector） | **不能，只能写 Service** |

**一句话分工**：Service 解决「怎么找到一组 Pod」，Ingress 解决「什么样的 HTTP 请求该去哪组 Pod」。

### 没有 Ingress 会怎样（它到底解决什么）

3 个应用要对外暴露，只能用 NodePort：

```
应用A → 31629    应用B → 32001    应用C → 31111
```

三个痛点：
1. **端口号和随机数一样**，客户端要记三个 30xxx
2. **不能按域名分流** —— `a.com` / `b.com` 指向同一个 IP:80，NodePort 只看 IP:port，分不出来
3. **不能用标准 80/443** —— NodePort 取值范围被限制在 30000-32767

根因是同一个：**Service 是四层的，不解析 HTTP，看不懂 Host 头。**

Ingress 补的就是这一层：

```
                    ┌─ Host: a.local  → app-a
客户端 → :30282 ────┼─ Host: b.local  → app-b
                    └─ Host: demo.local → web-cip
```

### 为什么非它不可（三堵墙）

**墙 1：upstream 的 IP 一直在变**
传统 `server 10.244.85.196:80;` —— Pod 一重建 IP 作废，得改配置 + reload。
Ingress 里写 `service.name: web-cip`，由 Service/EndpointSlice 持续跟踪。

**墙 2：配置没有地方存**
K8s 里 nginx 跑在容器里，Pod 一重建配置回到镜像默认值，不可能把业务路由写进镜像。

**墙 3（真正的答案）：配置归属权无法切分**
50 个应用、10 个团队，传统模式下所有人编辑同一个 nginx.conf —— 冲突、误覆盖、reload 风暴、权限无法分。
Ingress 模式下每个团队 apply 自己的 Ingress 对象，**controller 负责合并成一份 nginx.conf**。

> 这就是声明式思想在入口网关上的落地：**关注点分离 + 控制器收敛**。不是写不出那份 nginx.conf，是这个协作模型在传统模式下不成立。

## 二、部署 Ingress Controller

### ⚠️ 环境坑：registry.k8s.io 直连不通

```
$ sudo crictl pull registry.k8s.io/ingress-nginx/controller:v1.15.1
... Head "https://europe-west4-docker.pkg.dev/v2/k8s-artifacts-prod/images/
    ingress-nginx/controller/manifests/v1.15.1": dial tcp 142.250.99.82:443:
    connect: connection refused
```

`registry.k8s.io` 会 302 重定向到 Google 的 `europe-west4-docker.pkg.dev`，然后被拒。

**可用路径（实测通过）**：

| 地址 | 结果 |
|---|---|
| `m.daocloud.io/registry.k8s.io/ingress-nginx/controller:v1.15.1` | ✅ 116MB |
| `docker.1ms.run/ingress-nginx/controller` | ❌ not found（1ms.run 只代理 docker.io） |
| `docker.1ms.run/registry.k8s.io/...` | ❌ |
| `docker.m.daocloud.io/registry.k8s.io/...` | ❌（域名是 `m.daocloud.io`，不带 docker 前缀） |
| `registry.aliyuncs.com/google_containers/ingress-nginx-controller` | ❌ authorization failed |

本集群 containerd mirror 现状：`/etc/containerd/certs.d/` 下**只有 `docker.io/hosts.toml`**，配了 3 个 mirror（docker.1ms.run / docker.m.daocloud.io / docker.xuanyuan.me）。`registry.k8s.io` 没配 mirror。

### 部署步骤

```bash
# 1. 下载官方 manifest
curl -sLo /tmp/ingress-nginx.yaml \
  https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.15.1/deploy/static/provider/baremetal/deploy.yaml

# 2. 看原始镜像地址
grep -n 'image:' /tmp/ingress-nginx.yaml

# 3. 替换成可拉取的地址（一条搞定，manifest 里有 controller 和 certgen 两个镜像）
sed -i 's|registry.k8s.io/|m.daocloud.io/registry.k8s.io/|g' /tmp/ingress-nginx.yaml

# 4. 部署
kubectl apply -f /tmp/ingress-nginx.yaml

# 5. 观察
kubectl get pods -n ingress-nginx -w
```

### 部署后的家底

| 对象 | 类型/值 | 作用 |
|---|---|---|
| Deployment `ingress-nginx-controller` | 1/1 Running | 干活的，里面跑 nginx |
| Service `ingress-nginx-controller` | **NodePort 80:30282 / 443:31233** | **流量入口** |
| Service `ingress-nginx-controller-admission` | ClusterIP 443 | **不是入口**！是 apiserver 回调的 webhook，校验 Ingress 合法性 |
| IngressClass `nginx` | controller=`k8s.io/ingress-nginx` | 声明"我是 nginx 这一类 controller" |
| ConfigMap `ingress-nginx-controller` | — | 全局 nginx 参数 |
| Secret `ingress-nginx-admission` | — | webhook 的 TLS 证书 |
| ServiceAccount + ClusterRole + Binding | — | controller 的 RBAC，它必须有权 watch Ingress/Service/EndpointSlice/Secret |
| Job × 2（admission-create / admission-patch） | `ttlSecondsAfterFinished: 0` | 跑完**立即自删**，所以看不到 |

```bash
kubectl -n ingress-nginx get all,cm,secret,sa
kubectl get ingressclass
kubectl -n ingress-nginx get svc -o wide
```

**IngressClass 的作用**：集群里可以同时跑多个 controller 实现（nginx / traefik / contour）。`ingressClassName: nginx` 就是告诉系统"这条规则给 nginx 那个 controller 处理"。只装一个时感觉多余，但漏写会走默认 class —— 没有默认 class 就完全不生效。

## 三、端口归属：30282 属于谁？

**属于 controller 的 Service，跟 Ingress、IngressClass 都无关。**

| 你做的事 | 30282 会变吗 | 原因 |
|---|---|---|
| 新建 Ingress（任意多个） | ❌ 不变 | Ingress 没有网络实体 |
| 新建 IngressClass | ❌ 不变 | 只是声明，无实体 |
| 装第二个 controller | ❌ 不变（但它自己占新端口） | Service 之间互相独立 |
| 改 ingress-nginx 的 Service | ✅ **会变** | 30282 只属于这个 Service |

**所有 Ingress 规则共用同一个入口端口** —— 这正是 Ingress 的设计目的。如果每条规则都要新端口，Ingress 就没有存在意义了。

改端口的正确方式（动 Service，不动 Ingress）：

```bash
kubectl -n ingress-nginx patch svc ingress-nginx-controller --type merge \
  -p '{"spec":{"ports":[{"name":"http","port":80,"nodePort":30080},
                        {"name":"https","port":443,"nodePort":30443}]}}'
```

### IngressClass 只是声明，controller 才是实现

```bash
kubectl create ingressclass nginx2 --controller=k8s.io/ingress-nginx
```

这样建的 class **什么都不变** —— 没有 controller 会去认领它。用它写的 Ingress 彻底不生效。
**这句话值得记住：只有声明没有实现，规则就是废纸。**

## 四、控制循环原理

### 一个 Pod，两个角色

```bash
kubectl -n ingress-nginx exec $CP -- ps aux | grep -E 'nginx|controller'
```

- **controller 进程（Go，主进程）** —— spawn 出 nginx，然后一直盯着 K8s API
- **nginx 进程** —— 真正转发流量的

**controller 是大脑，nginx 是手脚。**

### 循环

```
1. WATCH   ── 长连接监听 Ingress / Service / EndpointSlice / Secret / ConfigMap
                  ↓
2. 翻译    ── 用模板引擎（nginx.tmpl）把 K8s 对象渲染成 nginx.conf
                  ↓
3. RELOAD  ── 让 nginx 加载新配置
                  ↓
4. 回到 1
```

**三个要点：**

1. **是 watch，不是轮询** —— 长连接事件流，apiserver 一有变化就推过来，所以 apply 后秒级生效
2. **是「期望状态收敛」（reconcile loop），不是命令式执行** —— 每次问"声明的样子"和"现在跑的样子"一致吗，不一致就修。和 Deployment 的 ReplicaSet controller 同一思想
3. **reload 不是每次都做**：
   - 路由规则变了（server_name / location）→ 重新生成 nginx.conf + **reload**
   - 后端端点变了（Pod IP 增删）→ 走 nginx 内嵌 Lua 的**共享内存**动态更新，**通常不 reload**

### YAML → nginx.conf 的真实对照（实测）

你写的：

```yaml
rules:
  - host: demo.local
    http:
      paths:
        - path: /
          pathType: Prefix
          backend: {service: {name: web-cip, port: {number: 80}}}
```

controller 生成（`kubectl -n ingress-nginx exec $CP -- grep -n 'server_name' /etc/nginx/nginx.conf`）：

```
197:		server_name "_" ;              ← 兜底 server
337:		server_name "demo.local" ;     ← ★ 你的 host 变成了这一行
```

**`host`→`server_name`、`path`→`location`、`backend.service`→`upstream`。**

## 五、Ingress 资源字段全表

```yaml
apiVersion: networking.k8s.io/v1        # 内置 API 组，不是 CRD
kind: Ingress
metadata:
  name: web-ing
  namespace: default
spec:
  ingressClassName: nginx               # ★ 交给哪个 controller
  defaultBackend: {...}                 # 没有规则命中时的兜底（可选）
  tls:                                  # TLS 证书
    - hosts: [tls.local]
      secretName: tls-local             # 必须是 kubernetes.io/tls 类型
  rules:
    - host: demo.local                  # 七层路由第一级
      http:
        paths:
          - path: /                     # 七层路由第二级
            pathType: Prefix            # ★ 必填，不写通不过校验
            backend:
              service:
                name: web-cip           # ★ 只能是 Service，不能写 Pod
                port:
                  number: 80            # 或 name: http
```

### Ingress 是内置对象，不是 CRD

```bash
kubectl get ingress -A                  # 能查（只是空的）→ 资源存在
kubectl get crd | grep -i ingress       # 空 → 证明不是 CRD
kubectl api-resources | grep -i ingress
```

K8s API 按「组」组织：

| apiVersion | API 组 | 代表对象 |
|---|---|---|
| `v1` | core 组（无组名） | Pod / Service / ConfigMap / Secret |
| `apps/v1` | apps 组 | Deployment / StatefulSet |
| `batch/v1` | batch 组 | Job / CronJob |
| **`networking.k8s.io/v1`** | networking 组 | **Ingress / IngressClass / NetworkPolicy** |
| `rbac.authorization.k8s.io/v1` | rbac 组 | Role / ClusterRole |
| `crd.projectcalico.org/v1` | 自定义组 | IPPool ← **这才是 CRD** |

分组是为了让不同领域独立演进版本。Ingress 自己就走过 `extensions/v1beta1` → `networking.k8s.io/v1beta1` → **`networking.k8s.io/v1`**（1.19 GA，1.22 删除旧版）。

**内置 vs CRD 的区别**：内置由 apiserver 原生提供，开箱即用；CRD 靠 controller 往 apiserver 注册，**没有 controller 就是一堆没人处理的数据**。Calico 的 IPPool、Gateway API 都是 CRD。

## 六、★ pathType 三取值 + 八路径实测

官方定义（kubernetes.io 中文版原文）：

> **ImplementationSpecific**：匹配方法取决于 IngressClass。具体实现可以将其作为单独的 pathType 处理或者作与 Prefix 或 Exact 类型相同的处理。
> **Exact**：精确匹配 URL 路径，且区分大小写。
> **Prefix**：基于以 `/` 分隔的 URL 路径前缀匹配。匹配区分大小写，并且对路径中各个元素**逐个执行匹配操作**。路径元素指的是由 `/` 分隔符分隔的路径中的标签列表。

英文原文的关键词是 **element-wise**（逐元素）—— 这一个词就决定了 `/foo` 匹配不到 `/foobar`。

### 实测配置

```yaml
rules:
  - host: path.local
    http:
      paths:
        - path: /foo
          pathType: Prefix
          backend: {service: {name: app-a, port: {number: 80}}}
        - path: /bar
          pathType: Exact
          backend: {service: {name: app-b, port: {number: 80}}}
```

### 实测结果（判定依据 = controller 日志里的 upstream 归属）

| 路径 | upstream | 结果 |
|---|---|---|
| `/foo` | `default-app-a-80` | ✅ 命中 |
| `/foo/` | `default-app-a-80` | ✅ 命中（带尾斜杠也算） |
| `/foo/abc` | `default-app-a-80` | ✅ 命中（子路径命中） |
| **`/foobar`** | `upstream-default-backend` | ⛔ **不命中 ★ 分水岭** |
| `/bar` | `default-app-b-80` | ✅ 精确命中 |
| `/bar/` | `upstream-default-backend` | ⛔ Exact 不容忍任何差异 |
| `/barx` | `upstream-default-backend` | ⛔ 不命中 |
| `/other` | `upstream-default-backend` | ⛔ 无规则 |

### 三条规律

**① `Prefix` 是按「路径段」匹配，不是字符串前缀**

`/foo` 匹配 `/foo`、`/foo/`、`/foo/abc`，但**匹配不到 `/foobar`** —— `foo` 和 `foobar` 是两个不同的路径元素。

**这是 nginx 原生行为被修正的地方**：nginx 原生的 `location /foo` 是纯字符串前缀，`/foobar` 会被匹配上；K8s 规范要求按段匹配，ingress-nginx 就做了修正（渲染成带段落边界的规则）。

> ⚠️ **你的 nginx 直觉在这个点上会骗你。**

**② `Exact` 是完全相等，一个字符都不能差** —— `/bar` 命中，`/bar/` 就不行。

**③ 大小写敏感** —— 官方三个 pathType 定义里都写了"区分大小写"。

### ImplementationSpecific

由 IngressClass 决定怎么处理。ingress-nginx 下它等价于 Prefix，但**因为依赖实现，可移植性差** —— 考试里优先用 `Prefix` / `Exact`。

## 七、★ 404 的三层来源（重要排障认知）

**同样看到 404，来源可能完全不同，排查方向也不同：**

| 404 来源 | 特征 | 排查方向 |
|---|---|---|
| ① **默认后端** | 日志里 upstream = `upstream-default-backend` → `127.0.0.1:8181` | Host / path 没匹配上任何规则 |
| ② **后端应用自己** | 日志里有真实 upstream（如 `default-app-a-80`） | 路由是通的，问题在应用 |
| ③ **后端 nginx 找不到文件** | 同 ②，且响应里有后端 nginx 的版本号 | 应用配置 / 文件路径问题 |

**判定方法：看 controller 日志的中括号内容**

```
GET /foo    404  [default-app-a-80] [] 10.244.58.231:80     ← 有 upstream，路由正常
GET /foobar 404  [upstream-default-backend] [] 127.0.0.1:8181  ← 无 upstream，路由未命中
```

**有 upstream = 路由是通的。** 这条能省掉大量排障时间。

### 默认后端是什么

不是独立 Pod —— **就是 controller 自己容器里 nginx 的另一个端口**：

```nginx
server {
    listen 8181 default_server reuseport backlog=4096;
    set $proxy_upstream_name "internal";
    access_log off;
    location / {
        return 404;                    # ★ 404 的实际产生地
    }
}
```

配合 80/443 上的兜底 server：

```nginx
server {
    server_name "_" ;                          # 下划线 = 匹配任意域名
    listen 80 default_server  reuseport backlog=4096 ;
    listen 443 default_server reuseport backlog=4096 ssl ;
    ssl_certificate_by_lua_file /etc/nginx/lua/nginx/ngx_conf_certificate.lua;
```

**完整路径**：Host 不匹配 → 落到 80 的 `default_server` → 转发给内部默认后端 → nginx 自己在 8181 上的 server → `return 404`。

**所以那个 404 是 nginx「自己发给自己」的**，后端 Pod 从头到尾没被碰过。

## 八、★ TLS 与 Fake Certificate

### 建 TLS 的正确姿势

```bash
# 1. 自签证书（CN 和 SAN 都要设）
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /tmp/tls.key -out /tmp/tls.crt \
  -subj "/CN=tls.local/O=cka" \
  -addext "subjectAltName=DNS:tls.local"

# 2. 存成 Secret —— 类型必须指定
kubectl create secret tls tls-local --cert=/tmp/tls.crt --key=/tmp/tls.key

# 3. Ingress 加 tls 段
#    spec.tls[].hosts + spec.tls[].secretName

# 4. 验证
curl -sk https://192.168.10.13:31233/ -H "Host: tls.local" -o /dev/null -w 'HTTPS %{http_code}\n'
echo | openssl s_client -connect 192.168.10.13:31233 -servername tls.local 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates -ext subjectAltName
```

**Secret 类型必须是 `kubernetes.io/tls`**（包含 `tls.crt` + `tls.key` 两个 key）。不指定会建出 `Opaque` 类型。

### 证书四要素

| 字段 | 含义 |
|---|---|
| **subject** | 证书所有者（我们的是 `CN=tls.local, O=cka`） |
| **issuer** | 签发者。**自签证书 = subject 完全相同** |
| **dates** | 有效期 |
| **SAN** | Subject Alternative Name，`DNS:tls.local` |

> **现代 TLS 客户端只看 SAN，不看 CN**（CN 自 RFC 6125 起废弃）。只设 CN 不设 SAN，客户端会报证书不匹配 —— 自签证书最常见的坑。

**SNI（Server Name Indication）**：TLS 握手发生在 HTTP 请求之前，服务器必须先拿出证书才能完成握手，但那时还不知道客户端要访问哪个域名。SNI 就是客户端在**握手阶段**就把域名告诉服务器。

> **换个说法：SNI 是 TLS 层的 Host 头。** 443 端口能承载多个域名的证书，全靠它。（`openssl s_client -servername` 就是手动指定 SNI。）

### ★ 核心认知：TLS 层与 HTTP 层是两个独立步骤

```
① TLS 握手阶段  —— 依据【SNI（TLS 层的域名）】选证书 → 握手完成
        ↓
② HTTP 请求阶段 —— 依据【Host 头（HTTP 层的域名）】选后端 → 转发
```

**两套完全独立的判断**，依据不同、发生时间不同。

### 实测：删掉 tls 段，HTTPS 还能通吗？

```bash
kubectl get ingress tls-ing -o jsonpath='{.spec.tls}'   # 空 → tls 段已删
curl -sk https://192.168.10.13:31233/ -H "Host: tls.local" -o /dev/null -w 'HTTPS %{http_code}\n'
# → HTTPS 200        ★ 仍然能通！
```

证书变成：

```
subject=O=Acme Co, CN=Kubernetes Ingress Controller Fake Certificate
issuer =O=Acme Co, CN=Kubernetes Ingress Controller Fake Certificate
```

**它是 controller 内置的兜底证书**，由那行 Lua 在每次握手时动态决定：

```
握手时客户端通过 SNI 告知域名
    ├─ 域名在某个 Ingress 的 tls.hosts 里 → 用对应 Secret 的证书
    └─ 不在                              → 用内置的 Fake Certificate
```

**为什么这样设计**：如果没有 Fake Certificate，任何一个没配证书的域名访问 443 都会握手失败，客户端只能看到"连接被重置"。有了它，至少能拿到一个可读的 HTTP 响应（200 / 404）。

**这也正是「多个 Ingress 共用同一个 443」能成立的基础**：握手时用 SNI 挑证书，握手后用 Host 挑后端，互不干扰。

## 九、一个请求的完整链路（含 LB）

```
① 客户端
   curl -H "Host: demo.local" http://1.2.3.4/
   DNS: demo.local → 1.2.3.4（LB 的 VIP）

② LoadBalancer —— 四层，只认 IP:port
   1.2.3.4:80 → 转发到 <NodeIP>:<NodePort>
   ★ 不解析 HTTP，Host 头原封不动往后传

③ 节点上的 NodePort 规则（kube-proxy）
   <NodeIP>:31xxx → DNAT → controller Pod IP:80
   ★ 这里用的是 Service 机制，对象是 controller 自己的 Service

④ Ingress Controller Pod（nginx）
   读 Host: demo.local + path: /
   跟 Ingress 翻译出来的 nginx.conf 比对
   ★ 整条链路上唯一看 HTTP 内容、做七层决策的地方

⑤ nginx 按匹配到的规则转发到 upstream
   ★ 实测：upstream 指向的是【Pod IP】，不是 ClusterIP

⑥ 后端 Pod 处理请求
⑦ 回程：conntrack 保证原路返回，不需要反向规则
```

### 三个容易搞混的点

**① 这条链路上有【两个】Service**

| 层 | 谁的 Service | 干什么 |
|---|---|---|
| 第 ③ 跳 | Ingress Controller 自己的 Service | 把外部流量送进 controller Pod |
| 后端 | 你应用的 Service | 概念上 Ingress 的后端声明 |

**② 只有一处碰应用层** —— LB / NodePort / ClusterIP 全是四层，只有 controller 看 HTTP。

**③ ClusterIP 是个「不存在的 IP」** —— 没绑在任何网卡上，ping 都不通，纯粹是 iptables 规则的匹配条件。

### 裸金属的四条落地路线

| 方案 | 客户端访问形式 | 优雅度 |
|---|---|---|
| **NodePort** | `http://<NodeIP>:30282` | ⭐ 端口非标，要记端口 |
| **hostNetwork** | `http://<NodeIP>`（80） | ⭐⭐ 省掉 NodePort，但端口被独占 |
| **MetalLB**（裸金属标准答案） | `https://<VIP>` | ⭐⭐⭐⭐ 分配虚拟 IP，标准端口 |
| **云上 LoadBalancer** | `https://1.2.3.4` | ⭐⭐⭐⭐⭐ 最省事 |

**IP 稳定性是递增的**：NodePort 端口（Service 不删就不变，删了重建会重分配）< LoadBalancer VIP（Service 删除后回收）< **域名（唯一真正稳定的入口）**。

**正确姿势：客户端永远只访问域名，IP 只是 DNS 记录里的一个值。**

> 直接拿 IP 访问一定失败 —— 请求头里的 Host 是那个 IP，匹配不上 `demo.local`。除非 Ingress 规则里不写 `host`。

**银行/企业内部常见形态是两层反代**：

```
外部客户端 → F5 / 外部 nginx（第一层，配证书、做第一道安全）
              ↓ proxy_pass
         K8s 的 NodePort 或 LoadBalancer（第二层，按域名/路径分流）
```

第一层管"怎么进来"，第二层（Ingress）管"进来后去哪"。这种架构下 NodePort 的不优雅被第一层遮住了。

## 十、踩坑实录

### 1. `registry.k8s.io` 直连不通，重定向后被拒

必须用 `m.daocloud.io/registry.k8s.io/...`。注意域名是 **`m.daocloud.io`**（不带 `docker.` 前缀），且路径里要保留 `registry.k8s.io`。

### 2. certgen Job 配了 `ttlSecondsAfterFinished: 0`，「只有 1 个 Pod」不是装漏

manifest 里有两个 Job（admission-create / admission-patch），跑完**立即自删**。所以 `get pods` 只看到 controller，属正常。

### 3. `ingress-nginx-controller-admission` 不是流量入口

ClusterIP 443 那个 Service 是给 **apiserver 回调**用的 webhook，校验 Ingress 对象合法性。看到 443 就以为是入口，会白折腾。

### 4. 不带 Host 头访问 → 404

请求头里的 Host 是 IP，匹配不上任何 `server_name` → 落到 default backend。**「必须用域名访问」不是建议，是硬性前提。**

### 5. ★ 后端应用的 404 与路由未命中的 404 必须区分

实测中 `curl /foo` 返回 404，**不是路由没配好** —— 是后端容器里的 nginx 在 root 目录下找不到 `foo` 这个**文件**（`index` 指令只在请求目录时生效）。

判定方法：**看 controller 日志的 upstream 归属**。有 `[default-app-a-80]` 说明路由正常，`[upstream-default-backend]` 才是真的没匹配。

### 6. ConfigMap 的 key 不允许包含 `/`

```bash
kubectl create configmap x --from-literal=foo/index.html='...'
# error: "foo/index.html" is not a valid key name for a ConfigMap:
# a valid config key must consist of alphanumeric characters, '-', '_' or '.'
```

想造目录结构要用 **subPath** 挂载，或者干脆换个后端设计。

### 7. 验证 Ingress 路由不要依赖「后端返回什么内容」

**验证路由归属，看 controller 日志的 upstream 就够了。** 依赖后端内容会被应用自身的行为干扰（比如后端 nginx 的路径匹配、404 页面），把结论带偏。

### 8. 删掉 `tls:` 段，HTTPS 依然能通

会 fallback 到 controller 内置的 **Fake Certificate**（`CN=Kubernetes Ingress Controller Fake Certificate`），走 `ssl_certificate_by_lua_file` 动态选择。

### 9. 自签证书必须设 SAN

只设 CN 不设 SAN，现代客户端一律报证书不匹配。`-addext "subjectAltName=DNS:xxx"` 不能省。

## 十一、复习自测题（答案）

1. **Ingress 和 Ingress Controller 是什么关系？**
   - Ingress 是**规则**（存 etcd，无网络实体）；Controller 是**执行者**（一个 Pod，里面跑 nginx）
   - 没有 controller，Ingress 资源就是一份没人读的文档

2. **Ingress 和 Service 的分工是什么？**
   - Service（L4，内核 DNAT）解决"怎么找到一组 Pod"
   - Ingress（L7，用户态 nginx）解决"什么样的 HTTP 请求该去哪组 Pod"
   - **Ingress 的后端只能写 Service，不能写 Pod**

3. **NodePort 30282 由谁决定？新建 Ingress 会变吗？**
   - 由 **controller 的 Service** 决定
   - 新建 Ingress / IngressClass 都**不会**变；只有改这个 Service 才会变
   - **所有 Ingress 规则共用同一个入口端口**

4. **不带 Host 头访问 Ingress 入口会怎样？为什么？**
   - 返回 404
   - 请求头 Host 是 IP，匹配不上任何 `server_name` → 落到 default backend（127.0.0.1:8181）

5. **`pathType: Prefix` 下，`/foo` 能匹配 `/foobar` 吗？**
   - **不能**
   - Prefix 是**按 `/` 分隔的路径元素逐个匹配**（element-wise），`foo` 和 `foobar` 是两个不同的元素
   - nginx 原生的 `location /foo` 是纯字符串前缀会匹配上，**ingress-nginx 做了修正去符合 K8s 语义**

6. **`/foo` 返回 404，怎么判断是路由问题还是应用问题？**
   - 看 controller 日志的中括号：`[default-app-a-80]` = 路由正常（应用问题）；`[upstream-default-backend]` = 路由未命中

7. **`Prefix` 和 `Exact` 最主要的行为差异？**
   - Prefix：`/foo` 能匹配 `/foo/`、`/foo/abc`，不匹配 `/foobar`
   - Exact：只有完全相等才匹配，`/bar/` 就不行
   - 两者都**区分大小写**

8. **controller 是怎么拿到后端 Pod IP 的？为什么不用 CoreDNS？**
   - 它 **watch `EndpointSlice`**，把健康 Pod IP 直接渲染进 upstream
   - 不用 DNS 的原因：① nginx 默认启动时只解析一次域名 ② DNS 解析出来是 ClusterIP，还要多走一跳 DNAT ③ DNS 表达不了"哪些 Pod 是 ready 的"
   - **改 Pod IP 不需要 reload**（走 Lua 共享内存动态更新），改路由规则才 reload

9. **TLS 层和 HTTP 层分别依据什么做判断？**
   - TLS 握手阶段：依据 **SNI** 选证书
   - HTTP 请求阶段：依据 **Host 头**选后端
   - 两者独立，这是"多个 Ingress 共用 443"能成立的基础

10. **删掉 Ingress 的 `tls:` 段后，`https://` 访问会失败吗？**
    - **不会**。TLS 握手会 fallback 到 controller 内置的 **Fake Certificate**，连接照常建立，只是证书不受信任
    - 请求仍按 Host 头正常路由到后端

11. **建 TLS 用的 Secret 类型是什么？不指定会怎样？**
    - `kubernetes.io/tls`（含 `tls.crt` + `tls.key`）
    - 不指定会建出 `Opaque` 类型

12. **Ingress 是内置 API 还是 CRD？apiVersion 为什么不是 v1？**
    - **内置 API**（`kubectl get crd | grep ingress` 为空可证）
    - K8s API 按「组」组织，`networking.k8s.io/v1` 是 networking 组，与 core 组的 `v1` 平级。分组是为了让不同领域独立演进版本

## 速记卡片

```bash
# ── 家底
kubectl -n ingress-nginx get all,cm,secret,sa
kubectl get ingressclass                      # NAME=nginx, CONTROLLER=k8s.io/ingress-nginx
kubectl -n ingress-nginx get svc -o wide      # NodePort 80:30282  443:31233

# ── 访问（必须带 Host 头；nodePort 从 Service 取）
NP=$(kubectl -n ingress-nginx get svc ingress-nginx-controller \
     -o jsonpath='{.spec.ports[?(@.name=="http")].nodePort}')
curl -s -H "Host: demo.local" http://<任意NodeIP>:$NP

# ── 排障三件套
kubectl describe ingress <name>               # Rules 段：Host/Path/Backends(显示 Pod IP)
kubectl -n ingress-nginx logs deploy/ingress-nginx-controller --tail=20
#     ↑ 中括号里看 upstream 归属：
#       [default-app-a-80]           = 路由命中，404 是应用发的
#       [upstream-default-backend]   = 路由未命中 → 127.0.0.1:8181 的 404

# ── 看生成的 nginx.conf
CP=$(kubectl -n ingress-nginx get pod -l app.kubernetes.io/component=controller \
     -o jsonpath='{.items[0].metadata.name}')
kubectl -n ingress-nginx exec $CP -- grep -n 'server_name' /etc/nginx/nginx.conf
kubectl -n ingress-nginx exec $CP -- grep -n -A8 'default_server' /etc/nginx/nginx.conf

# ── 看证书
echo | openssl s_client -connect <NodeIP>:31233 -servername <域名> 2>/dev/null \
  | openssl x509 -noout -subject -issuer

# ── 建 TLS
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /tmp/tls.key -out /tmp/tls.crt \
  -subj "/CN=tls.local/O=cka" -addext "subjectAltName=DNS:tls.local"
kubectl create secret tls tls-local --cert=/tmp/tls.crt --key=/tmp/tls.key

# ── 镜像源（本集群）
registry.k8s.io/...  →  m.daocloud.io/registry.k8s.io/...

# ── pathType 速记
Prefix  /foo → /foo ✅  /foo/ ✅  /foo/abc ✅  /foobar ⛔   (逐元素匹配)
Exact   /bar → /bar ✅  /bar/ ⛔  /barx ⛔                 (完全相等)
两者都区分大小写 | 不写 pathType 通不过校验

# ── 关键认知
Ingress = 规则（无实体） / Controller = 执行者 / Service = 入口（30282 属于它）
只有一处看 HTTP：controller 的 nginx
TLS 层看 SNI 选证书，HTTP 层看 Host 选后端 —— 两件独立的事
```
