## 集群更新凭证

UK8S 支持用户通过 kubectl 工具连接 kubernetes 集群，详见[安装及配置kubectl](/uk8s/manageviakubectl/connectviakubectl)。

### 确认凭证类型

控制台获取的凭证可能使用Token或客户端证书，以KubeConfig中的`users[].user`字段为准：`token`表示Token，`client-certificate-data`和`client-key-data`表示客户端证书及私钥。

开启授权管理的集群，子账号使用独立凭证；其刷新与撤销由主账号或有权限的集群管理员在「授权管理」中操作，详见[RBAC授权管理](/uk8s/auth/rbac)。刷新后请由对应账号重新获取配置，并更新使用它的客户端。

### 更新 Token 访问集群

对于仍使用Token且提供「更新凭证」入口的历史集群，如凭证存在泄漏风险，可按控制台提示更新Token，之后重新获取并替换本地配置。此操作不会撤销客户端证书，不能认为所有旧凭证都会因此失效。

### 妥善保管好您的证书访问集群

Master节点上的管理配置可能使用独立的客户端证书，不会随Token刷新而更新。请妥善保管证书和私钥；若发生泄漏，应由集群管理员或技术支持确认适用的失效处理方式，重新复制配置文件本身不会使旧凭证失效。
