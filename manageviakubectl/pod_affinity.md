## 将 Pod 分散到不同节点

多个副本分布在不同节点，可以减少单节点故障对服务的影响。 下面使用 Pod 反亲和性，让调度器优先把同一应用的副本放到不同节点。

### 一、配置优先反亲和性

保存为 `pod-affinity-demo.yaml`。将 `YOUR_NGINX_IMAGE` 替换为节点可拉取的 Nginx 镜像地址及固定版本。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spread-demo
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: spread-demo
  template:
    metadata:
      labels:
        app: spread-demo
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app: spread-demo
                topologyKey: kubernetes.io/hostname
      containers:
        - name: nginx
          image: YOUR_NGINX_IMAGE
          resources:
            requests:
              cpu: 100m
              memory: 64Mi
```

- `labelSelector` 匹配需要彼此分散的 Pod，需与模板中的标签一致；本例匹配同一命名空间内的 Pod。
- `topologyKey: kubernetes.io/hostname` 表示按节点区分拓扑域。
- `preferredDuringSchedulingIgnoredDuringExecution` 是优先规则，不保证每个副本都在不同节点。
  节点数量不足或资源、污点、存储等条件限制时，多个副本仍可能落在同一节点。
- `IgnoredDuringExecution` 表示运行中的 Pod 不会仅因相关条件变化而被自动迁移。

### 二、部署并检查分布

```bash
kubectl apply -f pod-affinity-demo.yaml
kubectl rollout status deployment/spread-demo -n default
kubectl get pods -n default -l app=spread-demo -o wide
```

查看输出中的 `NODE` 列判断副本分布，不能仅凭设置了反亲和性就认为已实现跨节点部署。

### 三、需要强制分散时

如业务要求副本不能位于同一节点，可改用 `requiredDuringSchedulingIgnoredDuringExecution`。 其配置结构与优先规则不同，不包含 `weight` 和
`podAffinityTerm` 包装层：

```yaml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: spread-demo
        topologyKey: kubernetes.io/hostname
```

这是替换 Pod 模板中 `spec.affinity` 的片段，不能作为独立资源提交。 强制规则需要足够多的合格节点；资源不足时 Pod 会处于 `Pending`，滚动更新也可能等待额外节点。
希望在节点或可用区之间控制副本分布差异时，可进一步使用拓扑分布约束。

更多说明参见 [Pod 亲和性与反亲和性](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)
和[拓扑分布约束](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/)。
