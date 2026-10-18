---
title: "Deployment 与滚动更新、回滚"
sidebar_position: 1
---

# Deployment 与滚动更新、回滚

## 核心结构（三层关系）

```
Deployment (spec.template)
   ↓ 创建/管理
ReplicaSet (selector.matchLabels)
   ↓ 创建/管理
Pods (spec.template.spec.containers)
```

- **Deployment**：发布工程师，管"应用名 + 镜像版本"
- **ReplicaSet**：副本管理员，管"几个 Pod、标签过滤"
- **Pod**：实际跑容器

**回滚的本质**：不是改 RS，而是 Deployment 切换自己的 selector / template 指向历史 revision 里的旧 RS，让旧 RS 重新扩容到 replicas。

## 一、命令式生成 YAML（CKA 主力姿势）

```bash
kubectl create deployment nginx --image=nginx:1.25 --replicas=3 \
  --dry-run=client -o yaml > deploy.yaml
```

看到的三层结构：
- `spec.replicas: 3`（Deployment 字段）
- `spec.selector.matchLabels.app: nginx`（Deployment 用它选 RS）
- `spec.template`（Pod 模板，RS 用它创建 Pod）

**注意**：`spec.selector` 一旦 apply 就**不可改**，考试时让它和 template 的 labels 保持一致是默认行为。

## 二、apply + 观察

```bash
kubectl apply -f deploy.yaml
kubectl get deploy,rs,pod -l app=nginx
```

- RS 名字 = `nginx-<hash>`（hash 是 template 的 hash）
- Pod 名字 = `nginx-<rs-hash>-<pod-hash>`
- `--label app=nginx` 一次过滤三类资源，`get deploy,rs,pod` 是表格视图常用前缀

## 三、触发滚动更新（三种姿势等价）

```bash
# A. set image（最高频）
kubectl set image deployment/nginx nginx=nginx:1.26
# 注意：左边是容器名（看 spec.containers[0].name），不是 Deployment 名
# 考试时多容器场景需要：kubectl set image deploy/web web=nginx:1.26 sidecar=envoy:v2

# B. edit
kubectl edit deployment nginx
# 改 spec.template.spec.containers[0].image

# C. 改 yaml 后 apply
sed -i 's/nginx:1.25/nginx:1.27/' deploy.yaml
kubectl apply -f deploy.yaml
```

**底层逻辑相同**：template 变 → 创建新 RS → 旧 RS 收尾。

## 四、监控滚动（四件套）

```bash
# 阻塞式：等滚动完才返回
kubectl rollout status deploy/nginx

# 实时观察 Pod 替换
kubectl get pods -l app=nginx -w

# 卡住第一个查这里
kubectl describe deploy nginx | tail -15

# 看历史版本
kubectl rollout history deploy/nginx
```

## 五、故障演练（必做）

```bash
# 故意触发失败
kubectl set image deploy/nginx nginx=nginx:999

# 观察：Pod 卡 ImagePullBackOff / ErrImagePull
kubectl get pods -l app=nginx

# Deployment 不会自动放弃，默认 progressDeadlineSeconds: 600
kubectl rollout status deploy/nginx   # 阻塞，不会返回
```

**回滚**（默认回上一版）：

```bash
kubectl rollout undo deploy/nginx
kubectl rollout status deploy/nginx
kubectl get pods -l app=nginx -o jsonpath='{.items[0].spec.containers[0].image}'   # 验证镜像
```

**指定版本回滚**（回过多次时）：

```bash
kubectl rollout history deploy/nginx
kubectl rollout undo deploy/nginx --to-revision=2
```

## 六、kubectl apply 的注解漂移警告（重要）

`kubectl apply -f` 创建的对象，kubectl 会保存一份"原始 YAML 快照"到 `metadata.annotations["kubectl.kubernetes.io/last-applied-configuration"]`。

如果之后用 `rollout undo` 回滚了镜像，**注解还是指向旧版本**。下次再 `kubectl apply -f deploy.yaml`，kubectl 会发现注解和 yaml 没差异，**把回滚后的状态覆盖回去**。

**生产实践**：

```bash
# 1. 回滚到目标版本
kubectl rollout undo deploy/nginx --to-revision=1

# 2. 立刻把 yaml 改成当前实际跑的版本，再 apply 一次同步注解
sed -i 's/nginx:1.27/nginx:1.25/' deploy.yaml
kubectl apply -f deploy.yaml
```

也就是 warning 最后那句 `Consider using 'kubectl apply' with your previous configuration file instead` 在提醒的事。

## 七、change-cause 备注（K8s 1.23+）

老版本的 `--record` 标记在 1.23 已经被移除，现在的姿势是手工加注解：

```bash
kubectl annotate deploy/nginx kubernetes.io/change-cause="bump to 1.26 for CVE-2025-XXXX" --overwrite=true
```

加 `--overwrite` 防止重复加时冲突。

## 八、pause/resume 的金丝雀用法

```bash
kubectl set image deploy/web web=nginx:1.27
kubectl rollout pause deploy/web               # 暂停滚动
kubectl exec -it pod/web-xxx -- curl localhost   # 进新 Pod 自检
kubectl rollout resume deploy/web              # 没问题再恢复
```

## 九、autoscale 伏笔（Week 7 完整展开）

```bash
kubectl autoscale deploy/nginx --min=2 --max=10 --cpu-percent=80
kubectl get hpa
```

注意：HPA 需要 metrics-server 才能真正采集 CPU，离线/裸集群装的话会一直显示 `<unknown>`，Week 7 会专门装。

## 十、踩坑实录

### 1. `kubectl set image` 的"容器名"≠ Deployment 名

如果一个 Deployment 里只有一个容器（最常见），容器名经常也是 `nginx`，**和 Deployment 名同名**，容易误以为是 Deployment 名。

多容器场景必须显式列出每个：

```bash
kubectl set image deploy/web web=nginx:1.26 sidecar=envoy:v2
```

### 2. apply 与命令式回滚的注解漂移

见上文第六节，是今天最重要的认知。

### 3. 滚动卡住不会自动放弃

默认 `progressDeadlineSeconds: 600`（10 分钟），超时才标 `failed`，期间 `rollout status` 会无限阻塞。要主动 `rollout undo` 或 `pause` 解除。

## 十一、复习自测题（答案）

1. **Deployment / RS / Pod 关系？回滚什么对象？**
   - Deployment 通过 selector 选 RS，RS 用 selector 选 Pod
   - 回滚 = Deployment 把 selector / template 切回历史 revision 的旧 RS（不是直接改 RS spec）

2. **maxSurge / maxUnavailable 默认？**
   - 都是 25%
   - 两者不能同时为 0（会死锁）

3. **滚动卡住怎么定位？回滚会卡吗？**
   - `kubectl get pods -l app=... -o wide` + `kubectl rollout status`
   - `kubectl describe deploy xxx | tail -15` 看事件
   - 回滚也可能卡（旧镜像也拉不到时）

4. **rollout history 默认看不到 change-cause，怎么加？**
   - `kubectl annotate deploy/xxx kubernetes.io/change-cause="..." --overwrite=true`

5. **pause / resume 场景？**
   - 金丝雀发布：set image → pause → 自检新 Pod → 自检通过 resume
   - 紧急止血：发现新版本异常，pause 卡住正在进行的滚动，留出决策时间

## 速记卡片

```
kubectl rollout
  ├─ status      # 阻塞式观察滚动进度
  ├─ history     # 看 revision 和备注
  ├─ undo        # 默认回上一版，--to-revision=N 指定
  ├─ pause/resume    # 暂停/恢复（金丝雀/紧急止血）
  └─ restart     # 重启（pod template 变的不算，强制重启用这个）
```
