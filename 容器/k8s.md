
## K8S

### 一、Kubernetes (K8s) 简介
Kubernetes (K8s) 是一个开源的容器编排平台，用于自动化部署、扩展和管理容器化应用程序。最初由 Google 开发，目前由云原生计算基金会（CNCF）维护和发展。

#### 核心功能
1. **容器编排**：管理成百上千的容器实例。
2. **负载均衡和服务发现**：自动分配请求到容器实例，支持高可用性。
3. **自动扩展**：根据流量和资源使用动态扩展或收缩容器。
4. **自愈能力**：自动重启失败的容器、替换宕机的节点。
5. **声明式管理**：通过配置文件定义系统状态，K8s负责实现和维护这一状态。

### 二、K8s 与 Docker 的关系
#### 1. Docker 的角色
Docker 是一个**容器化技术**平台，提供以下功能：
- **容器化**：打包和分发应用及其依赖。
- **运行时**：运行和管理容器实例。
- **镜像管理**：构建、存储、拉取容器镜像。

Docker 单独运行时，只能管理单个主机上的容器，无法处理跨主机的大规模容器集群管理。

#### 2. K8s 与 Docker 的互补关系
- **协作**：
  - Kubernetes 使用 Docker 作为容器运行时来运行容器。
  - K8s 通过声明式的方式管理容器的部署、扩展和服务，Docker 负责具体的容器生命周期管理。
- **限制**：
  - Docker 仅是众多容器运行时的一种，K8s 还支持其他运行时（如 containerd、CRI-O）。

#### 3. Docker 和 Kubernetes 的区别
| 特性               | Docker                          | Kubernetes                             |
|--------------------|---------------------------------|---------------------------------------|
| 核心功能            | 容器化、镜像构建和管理             | 容器编排、服务管理                    |
| 适用范围            | 单机管理容器                     | 集群中管理大规模容器                   |
| 提供的功能          | 容器构建、运行                   | 负载均衡、自动扩展、健康检查等           |
| 依赖关系            | 可作为 Kubernetes 的运行时之一    | 需要容器运行时支持（如 Docker、containerd） |

### 三、k8s各组件

![alt text](image.png)

#### 3.1 Master

Master 由四个部分组成：

1. **API Server 进程**  
   核心组件之一，为集群中各类资源提供增删改查的 HTTP REST 接口，即操作任何资源都要经过 API Server。Master 上`kube-system`
   空间中运行的 Pod 之一`kube-apiserver`是 API
   Server 的具体实现。与其通信有三种方式：

- 最原始的通过 REST API 访问；
- 通过官方提供的 Client 来访问，本质上也是 REST API 调用；
- 通过 kubectl 客户端访问，其本质上是将命令转换为 REST API 调用，是最主要的访问方式。

2. **etcd**  
   K8s 使用 etcd 作为内部数据库，用于保存集群配置以及所有对象的状态信息。只有 API Server 进程能直接读写
   etcd。为了保证集群数据安全性，建议为其考虑备份方案。
   如果 etcd 不可用，应用容器仍然会继续运行，但用户无法对集群做任何操作（包括对任何资源的增删改查）。


3. **调度器（Scheduler）**  
   它是 Pod 资源的调度器，用于监听刚创建还未分配 Node 的 Pod，为其分配相应 Node。
   调度时会考虑资源需求、硬件/软件/指定限制条件以及内部负载情况等因素，所以可能会调度失败。
   调度器也是操作 API Server 进程的各项接口来完成调度的。比如 Watch 接口监听新建的 Pod，并搜索所有满足 Pod 需求的 Node 列表，
   再执行 Pod 调度逻辑，调度成功后将 Pod 绑定到目标 Node 上。


4. **控制器管理器（kube-controller-manager）**  
   Controller 管理器实现了全部的后台控制循环，完成对集群的健康并对事件做出响应。Controller 管理器是各种 Controller 的管理者，负责创建 controller，并监控它们的执行。
   这些 Controller 包括 NodeController、ReplicationController 等，每个 controller 都在后台启动了一个独立的监听循环（可以简单理解为一个线程），负责监控 API Server 的变更。

- Node 控制器：负责管理和维护集群中的节点。比如节点的健康检查、注册/注销、节点状态报告、自动扩容等。
- Replication 控制器：确保集群中运行的 Pod 的数量与指定的副本数（replica）保持一致，稍微具体的说，对于每一个 Replication
  控制器管理下的 Pod 组，具有以下逻辑：
    - 当 Pod 组中的任何一个 Pod 被删除或故障时，Replication 控制器会自动创建新的 Pod 来作为替代
    - 当 Pod 组内的 Pod 数量超过所定义的`replica`数量时，Replication 控制器会终止多余的 Pod
- Endpoint 控制器：负责生成和维护所有 Endpoint 对象的控制器。Endpoint 控制器用于监听 Service 和对应 Pod 副本的变化
- ServiceAccount 及 Token 控制器：为新的命名空间创建默认账户和 API 访问令牌。
- 等等。

`kube-controller-manager`所执行的各项操作也是基于 API Server 进程的。

#### 3.2 Node

Node 由三部分组成：kubelet、kube-proxy 和容器运行时（如 docker/containerd）。

1. **kubelet**  
   它是每个 Node 上都运行的主要代理进程。kubelet 以 PodSpec 为单位来运行任务，后者是一种 Pod 的 yaml 或 json 对象。
   kubelet 会运行由各种方式提供的一系列 PodSpec，并确保这些 PodSpec 描述的容器健康运行。

不是 k8s 创建的容器不属于 kubelet 管理范围，kubelet 也会及时将 Pod 内容器状态报告给 API Server，并定期执行 PodSpec
描述的容器健康检查。
同时 kubelet 也负责存储卷等资源的管理。

kubelet 会定期调用 Master 节点上的 API Server 的 REST API 以报告自身状态，然后由 API Server 存储到 etcd 中。

2. **kube-proxy**  
   每个节点都会运行一个 kube-proxy Pod。它作为 K8s Service 的网络访问入口，负责将 Service 的流量转发到后端的 Pod，并提供
   Service 的负载均衡能力。
   kube-proxy 会监听 API Server 中 Service 和 Endpoint 资源的变化，并将这些变化实时反应到节点的网络规则中，确保流量正确路由到服务。
   总结来说，kube-proxy 主要负责维护网络规则和四层流量的负载均衡工作。

3. **容器运行时**  
   负责直接管理容器生命周期的软件。k8s 支持包含 docker、containerd 在内的任何基于 k8s cri（容器运行时接口）实现的 runtime。