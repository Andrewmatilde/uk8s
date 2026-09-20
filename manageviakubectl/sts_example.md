## StatefulSet 部署示例

StatefulSet 用于需要稳定 Pod 名称和独立持久化存储的工作负载。 以下示例通过 Headless Service 提供稳定的网络标识，并通过 `volumeClaimTemplates`
为每个副本创建独立的 PVC。它展示存储与身份机制，不包含数据库复制或高可用配置。

### 一、准备工作

- 完成[安装及配置 kubectl](/uk8s/manageviakubectl/connectviakubectl)。
- 按[创建 PVC](/uk8s/manageviakubectl/createpvc)准备 `udisk-ssd-demo`，或选择实际可用的存储类。
- 确认节点可用区、云盘类型、存储容量和配额满足要求。RSSD 有额外限制，见 [在 UK8S 中使用 RSSD UDisk](/uk8s/volume/rssdudisk)。

```bash
kubectl get storageclass
```

### 二、创建 Headless Service 与 StatefulSet

保存为 `statefulset-demo.yaml`。将 `YOUR_NGINX_IMAGE` 替换为可拉取的 Nginx 镜像地址及固定版本， 并将 `storageClassName`
改成实际选择的存储类。应用示例将为三个副本分别申请云盘，可能产生费用。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-headless
  namespace: default
spec:
  clusterIP: None
  selector:
    app: statefulset-demo
  ports:
    - name: http
      port: 80
      targetPort: http
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
  namespace: default
spec:
  serviceName: web-headless
  replicas: 3
  selector:
    matchLabels:
      app: statefulset-demo
  template:
    metadata:
      labels:
        app: statefulset-demo
    spec:
      terminationGracePeriodSeconds: 30
      containers:
        - name: nginx
          image: YOUR_NGINX_IMAGE
          ports:
            - name: http
              containerPort: 80
          volumeMounts:
            - name: data
              mountPath: /data
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes:
          - ReadWriteOnce
        storageClassName: udisk-ssd-demo
        resources:
          requests:
            storage: 20Gi
```

`serviceName` 对应 Headless Service 名称；Service 和 StatefulSet 的选择器都与 Pod 标签匹配。 `volumeMounts[].name` 与
`volumeClaimTemplates[].metadata.name` 一致。 本例将数据卷挂载到 `/data`，保留 Nginx 默认网页目录用于访问测试。

### 三、查看运行情况

```bash
kubectl apply -f statefulset-demo.yaml
kubectl rollout status statefulset/web -n default
kubectl get pods -n default -l app=statefulset-demo -o wide
kubectl get pvc -n default
```

本例生成 `web-0`、`web-1`、`web-2`，以及对应的 `data-web-0`、`data-web-1`、`data-web-2` PVC。
副本间不会自动共享或同步云盘数据，数据复制由应用自身实现。

如果 Pod 一直处于 `Pending`，检查 Pod/PVC 的 Events、存储类的绑定模式以及可用区约束。

### 四、缩容与数据保留

默认情况下，缩容或删除 StatefulSet 不会自动删除它创建的 PVC。 如集群版本支持并配置了 `persistentVolumeClaimRetentionPolicy`，则按该策略处理。
删除 PVC 后，底层存储是否删除还取决于 PV 的回收策略；清理前请确认备份和数据保留要求。

更多说明见
[Kubernetes StatefulSet](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)。
