## kubectl 命令行简介

kubectl 通过 Kubernetes API 管理集群资源。首次使用请先完成
[安装及配置 kubectl](/uk8s/manageviakubectl/connectviakubectl)，或打开
[Web kubectl](/uk8s/manageviakubectl/webterminal)。

### 命令语法

```bash
kubectl [command] [TYPE] [NAME] [flags]
```

| 参数        | 含义                   | 示例                                     |
| --------- | -------------------- | -------------------------------------- |
| `command` | 要执行的操作               | `get`、`describe`、`apply`、`delete`      |
| `TYPE`    | 资源类型，常用类型支持缩写        | `pods` / `po`、`deployments` / `deploy` |
| `NAME`    | 资源名称；省略时通常查询该类型的资源列表 | `demo`                                 |
| `flags`   | 指定命名空间、配置文件或输出格式等    | `-n default`、`-o wide`                 |

用 `kubectl api-resources` 查看集群支持的资源类型，用 `kubectl <command> --help` 查看命令帮助。 下文的
`<pod-name>`、`<container-name>` 等均为占位符，执行前请替换。

### 确认目标集群与命名空间

```bash
# 查看已有上下文及当前上下文
kubectl config get-contexts
kubectl config current-context

# 切换上下文（会修改本地 kubeconfig）
kubectl config use-context <context-name>
```

对命名空间资源，未指定 `-n` 时使用当前 context 中配置的命名空间；没有配置时才使用 `default`。 建议在日常命令中显式指定 `-n`。`-A`
表示所有命名空间，访问范围仍受权限限制。

### 查看资源与排查问题

```bash
kubectl get pods -n default
kubectl get pods -n default -o wide
kubectl get deployments,services -n default
kubectl get pods -A
kubectl get nodes

# 查看某个 Pod 的状态、容器信息和相关事件
kubectl describe pod <pod-name> -n default

# 按时间查看命名空间中的事件
kubectl get events -n default --sort-by=.metadata.creationTimestamp

# 检查当前身份是否具有读取权限
kubectl auth can-i get pods -n default
```

`describe` 中的 Events 用于排查调度、镜像拉取、挂载等问题；应用日志需要使用 `logs`。 子账号没有节点或其他命名空间的读取权限时，相关命令可能返回 `Forbidden`。

### 查看容器日志

```bash
# 查看最近 100 行日志
kubectl logs <pod-name> -n default --tail=100

# 持续查看指定容器的日志
kubectl logs -f <pod-name> -c <container-name> -n default

# 查看同一 Pod 中该容器上一次运行的日志（例如重启前）
kubectl logs <pod-name> -c <container-name> -n default --previous
```

### 在容器中执行命令

使用 `--` 分隔 kubectl 参数和容器内命令。多容器 Pod 建议显式使用 `-c` 选择容器。

```bash
kubectl exec <pod-name> -c <container-name> -n default -- date
kubectl exec -it <pod-name> -c <container-name> -n default -- /bin/sh
```

命令或 Shell 必须存在于容器镜像中；精简镜像不一定包含 `/bin/sh` 或 `/bin/bash`。 `exec` 中执行的命令可能修改应用数据，请先确认目标容器。

### 创建与更新资源

将资源清单保存为 `app.yaml`，先检查目标集群及变更内容，再应用：

```bash
kubectl config current-context
kubectl apply --dry-run=server -f app.yaml -n default
kubectl diff -f app.yaml -n default
kubectl apply -f app.yaml -n default
```

`--dry-run=server` 请求服务端校验但不持久化资源；`diff` 展示预期差异，存在差异时退出码为 `1`。 `apply` 会创建或更新清单中的资源。清单如果写有
`metadata.namespace`，应与 `-n` 一致。

对于 Deployment，可查看发布进度：

```bash
kubectl rollout status deployment/<deployment-name> -n default
```

### 删除资源

以下操作会删除目标资源，仅在确认不再需要时执行。删除 PVC 还可能根据存储回收策略删除底层数据。

```bash
kubectl delete -f app.yaml -n default
```

更多命令参见 [kubectl 官方速查表](https://kubernetes.io/docs/reference/kubectl/quick-reference/)。
