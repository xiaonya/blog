---
title: "Namespace 与资源隔离"
sidebar_position: 4
---

# Namespace 与资源隔离

> 日期：2026-08-25
> 对应 CKA 考纲：Core Concepts - Namespace

---

## 一、今日目标

- [x] 创建/删除 Namespace
- [x] 验证 `-n` 和 `--all-namespaces` 的使用
- [x] 验证级联删除
- [x] 理解 Namespace vs Label 的区别
- [x] 验证 Service selector 跨不了 namespace

---

## 二、操作记录

### 1. 创建 Namespace 和 Pod

```bash
# 创建 dev 和 prod 两个 namespace（隐含操作）
kubectl create namespace dev
kubectl create namespace prod

# 在各自 namespace 下建 Pod
kubectl run web-dev --image=nginx:1.27 -n dev
kubectl run web-prod --image=nginx:1.27 -n prod

# 验证
kubectl get pods -n dev
kubectl get pods -n prod
```

### 2. 级联删除验证

```bash
kubectl run api-dev --image=busybox:1.36 -n dev -- sleep 3600
kubectl delete namespace dev    # api-dev 和 web-dev 一起消失
```

### 3. --all-namespaces + label selector

```bash
kubectl label pod web-prod -n prod env=prod
kubectl get pods -l env=prod --all-namespaces
# 输出：default/web3（Day2 遗留）和 prod/web-prod 都被筛出来
```

### 4. Service 跨 namespace 验证

```bash
# 不加 -n，默认走 default → NotFound
kubectl expose pod web-prod --port=80 --name=svc-test2 -n default
# Error: pods "web-prod" not found

# 加 -n prod → 成功
kubectl expose pod web-prod --port=80 --name=svc-test -n prod
kubectl get svc svc-test -n prod
kubectl get endpoints svc-test -n prod
# ENDPOINTS: 10.244.85.206:80 → Service 找到了后端 Pod
```

### 5. 清理

```bash
kubectl delete namespace prod    # 级联删除 web-prod + svc-test
```

---

## 三、关键概念（用自己的话填）

### Namespace 是什么

_（留白：用一句话说清 Namespace 的作用）_

### 不受 namespace 约束的资源

_（留白：至少列 5 个，并说明为什么它们不在 namespace 里）_

### Service 为什么只能选同 namespace 的 Pod

_（留白：从 API server 的角度解释）_

---

## 四、Namespace vs Label 对比表

| 维度 | Namespace | Label |
|---|---|---|
| 隔离性 | ? | ? |
| 筛选方式 | ? | ? |
| 跨域性 | ? | ? |
| 删除行为 | ? | ? |

> 填完和下面的参考答案对照

<details>
<summary>参考答案</summary>

| 维度 | Namespace | Label |
|---|---|---|
| 隔离性 | 物理隔离（墙），资源互相看不见 | 无隔离，只有分类 |
| 筛选方式 | `-n <ns>` 精确指定 | `-l key=value` 条件过滤 |
| 跨域性 | 不嵌套，不跨域 | 跨 namespace 可有相同 label |
| 删除行为 | 级联删除所有子资源 | 删 label 不影响 Pod |

一句话：**Namespace 是墙，Label 是贴纸**。

</details>

---

## 五、排障套路

### "找不到资源"排查三步

1. `kubectl get <resource> --all-namespaces` — 全局搜，确认资源存在
2. 看输出里的 NAMESPACE 列 — 确认在哪个 namespace
3. 加 `-n <正确的namespace>` 重新操作

### Service 没有 endpoints 排查

1. `kubectl get endpoints <svc> -n <ns>` — 确认是否真的为空
2. `kubectl describe svc <svc> -n <ns>` — 看 Selector 字段
3. `kubectl get pods -l <selector> -n <ns>` — 用 selector 查 Pod
4. 如果 Pod 在别的 namespace — Service 跨不了 namespace，需要把 Service 建到 Pod 所在的 namespace

---

## 六、易错点 / 踩坑记录

### 1. 忘了 -n（今天踩的）
```
kubectl expose pod web-prod --port=80 --name=svc-test2 -n default
→ Error: pods "web-prod" not found
```
原因：web-prod 在 prod namespace，default 里当然找不到。
教训：**操作前先想资源在哪个 namespace，不加 -n 就是 default。**

### 2. Endpoints API 废弃警告
```
Warning: v1 Endpoints is deprecated in v1.33+; use discovery.k8s.io/v1 EndpointSlice
```
不影响使用，EndpointSlice 是新 API，`kubectl get endpoints` 仍可用。

---

## 七、参考文档

- [Namespaces | Kubernetes](https://kubernetes.io/zh-cn/docs/concepts/overview/working-with-objects/namespaces/)
- [Labels and Selectors | Kubernetes](https://kubernetes.io/zh-cn/docs/concepts/overview/working-with-objects/labels/)
- `kubectl explain namespace`
- `kubectl explain pod.metadata.namespace`
