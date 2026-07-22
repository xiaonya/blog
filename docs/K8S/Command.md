# K8S 命令速查
常用的 Kubernetes 命令记录
```bash
# 安装时使用到的命令

# 查看安装的k8s版本
kubectl version
# 查看集群信息
kubectl cluster-info
# 查看组件状态
kubectl get componentstatuses
# 查看集群的详细信息
kubectl cluster-info dump

# 类似于 docker ps，查询资源列表
kubectl get pods nginx-pod
# 类似于 docker inspect，获取资源的详细信息
kubectl describe pods nginx-pod
# 类似于 docker logs，获取容器的日志
kubectl logs pods nginx-pod
# 类似于 docker exec，进入容器的交互式终端
kubectl exec -it nginx-pod -- /bin/bash





```