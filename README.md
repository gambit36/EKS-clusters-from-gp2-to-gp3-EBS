# EKS-clusters-from-gp2-to-gp3-EBS

Kubernetes是一个开源容器编排引擎，是一个由云原生计算基金会（CNCF）托管的快速增长的项目。 K8s 在本地和云中被大量采用，用于运行无状态和有状态的容器化工作负载。有状态工作负载需要持久存储。为了支持本地和与云提供商相关的基础设施，如存储和网络，Kubernetes 源代码最初包含所谓的“in-tree插件”。想要添加新存储系统或功能或只是想修复错误的存储和云供应商不得不依赖 Kubernetes 发布周期。为了将 Kubernetes 的生命周期与特定于供应商的实现分离，[容器存储接口](https://github.com/container-storage-interface/spec) (CSI) 的开发开始启动，CSI 是将任意块和文件存储系统暴露给 Kubernetes 等容器编排系统上的容器化工作负载的标准。客户现在可以从最新的 CSI 驱动程序中受益，而无需等待新的 Kubernetes 版本发布。
