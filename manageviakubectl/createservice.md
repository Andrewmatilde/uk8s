## 创建 Service

Service 为一组 Pod 提供稳定的访问入口。以下示例创建一个 Nginx Deployment， 并使用 `LoadBalancer` 类型的 Service，通过 UCloud
内网负载均衡供内网客户端访问。

执行前请完成[安装及配置 kubectl](/uk8s/manageviakubectl/connectviakubectl)，并确认集群的 CloudProvider
组件可正常工作。应用示例会申请负载均衡资源，可能产生费用。

### 一、准备资源清单

将以下内容保存为 `service-demo.yaml`。将 `YOUR_NGINX_IMAGE` 替换为节点可拉取的 Nginx 镜像地址及固定版本，镜像中的服务应监听 TCP 80 端口。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: service-demo
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: service-demo
  template:
    metadata:
      labels:
        app: service-demo
    spec:
      containers:
        - name: nginx
          image: YOUR_NGINX_IMAGE
          ports:
            - name: http
              containerPort: 80
          readinessProbe:
            httpGet:
              path: /
              port: http
            initialDelaySeconds: 3
            periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: service-demo
  namespace: default
  annotations:
    service.beta.kubernetes.io/ucloud-load-balancer-type: "inner"
spec:
  type: LoadBalancer
  selector:
    app: service-demo
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: http
```

Service 的 `selector` 必须与 Pod 的标签一致。`port` 是 Service 端口，`targetPort` 对应容器端口。 私有镜像需在 Pod 模板中配置
`imagePullSecrets`，所引用的 Secret 必须位于同一命名空间； 公开镜像不需要填写占位的 Secret 名称。

本例显式选择内网负载均衡。需要公网入口时，请在创建前按 [ULB 参数说明](/uk8s/service/annotations)配置网络类型、负载均衡类型和 EIP 等参数， 并确认
CloudProvider 版本支持。不要假定负载均衡的所有参数都能在创建后修改。

### 二、创建并验证

```bash
kubectl apply -f service-demo.yaml
kubectl rollout status deployment/service-demo -n default
kubectl get service service-demo -n default
kubectl describe service service-demo -n default
kubectl get endpointslices -n default -l kubernetes.io/service-name=service-demo
```

负载均衡创建需要时间。`kubectl get service` 的 `EXTERNAL-IP` 列表示负载均衡入口， 内网负载均衡也可能在该列显示私网地址。请从能够访问该内网地址的机器测试：

```bash
curl http://<load-balancer-address>
```

若入口一直处于 `Pending`，查看 Service 的 Events 和 CloudProvider 状态。 若入口已分配但应用不可访问，检查 Pod 是否就绪、Service
标签是否匹配、EndpointSlice 是否有后端， 以及负载均衡的网络和健康检查配置。
