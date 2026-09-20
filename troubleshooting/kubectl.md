# kubectl 常见问题

连接超时、身份认证失败、权限不足或证书错误，请先按 [安装及配置 kubectl：常见连接问题](/uk8s/manageviakubectl/connectviakubectl)排查。 本文介绍
`kubectl top` 无法获取资源指标时的检查方法。

## 一、确认资源指标 API 状态

`kubectl top` 依赖资源指标 API，通常由 Metrics Server 提供。 能够执行 `kubectl get pods` 不代表资源指标 API 一定可用。

```bash
kubectl top pods -n default
kubectl get apiservice v1beta1.metrics.k8s.io
kubectl describe apiservice v1beta1.metrics.k8s.io
```

查看 `Available` 条件及失败原因。上述 APIService 检查需要相应的集群级读取权限； 若返回 `Forbidden`，请由具备权限的管理员检查。

## 二、检查实际后端服务

从 APIService 配置读取后端的命名空间和 Service 名称，不要假定所有集群都使用相同名称：

```bash
kubectl get apiservice v1beta1.metrics.k8s.io -o yaml
```

根据 `spec.service.namespace` 和 `spec.service.name` 替换下列占位符：

```bash
kubectl get service <service-name> -n <namespace>
kubectl get endpointslices -n <namespace> -l kubernetes.io/service-name=<service-name>
kubectl get pods -n <namespace>
kubectl logs <metrics-server-pod> -n <namespace> --tail=100
```

结合 Events 和日志检查 Pod 就绪状态、Service 后端、Metrics Server 到节点 Kubelet 的连接、 证书校验及权限。新建 Pod 的指标可能需要等待采集后才可用。

Metrics Server 的版本需与 Kubernetes 版本兼容，要求及兼容矩阵见
[Metrics Server 官方说明](https://github.com/kubernetes-sigs/metrics-server)。

## 三、区分资源指标与自定义指标

- `metrics.k8s.io` 提供 CPU、内存等资源指标，供 `kubectl top` 及使用资源指标的 HPA 消费。
- `custom.metrics.k8s.io` 和 `external.metrics.k8s.io` 用于自定义指标或外部指标，通常由指标适配器提供。

部署 Prometheus 不意味着需要把资源指标 API 的后端改成 Prometheus。 如果集群使用自定义指标适配器，请先确认各 APIService 的用途、后端和 HPA
依赖，再由管理员修复。

不要直接覆盖 APIService 来“回退”指标服务，也不要以关闭 TLS 校验作为通用修复方法。 现代 Kubernetes 中，APIService 对象使用
`apiregistration.k8s.io/v1`； 对象名称 `v1beta1.metrics.k8s.io` 中的 `v1beta1` 是指标 API 的版本，两者不是同一概念。

更多说明见
[Kubernetes 资源指标管道](https://kubernetes.io/docs/tasks/debug/debug-cluster/resource-metrics-pipeline/)。
