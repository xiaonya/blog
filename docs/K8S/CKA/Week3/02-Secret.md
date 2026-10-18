---
title: "Secret：类型、消费与安全边界"
sidebar_position: 2
---

# Secret：类型、消费与安全边界

> Week 3 · 配置与密钥 · 2026-09-02

## 一、Secret 与 ConfigMap 的本质区别

|            | ConfigMap | Secret                     |
| ---------- | --------- | -------------------------- |
| `data` 值形态 | 明文        | base64 编码                  |
| 用途         | 非敏感配置     | 密码 / token / 证书            |
| 安全属性       | 无         | base64 是编码**不是加密**，防君子不防小人 |

**核心结论**：base64 能一条命令还原，所以 Secret 的真正防线是 RBAC（谁能 `get secret`），不是编码本身。

还原明文：

```bash
kubectl get secret db-cred -o jsonpath='{.data.password}' | base64 -d; echo
```

---

## 二、Secret 类型（概述页表格，CKA 考前两种）

| 类型                                  | 创建命令                                    | 用途        |
| ----------------------------------- | --------------------------------------- | --------- |
| Opaque                              | `kubectl create secret generic`         | 通用键值，默认类型 |
| kubernetes.io/tls                   | `kubectl create secret tls`             | TLS 证书    |
| kubernetes.io/docker-registry       | `kubectl create secret docker-registry` | 镜像仓库凭据    |
| kubernetes.io/service-account-token | 自动生成                                    | SA 令牌     |

---

## 三、创建方式

### 1. generic（命令行，考试最快）

```bash
kubectl create secret generic db-cred \
  --from-literal=username=admin \
  --from-literal=password='S3cret!'
```

### 2. stringData 免编码写法

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-cred2
type: Opaque
stringData:
  username: admin
  password: "S3cret!"
```

- `stringData` 是创建时的**明文输入**，API Server 写入时自动转 base64 存进 `data`
- 转码后 `stringData` 字段**不持久化**（`get -o jsonpath='{.stringData.password}'` 返回空）
- ⚠️ **坑**：明文会残留进 `metadata.annotations["kubectl.kubernetes.io/last-applied-configuration"]` 注解里。严格保密场景应自己 base64 后直接写 `data`，避开 `stringData`。

---

## 四、消费方式

### env 注入（结构与 configMapKeyRef 同构）

```yaml
    env:
    - name: DB_USER
      valueFrom:
        secretKeyRef:
          name: db-cred
          key: username
```

注入的是**解码后的明文**。

### volume 挂载

```yaml
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secret
      readOnly: true
  volumes:
  - name: secret-volume
    secret:
      secretName: db-cred
      defaultMode: 0400    # Secret 默认 0400，ConfigMap 默认 0644
```

- 每个 key 一个文件，内容已解码为明文
- `subPath` 挂载单个 key：`subPath: password`（注意 subPath 不热更新）

---

## 五、tls 类型

```bash
openssl req -x509 -newkey rsa:2048 -nodes \
  -keyout tls.key -out tls.crt -days 365 \
  -subj "/CN=example.com"

kubectl create secret tls my-tls --cert=tls.crt --key=tls.key
```

- `type: kubernetes.io/tls`
- `data` 固定只有两个 key：**`tls.crt`** 和 **`tls.key`**
- 考题出现 HTTPS / 证书 / ingress → 条件反射用 `create secret tls`

---

## 六、踩坑实录

1. **`kubectl run` 遗留 args 导致 CrashLoop**：`kubectl run pod --image=busybox` 自动带出 `args: [<pod名>]`，手写 `command: ["sleep","3600"]` 没删 args，最终执行 `sleep 3600 pod名` → sleep 报 invalid number 退出 → `Error` + `BackOff` 循环重启。
2. **stringData 明文泄漏进注解**（见三·2）。
3. **echo 不带 -n 的换行坑**：`echo 'admin' | base64` 会多编一个换行（尾部 `Cg==`），Pod 拿到 `admin\n` 登录必挂。编码用 `echo -n`，可疑值用 `cat -A` 看尾随 `$`。

**两种“Pod 起不来”分诊**：

- 卡 `ContainerCreating` / `CreateContainerConfigError` → 配置解析失败（引用不存在的 Secret/ConfigMap 或 key）
- `Error` + `BackOff` → 容器启动后进程自身退出，查 `kubectl logs --previous`

---

## 七、复习题（答案）

1. Secret `data` 与 ConfigMap `data` 本质区别？—— Secret 是 base64 编码，ConfigMap 是明文；base64 是编码不是加密。
2. `stringData` 与 `data` 的关系？—— stringData 是创建时明文输入，API Server 写入时转 base64 存 data；stringData 不持久化。
3. env / volume 拿到的值形态？volume 默认权限？—— 都是明文；volume 默认 0400。
4. 创建 TLS Secret 的命令与固定 key？—— `kubectl create secret tls my-tls --cert=tls.crt --key=tls.key`；固定 key 为 `tls.crt`、`tls.key`。
