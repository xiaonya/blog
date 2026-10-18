---
title: "ConfigMap：创建、消费与热更新"
sidebar_position: 1
---

# ConfigMap：创建、消费与热更新

> Week 3 / 配置与密钥 · 实操日期 2026-09-01 · 集群：VMware K8S-Master

## 今日目标

1. 掌握 ConfigMap 三种创建方式及 `data` 结构差异
2. 掌握 Pod 中三种消费方式：envFrom / valueFrom / volume
3. 验证热更新行为差异（env vs volume）

## 一、三种创建方式对比（核心记忆点）

| 方式 | data 里的 key | data 里的 value | 适用场景 |
|------|--------------|----------------|---------|
| `--from-literal=K=V` | 变量名 | 明文字符串 | 一两个零散变量 |
| `--from-file=文件` | **文件名** | **整个文件内容**（YAML 中呈 `\|` 多行块） | 完整配置文件原样挂载 |
| `--from-env-file=文件` | 每行 K=V 拆成**独立 key** | 各自的 V | KEY=VALUE 清单，按 key 逐个引用 |

实操命令：

```bash
kubectl create configmap cm-literal --from-literal=APP_MODE=debug --from-literal=MAX_CONN=100
kubectl create configmap cm-file    --from-file=app.properties
kubectl create configmap cm-envfile --from-env-file=app.properties
```

同一个 app.properties，`cm-file` 只有一个 key（`app.properties`），`cm-envfile` 有三个 key（`db.host` / `db.port` / `db.user`）。

**查值用 `get -o yaml`，`describe` 只显示 key 数量看不到值。**

## 二、YAML 块标量速记

`data` 里 value 为多行时的写法：

| 符号 | 行为 |
|------|------|
| `\|` | 保留换行，末尾留一个换行（默认） |
| `\|-` | 保留换行，去掉末尾换行 |
| `>` | 折叠换行（换行变空格，段落文本用） |

ConfigMap 的 value 在 YAML 中始终是**明文**；base64 是 Secret 的特征，两者别混。

## 三、Pod 消费（考试重点写法）

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-env
spec:
  containers:
  - image: busybox:latest
    name: busybox
    command: ["sleep", "3600"]
    envFrom:                          # 整个 cm 的所有 key → 环境变量
    - configMapRef:
        name: cm-literal
    env:
    - name: DB_HOST                   # 变量名可自定义
      valueFrom:
        configMapKeyRef:              # 从 cm 里挑一个 key（必考）
          name: cm-envfile
          key: db.host
```

验证：`kubectl exec pod-env -- env | grep -Ei 'app_mode|max_conn|db_host'`

Volume 挂载（每个 key → 一个文件）：

```yaml
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
      readOnly: true
  volumes:
  - name: config-volume
    configMap:
      name: cm-file
```

验证：`kubectl exec pod-vol -- ls -l /etc/config`（可见 `..data -> ..2026_..` 符号链接，挂载只读）

## 四、热更新结论（对比实验验证）

| 消费方式 | ConfigMap 修改后 |
|---------|----------------|
| env（envFrom / valueFrom） | **不更新**，容器内 env 是启动时一次性注入，需重启 Pod |
| volume 挂载 | **约 1 分钟后自动刷新**（kubelet 周期同步 + 换符号链接指向），Pod 不重启 |

## 五、踩坑实录（比成功更有价值）

1. **YAML 缩进断裂** → `error: did not find expected '-' indicator`
   `containers` 列表项 `- image: xxx` 后续字段（name/command/envFrom…）必须与 `image` 对齐（比 `-` 多 2 格）。缩进存疑时用 `cat -A 文件名` 检查。
2. **引用不存在的 key** → Pod 卡在 `CreateContainerConfigError`
   `configMapKeyRef.key` 必须是 cm `data` 里真实存在的 key。原因定位：`kubectl describe pod xxx | grep -A5 Events`（`get` 只看症状，`describe` 看原因）。
3. **CRLF 行尾**（Windows 编辑传到 Linux）→ `cat -A` 里行尾显示 `^M$`，遇到诡异解析错误先 `dos2unix 文件名`。

## 六、其他要点

- ConfigMap 不可跨 namespace 引用；总大小限制 1MB
- `kubectl create cm xxx --dry-run=client -o yaml` 先预览再落库（好习惯）
- key 不存在但不希望阻塞启动：`configMapKeyRef` 加 `optional: true`
- subPath 方式挂载的**单个文件不热更新**（与整目录挂载不同）

## 七、文档坐标（kubernetes.io/docs）

- Concepts → Configuration → **Configure a Pod to Use a ConfigMap**（考试 YAML 直接抄写来源）
- Tasks → Configure Pods and Containers → **Distribute Configuration Data**

---

## 附：简易复习题（先答再翻答案）

1. `--from-file` 和 `--from-env-file` 对同一份 properties 文件生成的 ConfigMap，`data` 有什么区别？
2. Pod 引用的 cm key 不存在，Pod 状态是什么？哪条命令能最快看到原因？
3. ConfigMap 更新后，env 注入和 volume 挂载的容器分别发生什么？
4. `kubectl describe cm` 能看到 data 里的值吗？查值用哪条命令？
5. YAML 里 `\|` 和 `>` 有什么区别？哪种类型资源里会出现 base64 值？

<details>
<summary>答案</summary>

1. from-file：key=文件名，value=整个文件；from-env-file：每行 K=V 拆成独立 key。
2. CreateContainerConfigError；`kubectl describe pod xxx` 看 Events。
3. env 不更新（需重启 Pod）；volume 约 1 分钟后自动刷新，Pod 不重启。
4. 看不到（只显示 key 数量）；`kubectl get cm xxx -o yaml`。
5. `|` 保留换行，`>` 折叠换行为空格；Secret 的 data 值是 base64。
</details>
