## k8s部署方式

Kubernetes（K8s）提供了多种资源类型来管理容器化应用程序的部署和运行。最常用的三种资源类型是 **Deployment**、**StatefulSet** 和 **DaemonSet**。每种资源类型都有其特定的功能和用法，适用于不同的场景和需求。下面详细介绍这三种资源类型的功能以及用法。

### 1. Deployment

**功能**：
- Deployment 是 Kubernetes 中用于管理无状态应用程序的控制器。它允许用户声明希望运行的 Pod 的数量以及其特定的状态。
- Deployment 支持滚动更新和回滚功能，使得应用程序的更新过程变得简单和安全。

**用法**：
- 常用于无状态应用程序，如 Web 服务器、API 服务器等。
- 用户可以使用 YAML 文件定义 Deployment，并通过 `kubectl` 命令进行创建和管理。

**示例**：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-container
        image: my-image:latest
        ports:
        - containerPort: 80
```

在这个示例中，创建了一个名为 `my-deployment` 的 Deployment，期望运行 3 个副本的 Pod，每个 Pod 中运行一个名为 `my-container` 的容器。

### 2. StatefulSet

**功能**：
- StatefulSet 是用于管理有状态应用程序的控制器。与 Deployment 不同，StatefulSet 保证了 Pod 的顺序性、稳定的网络标识和持久存储。
- StatefulSet 提供了稳定的持久化存储（通过 PVC）和有序的 Pod 启动和终止。

**用法**：
- 常用于需要保持状态的应用程序，如数据库（例如 MySQL、PostgreSQL）、分布式缓存（例如 Redis）等。
- 用户可以使用 YAML 文件定义 StatefulSet，并通过 `kubectl` 命令进行创建和管理。

**示例**：

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: my-statefulset
spec:
  serviceName: "my-service"
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-container
        image: my-image:latest
        ports:
        - containerPort: 80
  volumeClaimTemplates:
  - metadata:
      name: my-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
```

在这个示例中，创建了一个名为 `my-statefulset` 的 StatefulSet，期望运行 3 个副本的 Pod，每个 Pod 都有自己的 PVC（持久卷声明）。

#### Pod 名称的稳定性

每个 StatefulSet 管理的 Pod 都有一个稳定的、唯一的名字。Pod 的名称由 StatefulSet 的名称和一个序号组成，例如 `my-statefulset-0`、`my-statefulset-1` 等。这个命名方式确保了每个 Pod 在整个生命周期中都有一个可预测的名称。

#### 顺序的创建和删除

StatefulSet 在创建和删除 Pod 时遵循严格的顺序：

- **创建顺序**：当 StatefulSet 被创建时，Kubernetes 会按照顺序依次创建 Pod。即首先创建 `my-statefulset-0`，然后是 `my-statefulset-1`，依此类推。只有当前一个 Pod 完全运行并且就绪（ready）后，才会创建下一个 Pod。
  
- **删除顺序**：在删除 StatefulSet 中的 Pod 时，Kubernetes 也会按照相反的顺序进行。即首先删除 `my-statefulset-2`，然后是 `my-statefulset-1`，最后是 `my-statefulset-0`。这确保了有状态应用程序可以优雅地关闭，避免数据丢失或不一致。

#### 有序的滚动更新

StatefulSet 支持有序的滚动更新。在更新过程中，Kubernetes 会按照 Pod 的序号顺序逐一更新 Pod。只有当当前 Pod 更新完成并且就绪后，才会更新下一个 Pod。这种方式确保了应用程序在更新过程中始终保持一定数量的可用实例。

### 3. DaemonSet

**功能**：
- DaemonSet 确保集群中的每个（或特定的）节点上都运行一个 Pod 的副本。它通常用于运行集群级别的服务，如日志收集、监控、网络代理等。
- 当新节点加入集群时，DaemonSet 会自动在这些节点上启动相应的 Pod。

**用法**：
- 常用于需要在每个节点上运行的服务，如 Fluentd（日志收集）、Prometheus Node Exporter（监控）等。
- 用户可以使用 YAML 文件定义 DaemonSet，并通过 `kubectl` 命令进行创建和管理。

**示例**：

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: my-daemonset
spec:
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-container
        image: my-image:latest
```

在这个示例中，创建了一个名为 `my-daemonset` 的 DaemonSet，确保在集群中的每个节点上都运行一个 `my-container` 容器。

### 总结

- **Deployment**：适用于无状态应用程序，支持滚动更新和回滚，确保 Pod 的数量和状态。
- **StatefulSet**：适用于有状态应用程序，提供稳定的网络标识和持久存储，确保 Pod 的顺序性和稳定性。
- **DaemonSet**：确保在每个节点上运行一个 Pod 的副本，适用于集群级别的服务。

根据应用程序的需求，选择合适的资源类型可以帮助更好地管理和部署容器化应用。



## k8s 回滚、扩容、缩容

在Kubernetes中，Deployment、StatefulSet和DaemonSet都支持回滚、扩容和缩容操作，但它们的实现方式和适用场景有所不同。下面详细介绍这三种资源的回滚、扩容和缩容的操作方法。

### 1. Deployment

#### 回滚
Deployment支持版本控制，可以轻松地回滚到先前的版本。可以使用以下命令进行回滚：

```bash
kubectl rollout undo deployment my-deployment
```

如果需要回滚到特定的修订版本，可以使用：

```bash
kubectl rollout undo deployment my-deployment --to-revision=<revision-number>
```

#### 扩容
要扩容Deployment中的Pod副本数量，可以使用以下命令：

```bash
kubectl scale deployment my-deployment --replicas=<new-replica-count>
```

例如，将副本数扩展到5：

```bash
kubectl scale deployment my-deployment --replicas=5
```

#### 缩容
缩容与扩容相同，使用相同的`scale`命令来减少副本数量：

```bash
kubectl scale deployment my-deployment --replicas=<new-replica-count>
```

例如，将副本数缩减到2：

```bash
kubectl scale deployment my-deployment --replicas=2
```

### 2. StatefulSet

#### 回滚
StatefulSet本身不支持像Deployment那样的版本控制和回滚功能。如果需要回滚，通常需要手动修改StatefulSet的配置或使用`kubectl apply`重新应用之前的配置文件。

#### 扩容
要扩容StatefulSet中的Pod副本数量，可以使用以下命令：

```bash
kubectl scale statefulset my-statefulset --replicas=<new-replica-count>
```

例如，将副本数扩展到5：

```bash
kubectl scale statefulset my-statefulset --replicas=5
```

注意：扩容时，StatefulSet会按照顺序创建新的Pod（例如从`my-statefulset-0`到`my-statefulset-4`）。

#### 缩容
缩容同样使用`scale`命令，但要注意，缩容时会按照顺序删除Pod，通常会从后向前删除：

```bash
kubectl scale statefulset my-statefulset --replicas=<new-replica-count>
```

例如，将副本数缩减到2：

```bash
kubectl scale statefulset my-statefulset --replicas=2
```

### 3. DaemonSet

#### 回滚
DaemonSet也没有内置的回滚功能。如果需要回滚，通常需要手动修改DaemonSet的配置或使用`kubectl apply`重新应用之前的配置文件。

#### 扩容
DaemonSet的设计目标是在每个节点上运行一个Pod，因此扩容的概念在DaemonSet中不是很适用。如果需要在特定节点上运行额外的Pod，可以通过调整DaemonSet的选择器来实现。

#### 缩容
缩容DaemonSet通常是删除DaemonSet本身或通过修改DaemonSet的选择器来减少Pod的数量。可以使用以下命令删除DaemonSet：

```bash
kubectl delete daemonset my-daemonset
```

如果要保留DaemonSet但减少运行的Pod数量，则需要手动调整DaemonSet的选择器。

### 总结

- **Deployment**：支持简单的回滚、扩容和缩容操作，适用于无状态应用。
- **StatefulSet**：支持扩容和缩容，但没有回滚功能，需要手动管理版本和配置。
- **DaemonSet**：没有回滚和扩容的概念，主要用于确保在每个节点上运行一个Pod，缩容通常通过删除DaemonSet或调整选择器来实现。



## HPA自动扩缩容

**Horizontal Pod Autoscaler (HPA)** 是 Kubernetes 中用于自动扩展和缩减 Pod 副本数量的功能，基于 CPU 使用率或其他指标（如自定义指标或外部指标）。HPA 允许 Kubernetes 根据实时负载自动调整 Pod 的数量，从而实现高可用性和资源的高效利用。

### 1. HPA 的工作原理

HPA 的工作原理主要包括以下几个步骤：

1. **监控指标**：
   HPA 会定期（通常每 30 秒）检查指定的指标（如 CPU 使用率、内存使用率或自定义指标）。可以使用 Kubernetes Metrics Server 或其他监控解决方案（如 Prometheus）提供这些指标。

2. **计算目标值**：
   HPA 会根据定义的目标（例如，目标 CPU 使用率）计算需要的 Pod 数量。例如，如果目标是 CPU 使用率为 50%，而当前 CPU 使用率为 100%，HPA 将会计算出需要增加 Pod 数量。

3. **调整 Pod 数量**：
   根据计算出的需要的 Pod 数量，HPA 会调用 Kubernetes API 来更新 Deployment、StatefulSet 或其他控制器的副本数，从而实现扩容或缩容。

### 2. HPA 的配置

HPA 的配置通常通过 YAML 文件进行定义。以下是一个简单的 HPA 配置示例，基于 CPU 使用率自动扩展 Nginx Deployment：

```yaml
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-deployment
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50  # 目标 CPU 使用率
```

### 3. HPA 的关键字段

- **scaleTargetRef**：指定要扩展的目标对象（如 Deployment、StatefulSet）。
- **minReplicas**：最小 Pod 副本数量，HPA 不会将副本数缩减到此值以下。
- **maxReplicas**：最大 Pod 副本数量，HPA 不会将副本数增加到此值以上。
- **metrics**：定义监控指标，可以是资源指标（如 CPU、内存）或自定义指标。

### 4. 使用 HPA 的步骤

#### 步骤 1: 确保 Metrics Server 可用

首先，确保 Kubernetes 集群中安装并运行了 Metrics Server。可以通过以下命令检查：

```bash
kubectl get pods -n kube-system
```

查找名为 `metrics-server` 的 Pod。

#### 步骤 2: 创建 Deployment

创建一个 Deployment，例如 Nginx：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        resources:
          requests:
            cpu: "200m"  # 请求的 CPU
          limits:
            cpu: "500m"  # 限制的 CPU
```

使用以下命令创建 Deployment：

```bash
kubectl apply -f nginx-deployment.yaml
```

#### 步骤 3: 创建 HPA

创建 HPA 配置并应用：

```bash
kubectl apply -f hpa.yaml
```

#### 步骤 4: 验证 HPA

使用以下命令查看 HPA 的状态：

```bash
kubectl get hpa
```

您可以使用以下命令查看 HPA 的详细信息：

```bash
kubectl describe hpa nginx-hpa
```

### 5. HPA 的扩缩容过程

- **扩容**：当 CPU 使用率超过 50% 时，HPA 会根据需要增加 Pod 的副本数。例如，如果当前有 2 个 Pod，且 CPU 使用率达到 100%，HPA 可能会将副本数增加到 4 个或更多，直到达到最大副本数。

- **缩容**：当 CPU 使用率低于 50% 时，HPA 会逐渐减少 Pod 的副本数，直到达到最小副本数。

### 6. 注意事项

- **冷启动时间**：扩容后的 Pod 需要时间启动并开始接收流量，因此 HPA 不会立即响应负载的变化。可以设置 HPA 的 `behavior` 字段来控制扩容和缩容的速率。

- **自定义指标**：除了 CPU 和内存，HPA 还可以基于自定义指标进行扩缩容。需要使用 Prometheus Adapter 或其他适配器。

- **适用场景**：HPA 适用于负载变化较大的应用，例如 Web 应用、API 服务等。

### 总结

Horizontal Pod Autoscaler (HPA) 是 Kubernetes 中实现自动扩缩容的重要工具。通过监控实时指标，HPA 可以根据负载动态调整 Pod 的数量，从而确保应用程序的可用性和性能。正确配置和使用 HPA 可以显著提高资源利用率并优化应用性能。