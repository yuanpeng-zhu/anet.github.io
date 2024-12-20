

TUN 和 TAP 是 Linux 系统中两种虚拟网卡设备，它们用于实现虚拟网络通信，广泛应用于 VPN、网络虚拟化和隧道技术中。

---

## **1. TUN 和 TAP 的基本概念**

### **TUN（Network TUNnel）**
- **工作层级**：TUN 设备工作在 **网络层（L3）**，负责处理 IP 数据包。
- **数据类型**：接收或发送的是 **IP 数据包**，即不包括链路层（MAC）头部。
- **应用场景**：用于实现隧道技术（如 VPN），将完整的 IP 数据包交给用户态程序处理。

### **TAP（Network TAP）**
- **工作层级**：TAP 设备工作在 **数据链路层（L2）**，负责处理以太网帧。
- **数据类型**：接收或发送的是 **完整的以太网帧**，包括 MAC 头部、IP 头部和数据。
- **应用场景**：用于网络虚拟化、模拟以太网设备（如虚拟机之间的通信）。

---

## **2. TUN 和 TAP 的原理**

### **虚拟网卡的作用**
- TUN 和 TAP 是虚拟网卡设备，它们模拟了物理网卡的行为，但没有实际的硬件。
- 它们将网络层或链路层的数据通过一个虚拟设备文件（`/dev/net/tun`）传递给用户态程序，允许用户态程序直接操作网络数据包。

### **数据流向**
1. **从内核到用户态：**
   - 内核将数据包发送到 TUN/TAP 设备。
   - 设备通过文件描述符将数据传递到用户态程序。

2. **从用户态到内核：**
   - 用户态程序向 TUN/TAP 设备写入数据。
   - 设备将数据注入内核协议栈，模拟正常的网络通信。

---

### **工作机制**
#### **TUN设备：网络层数据传递**
- 从 TUN 设备读到的数据：
  ```
  [IP头部 | 数据载荷]
  ```
- 从用户态写入 TUN 设备的数据：
  ```
  [IP头部 | 数据载荷]
  ```
- 数据进入内核时会直接注入到网络层（L3），如同通过物理网卡接收的 IP 包。


#### TUN 的工作方式
##### 虚拟网卡的角色：

TUN 设备就像一张虚拟网卡，你可以通过它发送和接收网络数据包。
它并不直接连接到物理网络，而是与运行在用户空间的程序（如 VPN 软件）配合工作。
只传 IP 包：

和实际网卡不同，TUN 设备只处理 IP 数据包（不包含 MAC 地址或以太网帧）。
双向数据流：

##### 从内核到用户空间：
当内核需要发送 IP 数据包到 TUN 设备时，数据包会从网络栈流向 TUN 设备，然后由用户空间的程序读取。
##### 从用户空间到内核：
用户空间程序可以写入 IP 数据包到 TUN 设备，内核会把它当作普通的网络数据进行处理和路由。

#### **TAP设备：链路层数据传递**
- 从 TAP 设备读到的数据：
  ```
  [以太网头部 | IP头部 | 数据载荷]
  ```
- 从用户态写入 TAP 设备的数据：
  ```
  [以太网头部 | IP头部 | 数据载荷]
  ```
- 数据进入内核时会直接注入到链路层（L2），如同通过物理网卡接收的以太网帧。

---

## **3. TUN 和 TAP 的功能**

| **功能**                  | **TUN设备**                           | **TAP设备**                            |
|--------------------------|--------------------------------------|---------------------------------------|
| **工作层级**              | 网络层（L3）                          | 数据链路层（L2）                       |
| **处理的数据类型**         | IP 数据包                             | 以太网帧                              |
| **典型用途**              | VPN（如 OpenVPN）                     | 虚拟机网络（如 QEMU、KVM）            |
| **头部信息**              | 只包含 IP 头部                        | 包括以太网头部和 IP 头部              |
| **性能开销**              | 相对较低（仅处理 L3 数据）             | 相对较高（需要处理 L2 和 L3 数据）     |

---

## **4. 使用方法**

### **创建 TUN/TAP 设备**
#### **前置条件**
1. 确保 Linux 内核启用了 TUN/TAP 支持：
   ```
   CONFIG_TUN=y
   ```
2. 安装必要的工具（如 `iproute2` 或 `tunctl`）。

#### **使用 `ip` 工具创建设备**
```bash
# 创建一个 TUN 设备
sudo ip tuntap add dev tun0 mode tun
sudo ip link set tun0 up

# 创建一个 TAP 设备
sudo ip tuntap add dev tap0 mode tap
sudo ip link set tap0 up
```

#### **使用 `tunctl` 工具创建设备**
```bash
# 创建一个 TUN 设备
sudo tunctl -t tun0
sudo ifconfig tun0 up

# 创建一个 TAP 设备
sudo tunctl -t tap0
sudo ifconfig tap0 up
```

---

### **与用户态程序通信**

#### **打开设备文件**
设备文件位于 `/dev/net/tun`，通过系统调用 `open()` 打开：
```c
int tun_fd = open("/dev/net/tun", O_RDWR);
```

#### **配置设备模式**
通过 `ioctl()` 配置设备为 TUN 或 TAP 模式：
```c
struct ifreq ifr;
memset(&ifr, 0, sizeof(ifr));
ifr.ifr_flags = IFF_TUN | IFF_NO_PI;  // 设置为 TUN 模式
strcpy(ifr.ifr_name, "tun0");         // 设备名称
ioctl(tun_fd, TUNSETIFF, &ifr);
```

#### **读写数据**
- 从 TUN/TAP 设备读取数据：
  ```c
  char buffer[1500];
  int n = read(tun_fd, buffer, sizeof(buffer));
  // 处理数据
  ```
- 向 TUN/TAP 设备写入数据：
  ```c
  write(tun_fd, buffer, n);
  ```

---

### **路由配置**
#### **为 TUN/TAP 配置 IP**
```bash
sudo ip addr add 10.0.0.1/24 dev tun0
```

#### **启用路由转发**
```bash
sudo sysctl -w net.ipv4.ip_forward=1
```

#### **设置路由规则**
```bash
sudo ip route add 10.0.0.0/24 dev tun0
```

---

## **5. 应用场景**

### **TUN设备的应用**
1. **VPN实现**
   - 如 OpenVPN、WireGuard，通过 TUN 捕获 IP 数据包，实现数据加密和隧道传输。
2. **隧道协议开发**
   - 使用 TUN 设备开发 GRE、IPsec 等协议。

### **TAP设备的应用**
1. **虚拟化网络**
   - 虚拟机（如 KVM、QEMU）之间通过 TAP 实现网络通信。
2. **容器网络**
   - Docker 和 Kubernetes 使用 TAP 设备构建容器之间的网络桥接。

---

## **6. 总结**

| **属性**    | **TUN设备**                      | **TAP设备**                        |
|------------|----------------------------------|-----------------------------------|
| **工作层级** | 网络层（L3）                     | 数据链路层（L2）                   |
| **数据类型** | IP 数据包                        | 以太网帧                          |
| **用途**    | 隧道协议、VPN、隧道加密           | 虚拟机、容器网络、以太网桥         |

TUN 和 TAP 是 Linux 网络虚拟化的核心组件，能够高效地将数据在内核和用户态之间传递。通过配置和程序开发，TUN 和 TAP 设备可用于实现复杂的网络功能，例如 VPN 和虚拟机互联。


https://zhuanlan.zhihu.com/p/388742230