## k8s调度

Kubernetes（通常简称为K8s）是一个用于自动化容器化应用程序的部署、扩展和管理的开源平台。在Kubernetes中，Pod是最基本的调度单元，而Pod的生命周期是Kubernetes管理资源和服务的核心部分。理解Kubernetes的生命周期有助于更好地设计和管理应用程序。下面详细解释Kubernetes Pod的生命周期及其不同阶段。

### Pod的生命周期

Pod的生命周期可以分为几个阶段，每个阶段都有不同的状态和条件：

1. **Pending（待调度）**：
   - 当Pod被创建时，它的状态为Pending。这表示Kubernetes正在调度Pod到合适的节点上。
   - 在这个阶段，Pod的容器还未被创建，可能是因为没有足够的资源，或者正在等待其他条件的满足。

2. **Running（运行中）**：
   - 一旦Pod的所有容器都成功创建并开始运行，Pod的状态就会变为Running。
   - 在这个阶段，Pod中的至少一个容器处于运行状态。
   - 如果容器在运行过程中崩溃，Kubernetes会尝试重启它。

3. **Succeeded（成功）**：
   - 如果Pod中的所有容器都成功完成并退出（通常是返回0），则Pod状态变为Succeeded。
   - 这通常适用于批处理任务或一次性任务。

4. **Failed（失败）**：
   - 如果Pod中的任何容器以非零状态代码退出，则Pod状态变为Failed。
   - 这表示容器在执行过程中遇到了错误。

5. **Unknown（未知）**：
   - 如果Kubernetes控制平面无法确定Pod的状态（例如，节点故障），则Pod状态将被标记为Unknown。

### Pod的生命周期管理

Kubernetes通过控制器（如Deployment、StatefulSet、DaemonSet等）来管理Pod的生命周期。控制器负责确保指定数量的Pod在运行，并根据需要进行创建、更新和删除。

#### 关键概念：

- **控制器**：负责管理Pod的状态和生命周期。例如，Deployment控制器可以确保在任何时候都有指定数量的Pod在运行。
  
- **探针（Probes）**：
  - Kubernetes使用探针（如就绪探针和活跃探针）来监控Pod中容器的健康状况。探针可以帮助Kubernetes决定何时重启容器或将流量发送到Pod。

- **生命周期钩子（Lifecycle Hooks）**：
  - Kubernetes提供了生命周期钩子（如`preStop`和`postStart`），允许用户在容器启动前或停止后执行特定的操作。

### Pod的终止

当Pod需要被删除时，Kubernetes会执行以下步骤：

1. **发出终止信号**：Kubernetes会向Pod中的容器发送终止信号（通常是SIGTERM），以通知它们即将停止。
2. **容器终止**：容器可以在接收到终止信号后执行清理操作。此时，容器可以在指定的时间内完成其运行（默认为30秒，称为`terminationGracePeriodSeconds`）。
3. **强制终止**：如果容器在指定的宽限期内没有结束，Kubernetes会强制终止容器（发送SIGKILL信号）。


## k8s preStop 与 postStart

在Kubernetes中，`preStop`和`postStart`是两种生命周期钩子（Lifecycle Hooks），允许用户在容器的特定生命周期事件发生时执行自定义操作。这些钩子可以帮助开发者在容器启动或停止之前进行必要的准备或清理工作。

### 1. `postStart`

`postStart`钩子在容器启动后立即执行。这个钩子会在容器的主进程启动之后、用户的应用程序开始运行之前被调用。使用`postStart`钩子可以用于执行一些在容器启动后需要立即完成的操作，例如初始化设置、启动后台服务、或者其他依赖于容器的环境准备工作。

#### 使用示例：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: example-pod
spec:
  containers:
  - name: example-container
    image: example-image
    lifecycle:
      postStart:
        exec:
          command: ["/bin/sh", "-c", "echo 'Container started' > /var/log/startup.log"]
```

在这个示例中，当容器启动后，会执行一个命令，将“Container started”写入`/var/log/startup.log`文件。

### 2. `preStop`

`preStop`钩子在容器被终止之前执行。它会在Kubernetes向容器发送终止信号（如`SIGTERM`）之前被调用。这使得开发者可以在容器关闭之前进行一些清理工作，比如关闭连接、保存状态、或执行其他必要的操作，以确保应用程序的优雅停机。

#### 使用示例：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: example-pod
spec:
  containers:
  - name: example-container
    image: example-image
    lifecycle:
      preStop:
        exec:
          command: ["/bin/sh", "-c", "echo 'Container is stopping' > /var/log/stop.log"]
```

在这个示例中，当容器被终止之前，会执行一个命令，将“Container is stopping”写入`/var/log/stop.log`文件。

### 钩子的执行顺序

- 在容器被终止时，Kubernetes会首先调用`preStop`钩子。
- `preStop`钩子的执行不应该超过容器的宽限期（`terminationGracePeriodSeconds`），否则Kubernetes会强制终止容器。
- 一旦`preStop`钩子完成或超时，Kubernetes会发送终止信号（`SIGTERM`），然后在宽限期结束后发送`SIGKILL`信号（如果容器未能正常终止）。

### 注意事项

- **异步执行**：`postStart`和`preStop`钩子的命令是异步执行的，Kubernetes不会等待命令完成后再继续执行后续操作。
- **错误处理**：如果钩子命令失败（返回非零状态码），Kubernetes会记录错误，但不会中止容器的启动或停止过程。
- **超时**：`preStop`钩子有一个默认的超时时间（即容器宽限期），如果在这个时间内未能完成，容器将被强制终止。

### 小结

`preStop`和`postStart`钩子在Kubernetes中提供了灵活性，使开发者能够在容器的特定生命周期事件中执行自定义操作。这对于确保应用程序的正常启动和优雅停机非常重要，可以帮助减少数据丢失和服务中断。