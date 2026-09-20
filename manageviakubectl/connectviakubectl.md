## 安装及配置 kubectl

kubectl 是 Kubernetes 的命令行工具。本文介绍如何安装客户端、从 UK8S 控制台获取
kubeconfig，并连接集群。如果只需临时操作，可使用[Web kubectl](/uk8s/manageviakubectl/webterminal)。

### 一、准备工作

- 确认已选择正确的项目和地域，并能在 UK8S 控制台查看目标集群。
- 确认当前账号具有获取凭证的 IAM 权限；子账号还需获得集群内资源的 RBAC 授权，详见
  [IAM 授权管理](/uk8s/auth/IAM)和[RBAC 授权管理](/uk8s/auth/rbac)。
- 在集群详情的「概览」页查看「K8S版本」，安装与集群兼容的 kubectl。
- 确认运行 kubectl 的机器能够访问所选 APIServer 地址及端口。

| 连接方式 | 适用环境                               | 凭证入口                 |
| ---- | ---------------------------------- | -------------------- |
| 内网连接 | 同 VPC 的云主机，或已通过专线、VPN 等方式打通集群内网的机器 | 「APIServer」右侧的「凭证」   |
| 外网连接 | 无法直接访问集群内网、且集群已提供外网 APIServer 的机器  | 「外网APIServer」右侧的「凭证」 |

以控制台实际显示为准。如果没有外网 APIServer，请使用内网连通的机器或 Web kubectl。 安装客户端下载需要访问下载源，但使用内网凭证管理集群不要求云主机开通外网。

![概览页中的 Kubernetes 版本与内外网凭证入口，资源信息已遮盖](/images/manageviakubectl/overview-current.png)

### 二、安装 kubectl

建议选择与集群相同次版本的客户端。kubectl 与 kube-apiserver 的版本偏差应在一个次版本以内， 详见
[Kubernetes 版本偏差策略](https://kubernetes.io/releases/version-skew-policy/#kubectl)。 不要直接安装最新版本而忽略集群版本。

以下以 Linux 为例。将 `KUBECTL_VERSION` 替换为目标版本；`v1.34.5` 仅为示例。

```bash
KUBECTL_VERSION="v1.34.5"

# x86_64 机器使用 amd64；aarch64 / ARM64 机器改为 arm64。
KUBECTL_ARCH="amd64"

curl -fLO "https://dl.k8s.io/release/${KUBECTL_VERSION}/bin/linux/${KUBECTL_ARCH}/kubectl"
curl -fLO "https://dl.k8s.io/release/${KUBECTL_VERSION}/bin/linux/${KUBECTL_ARCH}/kubectl.sha256"
```

校验下载内容：

```bash
echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check
```

只有显示 `kubectl: OK` 后，才继续安装：

```bash
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
```

其他安装方式和操作系统请参考官方说明： [Linux](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)、
[macOS](https://kubernetes.io/docs/tasks/tools/install-kubectl-macos/)、
[Windows](https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/)。同样需要选择兼容版本。

### 三、获取并保存 kubeconfig

1. 在 UK8S 集群列表中点击目标集群的「详情」，进入「概览」。
2. 在「APIServer 信息」中，按所需连接方式点击对应的「凭证」。
3. 在「内网集群凭证」或「外网集群凭证」弹窗中点击 **Copy**，复制完整的 KubeConfig。

![内网集群凭证弹窗，完整配置内容已遮盖](/images/manageviakubectl/credentials-internal-current.png)

![外网集群凭证弹窗，完整配置内容已遮盖](/images/manageviakubectl/credentials-external-current.png)

> kubeconfig 可能包含 Token 或客户端私钥，持有者可使用其中的身份访问集群。 请妥善保管，不要提交到代码仓库、粘贴到工单或公开截图中。上图已遮盖真实配置。

建议为每个集群保存独立文件，避免覆盖已有的 `~/.kube/config`。以下命令适用于 Linux/macOS：

```bash
mkdir -p "$HOME/.kube"
chmod 700 "$HOME/.kube"
```

使用文本编辑器将复制的完整 YAML 保存到 `~/.kube/uk8s-demo.yaml`，再设置文件权限：

```bash
chmod 600 "$HOME/.kube/uk8s-demo.yaml"
```

`uk8s-demo.yaml` 是本地文件名示例，可自行修改。保留原始缩进，不要把截图中的遮盖区域当作配置。 Windows 用户可保存到
`%USERPROFILE%\.kube\uk8s-demo.yaml`，并限制文件访问权限。

### 四、验证连接

先显式指定配置文件检查上下文，再查询集群。以下命令不会修改集群资源：

```bash
kubectl --kubeconfig="$HOME/.kube/uk8s-demo.yaml" config current-context
kubectl --kubeconfig="$HOME/.kube/uk8s-demo.yaml" cluster-info
kubectl --kubeconfig="$HOME/.kube/uk8s-demo.yaml" get pods -n default
```

如果账号只被授权访问特定命名空间，将 `default` 换成已授权的命名空间。 `No resources found` 表示查询成功但没有匹配资源；`Forbidden`
表示当前身份没有相应权限。

验证成功后，可在当前终端指定默认使用的文件，后续命令无需重复传入 `--kubeconfig`：

```bash
export KUBECONFIG="$HOME/.kube/uk8s-demo.yaml"
kubectl config current-context
```

上述环境变量仅影响当前终端及其子进程。未设置 `KUBECONFIG`、也未指定 `--kubeconfig` 时， kubectl 默认读取 `~/.kube/config`。多个集群的管理方式见
[配置多集群访问](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/)。

### 五、常见连接问题

| 现象                      | 排查方向                                       |
| ----------------------- | ------------------------------------------ |
| 连接超时、`no route to host` | 检查所选内外网入口、路由、VPN 及安全规则，确认可访问配置中的地址和端口      |
| `connection refused`    | 确认地址、端口及 APIServer 状态，避免使用过期的地址            |
| `Unauthorized`          | 检查凭证是否过期、被刷新或撤销，并重新获取当前账号的凭证               |
| `Forbidden`             | 身份认证通常已通过，但缺少目标资源或命名空间的 RBAC 权限            |
| `x509` 证书错误             | 检查系统时间、凭证有效期、CA 和访问地址是否匹配；重新获取对应入口的完整配置    |
| 访问到其他集群                 | 检查 `--kubeconfig`、`KUBECONFIG` 与当前 context |

不要用 `--insecure-skip-tls-verify` 绕过证书错误。凭证更新方法见 [集群凭证管理与更新](/uk8s/manageviakubectl/reset_token)。

集群内的程序同样需要身份认证和授权，通常使用 ServiceAccount；“在集群内访问”不等于免凭证。

### 六、设置命令补全（可选）

Bash 需先安装并加载系统的 `bash-completion`，再在当前终端执行：

```bash
source <(kubectl completion bash)
```

Zsh 可执行：

```zsh
autoload -Uz compinit
compinit
source <(kubectl completion zsh)
```

需要永久生效时，将适用的命令加入对应 Shell 的启动文件，避免重复添加。
