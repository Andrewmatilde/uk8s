## 安装及配置kubectl

> 获取凭证需要相应的 IAM 权限，子账号还需获得集群的 RBAC 授权，详见[授权管理](/uk8s/auth/rbac)。

本文主要演示如何在UCloud云主机上安装配置kubectl并管理Kubernetes集群。如果机器上已安装兼容版本的kubectl，可跳过安装步骤。

**云主机环境**

操作系统：linux，windows请移步[官方文档](https://kubernetes.io/docs/tasks/tools/install-kubectl/)。

网络：使用内网凭证时，需与集群同VPC或已打通内网；使用外网凭证时，需能访问外网APIServer。下载安装包时需能访问下载源。

### 一、安装kubectl

1. 在集群「概览」中查看K8S版本，选择相同次版本的kubectl，版本偏差不要超过一个次版本。以下以Linux amd64和v1.34.5为例，请按实际版本及架构修改，勿直接安装不兼容的最新版本。

```bash
KUBECTL_VERSION="v1.34.5"
curl -fLO "https://dl.k8s.io/release/${KUBECTL_VERSION}/bin/linux/amd64/kubectl"
```

2. 添加执行权限

```
chmod +x ./kubectl
```

3. 移至工作路径

```
sudo mv ./kubectl /usr/local/bin/kubectl
```

4.输入kubectl version --client，确认客户端安装成功。

```bash
kubectl version --client
```

**备注**：如果您需要在ubuntu或其他linux发行版安装kubectl，亦或使用yum安装，可以参见[官方文档](https://kubernetes.io/docs/tasks/tools/install-kubectl/)。

### 二、获取并配置集群凭证

可以通过控制台获取当前账号的集群凭证。集群内访问同样需要身份认证和授权，并非免凭证访问。

1. 通过Console获取集群凭证

进入集群详情的「概览」页，在「APIServer 信息」中点击对应的「凭证」：内网连接选择「APIServer」，外网连接选择「外网APIServer」。如果没有外网入口，请使用内网连接或[Web kubectl](/uk8s/manageviakubectl/webterminal)。

![概览页的内外网凭证入口，资源信息已遮盖](/images/manageviakubectl/overview-current.png)

在弹窗中点击Copy，复制完整的KubeConfig。创建本地目录后，将配置保存为`~/.kube/config`；如果该文件已存在，请先备份，避免覆盖其他集群配置。

```bash
mkdir -p ~/.kube
chmod 700 ~/.kube
```

![内网集群凭证，正文已遮盖](/images/manageviakubectl/credentials-internal-current.png)

![外网集群凭证，正文已遮盖](/images/manageviakubectl/credentials-external-current.png)

保存后执行`chmod 600 ~/.kube/config`。配置可能包含Token或客户端私钥，请勿提交到代码仓库或公开展示。

2. 通过SCP从Master节点下载集群凭证到本地

仅适用于允许SSH访问Master节点的专有版集群。确认有权使用节点上的管理凭证并备份本地已有配置后，获取Master节点IP，在本地机器执行：

```
scp root@YOURMASTERIP:~/.kube/config ~/.kube/config
```

### 三、访问集群

你可以执行以下命令来验证kubectl是否可以成功访问集群信息；

```
kubectl cluster-info
```

### 四、设置命令自动补全

在kubectl所在节点执行安装

```
yum install bash-completion -y
```

kubectl支持命令自动补全，执行以下命令即可开启。

```
echo "source <(kubectl completion bash)" >> ~/.bashrc
```
