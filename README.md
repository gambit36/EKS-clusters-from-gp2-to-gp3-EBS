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
