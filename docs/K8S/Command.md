---
title: 命令速查
sidebar_position: 4
---

# 命令速查

常用的 Kubernetes 命令记录，按使用场景分类。练习和考试时按场景找，比按字母找快。

## 0. 效率配置（每台新机器先做）

```bash
alias k=kubectl
source <(kubectl completion bash)
complete -F __start_kubectl k

# 考试常用组合：先 dry-run 出骨架，改完再 apply
export do="--dry-run=client -o yaml"
export now="--force --grace-period=0"
```

## 1. 集群信息与组件

```bash
# 查看安装的 k8s 版本（客户端 + 服务端）
kubectl version
# 查看集群信息（apiserver 地址、CoreDNS 地址）
kubectl cluster-info
# 查看集群的详细信息（各组件配置快照，排障用）
kubectl cluster-info dump
# 查看组件状态（旧命令，新版本已 deprecated，看 kube-system 里的 Pod 即可）
kubectl get componentstatuses

# 节点视角
kubectl get nodes -o wide
kubectl describe node k8s-node01
kubectl top nodes            # 需要 metrics-server
kubectl api-resources        # 有哪些 kind、缩写、是否 namespaced
kubectl api-versions         # 有哪些 API 组版本
```

## 2. 查询与排障

```bash
# 类似于 docker ps，查询资源列表
kubectl get pods nginx-pod
# 类似于 docker inspect，获取资源的详细信息（Events 段是最有用的排障信息）
kubectl describe pods nginx-pod
# 类似于 docker logs，获取容器的日志
kubectl logs pods nginx-pod
kubectl logs -f deploy/web --tail=100          # 跟日志
kubectl logs nginx-pod --previous              # 上一个已死容器的日志（CrashLoopBackOff 必用）
# 类似于 docker exec，进入容器的交互式终端
kubectl exec -it nginx-pod -- /bin/bash
kubectl exec nginx-pod -- env | grep DB        # 单次执行，不进交互

# 事件与字段提取
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl get events -A --field-selector type=Warning
kubectl get pod web -o yaml
kubectl get pod web -o jsonpath='{.spec.containers[*].image}'
kubectl get pods -o custom-columns=NAME:.metadata.name,NODE:.spec.nodeName,IP:.status.podIP
kubectl get pods --sort-by=.status.startTime
```

## 3. 生成 YAML 骨架（考试核心动作）

```bash
# Pod
kubectl run web --image=nginx:1.27 $do > pod.yaml
# Deployment
kubectl create deployment web --image=nginx:1.27 --replicas=3 $do > deploy.yaml
# Job / CronJob
kubectl create job hello --image=busybox:1.36 -- echo hi $do > job.yaml
kubectl create cronjob backup --image=busybox:1.36 --schedule="*/5 * * * *" -- date $do > cj.yaml
# Service / Ingress
kubectl expose deployment web --port=80 --target-port=8080 --type=NodePort $do > svc.yaml
kubectl create service clusterip web --tcp=80:8080 $do > svc.yaml
kubectl create ingress web --rule="demo.local/=web:80" $do > ing.yaml
# 配置与密钥
kubectl create configmap app-cm --from-literal=LOG_LEVEL=debug --from-file=./app.conf $do > cm.yaml
kubectl create secret generic db-cred --from-literal=password='S3cret!' $do > secret.yaml
kubectl create secret tls tls-local --cert=tls.crt --key=tls.key $do > tls.yaml
# 权限
kubectl create serviceaccount app-sa $do > sa.yaml
kubectl create role pod-reader --verb=get,list,watch --resource=pods $do > role.yaml
kubectl create rolebinding rb --role=pod-reader --serviceaccount=dev:app-sa $do > rb.yaml
```

## 4. 工作负载运维

```bash
kubectl apply -f deploy.yaml
kubectl apply -f .                       # 整个目录
kubectl delete -f deploy.yaml
kubectl edit deploy web                  # 直接改线上对象，改完立即生效

# 滚动更新与回滚
kubectl rollout status deploy/web
kubectl rollout history deploy/web
kubectl rollout undo deploy/web
kubectl rollout undo deploy/web --to-revision=1
kubectl rollout restart deploy/web       # 触发一次重启（改配置后用）
kubectl rollout pause/resume deploy/web  # 金丝雀发布的分段控制

# 改镜像与副本
kubectl set image deploy/web nginx=nginx:1.27
kubectl scale deploy/web --replicas=5
kubectl autoscale deploy/web --min=2 --max=10 --cpu-percent=70
```

## 5. 节点维护与调度

```bash
kubectl cordon k8s-node01                       # 标记不可调度（不影响已运行 Pod）
kubectl drain k8s-node01 --ignore-daemonsets --delete-emptydir-data   # 驱逐 + 封禁
kubectl uncordon k8s-node01                     # 恢复调度

kubectl taint nodes k8s-node01 gpu=true:NoSchedule
kubectl taint nodes k8s-node01 gpu=true:NoSchedule-   # 末尾减号表示删除污点
kubectl label node k8s-node01 disk=ssd
```

## 6. 存储

```bash
kubectl get pv,pvc,sc -A
kubectl describe pvc data-claim          # 卡 Pending 时先看 Events
kubectl patch pvc data-claim -p '{"spec":{"resources":{"requests":{"storage":"2Gi"}}}}'
kubectl get pvc -o custom-columns=NAME:.metadata.name,SC:.spec.storageClassName,STATUS:.status.phase
```

## 7. 网络

```bash
kubectl get svc,endpoints,endpointslices -A
kubectl get ingress -A
kubectl get networkpolicy -A
kubectl port-forward svc/web 8080:80     # 本机临时打通，不走 Ingress
kubectl get pods -o wide                 # 看 Pod IP，验证同名 Service 后面到底挂了谁
```

## 8. 安装与运行时（kubeadm / containerd）

```bash
kubeadm init --pod-network-cidr=10.244.0.0/16
kubeadm token create --print-join-command   # 重新生成 join 命令（token 24h 过期）
kubeadm reset                              # 清掉本机 kubeadm 痕迹
kubeadm certs check-expiration             # 证书到期排查第一招
kubeadm certs renew all

crictl ps
crictl images
crictl logs <container-id>
crictl pull nginx:1.27
systemctl status containerd
journalctl -u kubelet -f                    # 控制平面静态 Pod 起不来时看这里
```

## 9. 权限排查

```bash
kubectl auth can-i get secrets --as=system:serviceaccount:dev:app-sa
kubectl auth can-i --list --as=system:serviceaccount:dev:app-sa
kubectl create token app-sa -n dev           # 1.24+ 手工签一个 SA token
kubectl config set-context --current --namespace=dev
kubectl config get-contexts
```

## 10. 考试习惯（比命令本身更重要）

- 题干先看 **namespace**，切好上下文再动手：`kubectl config set-context --current --namespace=X`
- 能 `dry-run` 就不要从零手写 YAML，改骨架比写全文快且不出缩进错
- 任何对象状态不对，第一个动作是 `kubectl describe` 看 Events，第二个是 `logs --previous`
- `apply` 返回成功 ≠ 应用上线，发布确认要用 `kubectl rollout status`
- 命令记不全就查官方文档，考试环境允许访问 `kubernetes.io/docs`
