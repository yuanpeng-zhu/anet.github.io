在 Kubernetes（k8s）中，实现持久化存储（Persistent Storage）主要通过以下机制来完成，这些机制确保数据即使在 Pod 被销毁或重启后仍然可以保留和恢复：

## k8s持久化存储

### 1. **使用 PersistentVolume (PV) 和 PersistentVolumeClaim (PVC)**

#### **PersistentVolume (PV)**  
PV 是集群管理员创建的存储资源的抽象，它表示一个独立于具体 Pod 的存储卷，由管理员预先配置或动态提供。  

- PV 支持多种存储后端，如：
  - 网络存储（如 NFS、GlusterFS、Ceph、iSCSI）
  - 云存储（如 AWS EBS、GCP Persistent Disks、Azure Disks）
  - 本地存储（如主机目录或块设备）

#### **PersistentVolumeClaim (PVC)**  
PVC 是用户申请存储的方式，它定义了所需存储资源的大小、访问模式等。  
- PVC 会绑定到满足需求的 PV（手动或动态绑定）。

#### **访问模式（Access Modes）**
- `ReadWriteOnce` (RWO): 允许一个节点读写。
- `ReadOnlyMany` (ROX): 允许多个节点只读。
- `ReadWriteMany` (RWX): 允许多个节点读写。

##### 配置示例：
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: "/data/my-pv"

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi

---
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: my-container
    image: nginx
    volumeMounts:
    - mountPath: "/usr/share/nginx/html"
      name: my-volume
  volumes:
  - name: my-volume
    persistentVolumeClaim:
      claimName: my-pvc
```

---

### 2. **StorageClass 动态存储供应**
StorageClass 提供了一种动态创建 PV 的方法。管理员定义 StorageClass，用户通过 PVC 请求存储时，后端自动创建并绑定 PV。

#### **配置示例：**
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-storage
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
  storageClassName: fast-storage
```

---

### 3. **使用 StatefulSet**
StatefulSet 是一种适合有状态应用的工作负载类型。每个 Pod 会有独立的 PVC，确保每个实例有自己的存储。  
StatefulSet 常用于数据库（如 MySQL、PostgreSQL、MongoDB）。

#### 示例：
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: my-database
spec:
  serviceName: "database"
  replicas: 3
  selector:
    matchLabels:
      app: database
  template:
    metadata:
      labels:
        app: database
    spec:
      containers:
      - name: db
        image: mysql:5.7
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 10Gi
```

---

### 4. **HostPath (仅限测试或单节点环境)**  
HostPath 将存储卷直接挂载到主机的某个目录。  
- 仅适合开发测试，不建议在生产环境使用，因为它与节点绑定。

#### 示例：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
  - name: test-container
    image: busybox
    volumeMounts:
    - mountPath: "/test-data"
      name: host-volume
  volumes:
  - name: host-volume
    hostPath:
      path: "/data"
```

---

### 5. **云存储集成**
- 使用云提供商的存储插件（如 AWS EBS、GCP PD、Azure Disk）。
- 利用 CSI（Container Storage Interface）驱动统一管理存储后端。

#### 示例：
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: aws-ebs-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
  storageClassName: gp2
```


## k8s挂载NFS

在 Kubernetes 中，挂载 NFS（Network File System）到容器需要使用 `PersistentVolume` 和 `PersistentVolumeClaim`，或者直接使用 Pod 的 `volumes` 配置。以下是具体步骤：

---

### 1. **确保 NFS 服务已配置并可访问**
- 确认 NFS 服务器地址（例如：`192.168.1.100`）。
- 确认共享路径（例如：`/shared/nfs`）。
- 确保 NFS 客户端在 Kubernetes 节点上已安装（如 `nfs-common`）。

#### 在 Ubuntu 节点安装 NFS 客户端：
```bash
sudo apt update
sudo apt install -y nfs-common
```

---

### 2. **创建 PersistentVolume (PV)**
在 PV 中指定 NFS 的地址和共享路径。

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteMany # 允许多个节点同时挂载
  persistentVolumeReclaimPolicy: Retain
  nfs:
    server: 192.168.1.100 # NFS 服务器地址
    path: /shared/nfs     # NFS 共享路径
```

---

### 3. **创建 PersistentVolumeClaim (PVC)**
PVC 是 Pod 申请存储的方式，用来绑定到上一步创建的 PV。

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
```

---

### 4. **创建 Pod 并挂载 PVC**
在 Pod 的 `volumes` 中引用 PVC，将 NFS 挂载到容器。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nfs-test-pod
spec:
  containers:
  - name: test-container
    image: busybox
    command: [ "sleep", "3600" ]
    volumeMounts:
    - mountPath: "/mnt/nfs" # 容器内的挂载路径
      name: nfs-volume
  volumes:
  - name: nfs-volume
    persistentVolumeClaim:
      claimName: nfs-pvc
```

---

### 5. **直接在 Pod 中挂载 NFS（无需 PV/PVC，适合测试）**
如果不需要 PV/PVC，可以直接在 Pod 配置中使用 `nfs`。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nfs-direct-pod
spec:
  containers:
  - name: test-container
    image: busybox
    command: [ "sleep", "3600" ]
    volumeMounts:
    - mountPath: "/mnt/nfs" # 容器内的挂载路径
      name: nfs-volume
  volumes:
  - name: nfs-volume
    nfs:
      server: 192.168.1.100 # NFS 服务器地址
      path: /shared/nfs     # NFS 共享路径
```

---

### 6. **验证**
1. 部署 Pod 后，通过 `kubectl exec` 登录容器，验证挂载是否成功：
   ```bash
   kubectl exec -it nfs-test-pod -- sh
   cd /mnt/nfs
   touch test-file
   ```
2. 确认在 NFS 服务器上 `/shared/nfs` 中是否存在 `test-file`。

---

### 注意事项
1. **访问权限**  
   - 确保 NFS 服务器的共享路径配置了正确的访问权限，允许 Kubernetes 节点访问。
   - 配置示例（`/etc/exports` 文件）：
     ```bash
     /shared/nfs  *(rw,sync,no_subtree_check,no_root_squash)
     ```
   - 刷新配置：
     ```bash
     sudo exportfs -rav
     ```

2. **网络防火墙**  
   确保 NFS 相关端口（通常为 2049）在 Kubernetes 节点与 NFS 服务器之间是开放的。

3. **挂载模式**  
   使用 `ReadWriteMany` 模式时，确保共享路径支持多节点读写，适用于多个 Pod 共享数据场景。
