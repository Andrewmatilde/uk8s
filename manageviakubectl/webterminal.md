## 使用 Web kubectl

UK8S 提供浏览器终端，可在不安装本地客户端、不复制 kubeconfig 的情况下使用 kubectl。 终端操作仍受当前账号权限限制。

### 一、打开终端

1. 登录 UK8S 控制台，选择目标集群所在的项目和地域。
2. 在集群列表找到目标集群，点击操作列中的 **kubectl**。
3. 浏览器会打开独立的命令行页面，等待终端显示 Shell 提示符。

![集群列表操作列中的 kubectl 入口](/images/manageviakubectl/web-kubectl-entry-current.png)

### 二、检查客户端与访问权限

在终端中执行：

```bash
kubectl version --client
kubectl auth can-i get pods -n default
kubectl get pods -n default
```

子账号需将 `default` 替换为已授权的命名空间。`auth can-i` 返回 `yes` 表示允许该操作， 返回 `no` 表示权限不足。截图展示一次实际终端检查，客户端版本以当前集群为准。

![Web kubectl 中的客户端版本与读取权限检查](/images/manageviakubectl/web-kubectl-terminal-current.png)

常用命令参见 [kubectl 命令行简介](/uk8s/manageviakubectl/intro_of_kubectl)。 使用结束后关闭终端页面，不要将终端会话当作长期运行任务的环境。

### 三、账号权限与会话

开启授权管理后，子账号使用自己的集群凭证，资源和命名空间的访问范围取决于 RBAC 授权。 控制台终端还涉及 IAM 中的 `GetUK8STerminalToken` 权限。

子账号的终端会话使用临时 Pod，存在会话时长限制；断开或超时后可从集群列表重新打开。 具体机制及限制参见 [RBAC 授权管理：控制台命令](/uk8s/auth/rbac)。

### 四、常见问题

| 现象                           | 处理方法                                 |
| ---------------------------- | ------------------------------------ |
| 点击 kubectl 后没有新页面            | 检查浏览器是否拦截了控制台弹出窗口                    |
| 页面打开但无法连接                    | 检查集群状态、账号的终端权限，以及浏览器到终端服务的网络连接       |
| 命令返回 `Forbidden` 或权限检查为 `no` | 确认命名空间和 RBAC 授权；Web kubectl 不会绕过账号权限 |
| 会话已断开或超时                     | 关闭旧页面，从集群列表重新打开                      |
| 终端持续不可用                      | 使用本地 kubectl 连接排查，或联系技术支持检查终端组件      |

本地连接方式见[安装及配置 kubectl](/uk8s/manageviakubectl/connectviakubectl)。 不要直接套用旧版 `uk8s-kubectl:v1.14.6` 的
Deployment 和 RBAC 清单修复当前集群； 终端组件的镜像、身份与会话管理方式需要与集群版本和授权模式匹配。
