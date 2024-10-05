`Cgroups`（Control Groups，控制组）是Linux内核中的一个功能模块，用于对进程组进行资源管理和控制。它允许将系统资源（如CPU、内存、网络带宽、IO等）分配给不同的进程或进程组，并且能够对这些资源进行监控、限制和优先级分配。`Cgroups`的目标是提高系统的资源利用率，同时确保系统资源能够被合理地使用和隔离。

### 一、Cgroups的主要功能
Cgroups的核心功能主要体现在以下几个方面：
1. **资源限制（Resource Limiting）**：可以为特定的进程组设定资源上限，例如限制它们最多只能使用多少内存或多少CPU时间。
2. **优先级分配（Prioritization）**：可以通过设置资源的优先级，确保某些关键进程能获得更多的资源，避免被其他进程挤占。
3. **资源隔离（Resource Isolation）**：不同的进程组可以互相隔离资源使用，确保它们的操作不会相互影响。
4. **资源计费（Resource Accounting）**：可以监控和统计某个进程组使用了多少资源，如CPU时间、内存使用量、IO流量等。
5. **资源控制（Resource Control）**：能够实时动态地调整进程组的资源分配和优先级，适应负载变化的需求。
### Cgroups的层次结构
Cgroups以**层次结构**的方式组织，通过将进程放入特定的cgroup（控制组）来应用资源限制。每个cgroup形成一个树形层次，每个子节点继承其父节点的资源控制属性，子节点的资源限制会覆盖父节点的设定。每个进程只能属于一个特定的cgroup，但它可以属于不同的控制子系统。

层次结构的构成可以分为两个部分：
- **控制层次（Hierarchy）**：层次树中的节点称为cgroup，每个cgroup表示一组进程的集合。每个节点可以定义不同的资源控制策略。
- **控制器（Subsystems）**：控制器是Cgroups中负责管理某一类系统资源的模块，每个控制器管理一种或多种资源。常见的控制器包括对CPU、内存、IO等的控制。

### 常见的控制器（Subsystems）
Cgroups中的控制器负责管理和控制不同的资源，每个控制器实现特定类型的资源管理。以下是一些主要的控制器类型：
1. **cpu** 控制器
   - 该控制器用于限制和分配CPU时间。可以将某个cgroup中的进程分配一定的CPU时间份额，避免某个进程组占用过多的CPU资源。
   - 可以通过`cpu.shares`参数来设置CPU的相对权重，值越高，进程组获得的CPU时间越多。
   关键参数：
   - `cpu.shares`：设置CPU的权重，控制进程组获得的CPU份额。
   - `cpu.cfs_quota_us` 和 `cpu.cfs_period_us`：这些参数用来定义cgroup中的进程在指定的时间段内最多可以使用的CPU时间。
   举例：
   如果 `cpu.shares` 值设置为 512，另一个cgroup的值为 1024，后者将分配到前者两倍的CPU时间。
2. **cpuacct** 控制器
   - 该控制器用于对CPU的使用情况进行监控和统计。它不影响进程的实际运行，而是为每个进程组记录下其使用了多少CPU资源。
   关键文件：
   - `cpuacct.usage`：总的CPU时间（纳秒）。
   - `cpuacct.usage_percpu`：每个CPU核心的CPU时间。
3. **cpuset** 控制器
   - 该控制器用于将cgroup中的进程绑定到指定的CPU和内存节点（NUMA节点）。可以控制某些进程只能在特定的CPU上运行，或只能使用特定的内存节点，从而避免某些进程干扰其他进程。
   关键参数：
   - `cpuset.cpus`：指定可用的CPU核。
   - `cpuset.mems`：指定可用的内存节点。
4. **memory** 控制器
   - 该控制器负责限制和监控进程组的内存使用情况。可以为某个进程组设置内存的最大使用量，避免其使用过多内存而影响其他进程的运行。
   关键参数：
   - `memory.limit_in_bytes`：限制内存的最大使用量。
   - `memory.memsw.limit_in_bytes`：限制内存加交换分区的最大使用量。
   - `memory.usage_in_bytes`：当前的内存使用量。
   当一个进程使用超过指定的内存限制时，会触发内存回收，甚至在严重的情况下被强制终止（OOM）。
5. **blkio** 控制器
   - 该控制器用于限制和监控进程的块设备I/O操作，主要包括磁盘读写。可以对每个cgroup设置I/O带宽的上限，从而避免某些进程占用过多的磁盘I/O资源。
   关键参数：
   - `blkio.weight`：设置I/O操作的相对权重。
   - `blkio.throttle.read_bps_device` 和 `blkio.throttle.write_bps_device`：限制读取和写入时的带宽（字节/秒）。
6. **net_cls** 和 **net_prio** 控制器
   - 这两个控制器用于对网络资源进行管理。`net_cls` 控制器可以标记每个cgroup中的网络数据包，以便于网络流量的分类管理；`net_prio` 控制器则用于设置不同cgroup的网络流量优先级。
7. **devices** 控制器
   - `devices` 控制器可以限制进程访问某些设备（如块设备、字符设备等）。通过该控制器，可以控制进程组中进程对特定设备的访问权限（如读、写、执行权限）。
   关键参数：
   - `devices.allow` 和 `devices.deny`：控制哪些设备可以被进程访问。
8. **freezer** 控制器
   - `freezer` 控制器用于冻结或恢复cgroup中的所有进程。在冻结状态下，cgroup中的进程会停止执行，可以用于系统维护或调试目的。
   关键参数：
   - `freezer.state`：用于冻结或解冻进程。

### Cgroups 版本
目前Cgroups有两个主要版本：
1. **Cgroups v1**：
   - 每个控制器可以有独立的层次结构，可以为同一个进程在不同的资源控制器下应用不同的层次结构。
2. **Cgroups v2**：
   - 将所有控制器合并到一个单一的层次结构中，简化了管理。Cgroups v2也引入了更加严格的资源控制模型和新的API。
在Cgroups v1中，控制器的层次结构是相互独立的，因此进程可以属于不同的cgroup层次，但在Cgroups v2中，所有的控制器共享同一个层次结构，简化了配置和管理。

### Cgroups的工作流程
Cgroups的工作流程大致可以分为以下几个步骤：
1. **创建cgroup**：管理员或系统进程可以通过创建一个新的cgroup来为一组进程应用特定的资源控制策略。可以使用命令行工具（如`cgcreate`）或通过直接操作cgroup虚拟文件系统（通常挂载在`/sys/fs/cgroup`）来创建新的cgroup。
   例如，创建一个新的 CPU 控制组：
   ```
   mkdir /sys/fs/cgroup/cpu/mygroup
	```
1. **将进程加入cgroup**：通过向cgroup的`tasks`文件中写入进程ID（PID），可以将进程分配到某个cgroup中。该进程的所有子进程也会自动继承该cgroup的资源控制策略。
   例如，将进程 ID 为 1234 的进程加入到 `mygroup`：
   ```
	echo 1234 > /sys/fs/cgroup/cpu/mygroup/tasks
	```
1. **设置资源限制**：通过向cgroup的控制文件（如`cpu.shares`或`memory.limit_in_bytes`）中写入值，可以设定特定的资源限制策略。
   例如，为 `mygroup` 设置 CPU 权重：
   ```
	echo 512 > /sys/fs/cgroup/cpu/mygroup/cpu.shares
	```
4. **监控和调整**：管理员可以实时监控cgroup的资源使用情况，并动态调整资源配额，确保系统的稳定运行。例如，读取当前 CPU 使用情况：
```
	cat /sys/fs/cgroup/cpu/mygroup/cpuacct.usage
```
