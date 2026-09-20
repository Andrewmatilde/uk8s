## 创建 PVC

PVC（PersistentVolumeClaim）用于申请持久化存储。本文以 UDisk CSI 动态创建云盘并挂载到 Pod 为例。
执行前请完成[安装及配置 kubectl](/uk8s/manageviakubectl/connectviakubectl)。

### 一、选择 StorageClass

先查看实际可用的存储类，不要假定所有集群都有相同名称或数量的默认存储类：

```bash
kubectl get storageclass
kubectl describe storageclass <storage-class-name>
```

使用 UDisk CSI 时，检查 `provisioner` 为 `udisk.csi.ucloud.cn`，并确认存储类型、计费配置、
回收策略和可用区要求。创建存储类需要集群级权限，普通使用者可以引用管理员已创建的存储类。

如果需要新建 SSD 存储类，可将以下内容保存为 `storageclass.yaml`：

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: udisk-ssd-demo
provisioner: udisk.csi.ucloud.cn
parameters:
  type: "ssd"
  fsType: "ext4"
  chargeType: "dynamic"
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

```bash
kubectl apply -f storageclass.yaml
```

`WaitForFirstConsumer` 会等待使用该 PVC 的 Pod 参与调度后再分配存储，便于匹配节点所在可用区。 `Retain` 在 PVC 删除后保留 PV
和底层云盘，需要管理员后续处理，保留期间云盘仍可能产生费用。 使用 `Delete` 策略时，删除 PVC 可能连同底层云盘及数据一起删除。

更多参数和限制参见[在 UK8S 中使用 UDisk](/uk8s/volume/udisk)。 历史 FlexVolume 集群请按其存储插件方案处理，不要直接将旧 PV 的驱动字段替换为 CSI。

### 二、创建 PVC 并挂载到 Pod

保存为 `pvc-demo.yaml`。`udisk-ssd-demo` 对应上一步的存储类；如使用已有存储类，请替换名称。 将 `YOUR_NGINX_IMAGE` 替换为节点可拉取、包含 `df`
命令的 Nginx 镜像地址及固定版本。 私有镜像还需配置相应的 `imagePullSecrets`。

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: demo-data
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: udisk-ssd-demo
  resources:
    requests:
      storage: 20Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: pvc-demo
  namespace: default
spec:
  containers:
    - name: nginx
      image: YOUR_NGINX_IMAGE
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: demo-data
```

`20Gi` 是示例容量，实际容量、规格和计费限制以存储产品说明为准。 `ReadWriteOnce` 表示卷可由单个节点以读写方式挂载，不等同于只能被一个 Pod 使用。

应用示例会创建云盘并可能产生费用：

```bash
kubectl apply -f pvc-demo.yaml
kubectl get pvc,pods -n default
kubectl describe pvc demo-data -n default
kubectl exec pvc-demo -n default -- df -h /data
```

使用 `WaitForFirstConsumer` 时，尚未创建消费 Pod 的 PVC 处于 `Pending` 可以是正常现象。 若 Pod 创建后仍无法挂载，请结合 PVC 和 Pod 的
Events 检查存储类、CSI 插件、可用区及配额。

UFS 文件存储的使用方式见[在 UK8S 中使用 UFS](/uk8s/volume/ufs)。 存储类绑定机制参见
[Kubernetes StorageClass](https://kubernetes.io/docs/concepts/storage/storage-classes/)。
