## 使用web kubectl

UK8S在控制台中提供Web终端，无需在本地安装kubectl即可操作和管理集群。

1. 在UK8S集群列表找到目标集群，点击操作列中的「kubectl」，打开独立的命令行页面。

![集群列表中的kubectl入口](/images/manageviakubectl/web-kubectl-entry-current.png)

2. 等待终端显示提示符后，可执行以下命令检查客户端和访问权限。将`default`替换为实际授权的命名空间。

```bash
kubectl version --client
kubectl auth can-i get pods -n default
kubectl get pods -n default
```

![Web kubectl客户端及读取权限检查](/images/manageviakubectl/web-kubectl-terminal-current.png)

开启授权管理后，子账号终端使用自己的凭证，访问范围受RBAC权限限制，详见[授权管理](/uk8s/auth/rbac)。会话断开或超时后，可从集群列表重新打开。

终端持续不可用时，可通过[本地kubectl](/uk8s/manageviakubectl/connectviakubectl)连接排查，或联系技术支持。
