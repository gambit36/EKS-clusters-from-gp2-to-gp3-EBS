# EKS-clusters-from-gp2-to-gp3-EBS

Kubernetes是一个开源容器编排引擎，是一个由云原生计算基金会（CNCF）托管的快速增长的项目。 K8s 在本地和云中被大量采用，用于运行无状态和有状态的容器化工作负载。有状态工作负载需要持久存储。为了支持本地和与云提供商相关的基础设施，如存储和网络，Kubernetes 源代码最初包含所谓的“in-tree插件”。想要添加新存储系统或功能或只是想修复错误的存储和云供应商不得不依赖 Kubernetes 发布周期。为了将 Kubernetes 的生命周期与特定于供应商的实现分离，[容器存储接口](https://github.com/container-storage-interface/spec) (CSI) 的开发开始启动，CSI 是将任意块和文件存储系统暴露给 Kubernetes 等容器编排系统上的容器化工作负载的标准。客户现在可以从最新的 CSI 驱动程序中受益，而无需等待新的 Kubernetes 版本发布。

AWS 在 re:Invent 2017 上推出了他们的托管 Kubernetes 服务 Amazon Elastic Kubernetes Service (Amazon EKS)。 2019 年 9 月，AWS 在 Amazon EKS 中发布了对 Amazon Elastic Block Store (Amazon EBS) 容器存储接口 (CSI) 驱动程序的支持，并在 2021 年 5 月，AWS 宣布了此 CSI 插件的全面可用性。 Kubernetes Amazon EBS 相关的in-tree存储插件（称为“kubernetes.io/aws-ebs”类型的配置器）仅支持 Amazon EBS 类型 io1、gp2、sc1 和 st1，不支持卷快照相关功能。 

我们的客户现在询问何时以及如何将 EKS 集群从 Amazon EBS in-tree 插件迁移到 Amazon EBS CSI 驱动程序，以利用其他 EBS 卷类型（如 gp3 和 io2）并利用新功能（如 Kubernetes 卷快照）。 

自 Kubernetes v1.17 以来，容器存储接口 (CSI) 迁移基础设施一直处于测试功能状态，并在[此 K8s 博客](https://kubernetes.io/blog/2019/12/09/kubernetes-1-17-feature-csi-migration-beta/)文章中进行了详细描述。 迁移最终会从 Kubernetes 源代码中删除 in-tree 插件，并且所有迁移的卷将由 CSI 驱动程序控制，了解这一点很重要。 这并不意味着这些迁移的 PV 将获得 CSI 驱动程序的新功能和属性。 它仅支持已被树内驱动程序支持的功能，如[此处](https://github.com/kubernetes-csi/external-snapshotter/pull/490#issuecomment-813598646)所述。 

这篇博文将引导您完成迁移方案并详细概述必要的步骤。 

## 先决条件 
您需要一个版本为 1.17 或更高版本的 EKS 集群以及相应版本的 kubectl。 确保您已获得安装 Amazon EBS CSI 相关对象的授权。 

Kubernetes 使用所谓的[feature gates]https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/ 来实现存储迁移。 Amazon EBS 的 [CSIMigration 和 CSIMigrationAWS](https://kubernetes.io/docs/concepts/storage/volumes/#aws-ebs-csi-migration) 功能在启用时会将所有插件操作从现有的 in-tree 插件重定向到 ebs.csi.aws.com CSI 驱动程序。 请注意，Amazon EKS 尚未为 Amazon EBS 迁移启用 CSIMigration 和 CSIMigrationAWS 功能。 不过，您已经可以与 in-tree 插件并行使用 Amazon EBS CSI 驱动程序。 

为了演示起见，我们将创建一个动态 PersistentVolume (PV)，稍后我们将对其进行迁移。 

我们使用 [K8s 文档](https://kubernetes.io/docs/concepts/storage/dynamic-provisioning/)中描述的动态卷配置。

基于树内存储驱动程序的默认 StorageClass (SC) gp2 将用于创建 [PersistentVolumeClaim](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) (PVC) ： 

```
$ kubectl get sc
NAME                    PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
gp2   (default)     kubernetes.io/aws-ebs   Delete          WaitForFirstConsumer   false                  242d

$ cat ebs-gp2-claim.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ebs-gp2-claim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: gp2 

$ kubectl apply -f ebs-gp2-claim.yaml
persistentvolumeclaim/ebs-gp2-claim created
```

由于 gp2 StorageClass 具有 WaitForFirstConsumer 的 Volume Binding Mode（属性 volumeBindingMode）并且尚无 Pod 消费 PVC，因此 PVC 的创建状态为“pending”。 

```
$ kubectl get pvc ebs-gp2-claim
NAME            STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
ebs-gp2-claim   Pending                                      gp2            45s
```

因此，让我们创建一个使用 PVC 的 pod（我们的“演示应用程序”）： 
```
$ cat test-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-gp2-in-tree
spec:
  containers:
  - name: app
    image: centos
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo $(date -u) >> /data/out.txt; sleep 5; done"]
    volumeMounts:
    - name: persistent-storage
      mountPath: /data
  volumes:
  - name: persistent-storage
    persistentVolumeClaim:
      claimName: ebs-gp2-claim

$ kubectl apply -f test-pod.yaml
pod/app-gp2-in-tree created
```

几秒钟后，pod 被创建： 
```
$ kubectl get po app-gp2-in-tree
NAME              READY   STATUS    RESTARTS   AGE
app-gp2-in-tree   1/1     Running   0          16s
```

这将动态提供底层 PV pvc-646fef81-c677-46f4-8f27-9d394618f236，它现在绑定到 PVC “ebs-gp2-claim” 
```
$ kubectl get pvc ebs-gp2-claim
NAME            STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
ebs-gp2-claim   Bound    pvc-646fef81-c677-46f4-8f27-9d394618f236   1Gi        RWO            gp2            5m3s
```

让我们快速检查一下该卷是否包含一些数据： 
```
$ kubectl exec app-gp2-in-tree -- sh -c "cat /data/out.txt"
…
Thu Sep 16 13:56:34 UTC 2021
Thu Sep 16 13:56:39 UTC 2021
Thu Sep 16 13:56:44 UTC 2021
```

先来看看PV的细节： 
```
$ kubectl get pv pvc-646fef81-c677-46f4-8f27-9d394618f236
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                   STORAGECLASS   REASON   AGE
pvc-646fef81-c677-46f4-8f27-9d394618f236   1Gi        RWO            Delete           Bound    default/ebs-gp2-claim   gp2                     2m54s

$ kubectl get pv pvc-646fef81-c677-46f4-8f27-9d394618f236 -o yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  annotations:
    kubernetes.io/createdby: aws-ebs-dynamic-provisioner
    pv.kubernetes.io/bound-by-controller: "yes"
    pv.kubernetes.io/provisioned-by: kubernetes.io/aws-ebs
…
  labels:
    topology.kubernetes.io/region: eu-central-1
    topology.kubernetes.io/zone: eu-central-1c
  name: pvc-646fef81-c677-46f4-8f27-9d394618f236
…
spec:
  accessModes:
  - ReadWriteOnce
  awsElasticBlockStore:
    fsType: ext4
    volumeID: aws://eu-central-1c/vol-03d3cd818a2c2def3
…
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: topology.kubernetes.io/zone
          operator: In
          values:
          - eu-central-1c
        - key: topology.kubernetes.io/region
          operator: In
          values:
          - eu-central-1
  persistentVolumeReclaimPolicy: Delete
  storageClassName: gp2
…
```

PV 是（如预期的）由“kubernetes.io/aws-ebs”供应商创建的，如注释中所示。 规范部分中的“awsElasticBlockStore.volumeId”属性显示了实际的 Amazon EBS 卷 ID“vol-03d3cd818a2c2def3”以及创建 EBS 卷的 AWS 可用区 (AZ)——在本例中为 eu-central-1c。 EBS 卷和 EC2 实例是地区（非区域）资源。 nodeAffinity 部分建议 kube-scheduler 在创建 PV 的同一可用区中的节点上配置 pod。

以下命令是检索 Amazon EBS 详细信息的简短格式： 
```
$ kubectl get pv pvc-646fef81-c677-46f4-8f27-9d394618f236 –o jsonpath='{.spec.awsElasticBlockStore.volumeID}'
aws://eu-central-1c/vol-03d3cd818a2c2def3
```

我们希望将此 PV 用于稍后描述的存储迁移场景。 为了确保在删除相应的 PVC 时不会删除 PV，我们将把“VolumeReclaimPolicy”修补为“Retain”。 注意：这只能在 PV 级别而不是 SC 级别！ 
```
$ kubectl patch pv pvc-646fef81-c677-46f4-8f27-9d394618f236 -p '{"spec":{"persistentVolumeReclaimPol
icy":"Retain"}}'
persistentvolume/pvc-646fef81-c677-46f4-8f27-9d394618f236 patched
$ kubectl get pv pvc-646fef81-c677-46f4-8f27-9d394618f236
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                   STORAGECLASS   REASON   AGE
pvc-646fef81-c677-46f4-8f27-9d394618f236   1Gi        RWO            Retain           Bound    default/ebs-gp2-claim   gp2                     9m4s
```

要安装 Amazon EBS CSI 驱动程序，请按照我们的 GitHub 文档进行操作。 所需的高级步骤是：

* 将 Amazon EBS 操作所需的 IAM 权限附加到工作线程节点配置文件，或者按照最低权限使用 IRSA（服务账户的 IAM 角色）创建正确注释的 ServiceAccount (SA)
* 使用 YAML 安装外部卷快照控制器相关的 K8s 对象（CRD、RBAC 资源、部署和验证 webhook）
* 使用 YAML 或使用相应的 Helm chart 安装 Amazon EBS CSI 驱动程序（如果您使用 IRSA，请使用现有的服务帐户） 

仔细检查 Amazon EBS CSI 相关的 Kubernetes 组件是否已注册到 K8s API 服务器： 
```
$ kubectl api-resources | grep "storage.k8s.io/v1"
volumesnapshotclasses                          snapshot.storage.k8s.io/v1               false        VolumeSnapshotClass
volumesnapshotcontents                         snapshot.storage.k8s.io/v1               false        VolumeSnapshotContent
volumesnapshots                                snapshot.storage.k8s.io/v1               true         VolumeSnapshot
csidrivers                                     storage.k8s.io/v1                        false        CSIDriver
csinodes                                       storage.k8s.io/v1                        false        CSINode
csistoragecapacities                           storage.k8s.io/v1beta1                   true         CSIStorageCapacity
storageclasses                    sc           storage.k8s.io/v1                        false        StorageClass
volumeattachments                              storage.k8s.io/v1                        false        VolumeAttachment
```

确认它们已启动并正在运行：
```
$ kubectl get po -n kube-system -l 'app in (ebs-csi-controller,ebs-csi-node,snapshot-controller)'
NAME                                   READY   STATUS    RESTARTS   AGE
ebs-csi-controller-569b794b57-md99s    6/6     Running   0          6d15h
ebs-csi-controller-569b794b57-trkks    6/6     Running   0          6d15h
ebs-csi-node-4fkb8                     3/3     Running   0          6d14h
ebs-csi-node-vc48t                     3/3     Running   0          6d14h
snapshot-controller-6984fdc566-4c49f   1/1     Running   0          6d15h
snapshot-controller-6984fdc566-jlnbn   1/1     Running   0          6d15h
$ kubectl get csidrivers
NAME              ATTACHREQUIRED   PODINFOONMOUNT   STORAGECAPACITY   TOKENREQUESTS   REQUIRESREPUBLISH   MODES        AGE
ebs.csi.aws.com   true             false            false             <unset>         false               Persistent   7d19h
```

## 迁移场景 
首先，我们将在高层次上讨论迁移方案。

我们正在通过使用 Amazon EBS 快照作为外部快照机制复制基于in-tree的 PV 数据来进行物理存储迁移（注意：K8s in-tree Amazon EBS 插件不支持卷快照！）并使用 CSI 卷导入此数据 CSI Amazon EBS 驱动程序的[快照功能](https://kubernetes.io/docs/concepts/storage/volume-snapshots)。

现在我们将详细指导您完成迁移。

我们首先通过 AWS API 拍摄基于in-tree插件的 PV pvc-646fef81-c677-46f4-8f27-9d394618f236 的快照。 
```
$ kubectl get pv pvc-646fef81-c677-46f4-8f27-9d394618f236 -o jsonpath='{.spec.awsElasticBlockStore.volu
meID}'
aws://eu-central-1c/vol-03d3cd818a2c2def3

$ aws ec2 create-snapshot --volume-id vol-03d3cd818a2c2def3 --tag-specifications 'ResourceType=snapshot,Tags=[{Key="ec2:ResourceTag/ebs.csi.aws.com/cluster",Value="true"}]
{
…
    "SnapshotId": "snap-06fb1faafc1409cc5",
…
    "State": "pending",
    "VolumeId": "vol-03d3cd818a2c2def3",
    "VolumeSize": 1,
…    
}
```

等到快照处于“已完成”状态 
```
$ aws ec2 describe-snapshots --snapshot-ids snap-06fb1faafc1409cc5
{
    "Snapshots": [
        {
…
            "Progress": "100%",
            "SnapshotId": "snap-06fb1faafc1409cc5",
..
            "State": "completed",
            "VolumeId": "vol-03d3cd818a2c2def3",
…
        }
    ]
}
```

现在创建一个 VolumeSnapshotClass 对象： 
```
$ cat vsc-ebs-csi.yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: ebs-csi-aws
driver: ebs.csi.aws.com
deletionPolicy: Delete

$ kubectl apply -f vsc-ebs-csi.yaml
volumesnapshotclass.snapshot.storage.k8s.io/ebs-csi-aws created

$ kubectl get volumesnapshotclass
NAME          DRIVER            DELETIONPOLICY   AGE
ebs-csi-aws   ebs.csi.aws.com   Delete           12s
```

接下来，我们必须创建一个 VolumeSnapshotContent 对象，该对象使用 AWS 快照 snap-06fb1faafc1409cc5 并且已经引用了我们将在下一步中创建的 VolumeSnapshot。 这看起来很奇怪，但对于预先存在的快照的 VolumeSnapshotContent 和 VolumeSnapshot 的双向绑定是必要的！ 
```
$ cat vsc-csi.yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotContent
metadata:
  name: imported-aws-snapshot-content
spec:
  volumeSnapshotRef:
    kind: VolumeSnapshot
    name: imported-aws-snapshot
    namespace: default
  source:
    snapshotHandle: snap-06fb1faafc1409cc5 # <-- snapshot to import
  driver: ebs.csi.aws.com
  deletionPolicy: Delete
  volumeSnapshotClassName: ebs-csi-aws

$ kubectl apply -f vsc-csi.yaml
volumesnapshotcontent.snapshot.storage.k8s.io/imported-aws-snapshot-content created

$ kubectl get volumesnapshotcontent imported-aws-snapshot-content
NAME                            READYTOUSE   RESTORESIZE   DELETIONPOLICY   DRIVER            VOLUMESNAPSHOTCLASS   VOLUMESNAPSHOT          VOLUMESNAPSHOTNAMESPACE   AGE
imported-aws-snapshot-content   true         1073741824    Delete           ebs.csi.aws.com   ebs-csi-aws           imported-aws-snapshot   default                   12s
```
