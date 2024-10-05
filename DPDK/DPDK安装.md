系统环境
操作系统: Win11
虚拟机软件：VMware Workstation 16
虚拟机系统：Ubuntu20.04，CPU个数8，内存8G，网卡个数 3
要安装的 DPDK 版本：22.11

## 1. 编译安装 DPDK
### 先安装一些依赖的软件包：
```
apt-get install -y meson python3-pyelftools pkg-config gcc make git net-tools
```
### 再编译安装 DPDK：
```
wget https://fast.dpdk.org/rel/dpdk-22.11.tar.xz
tar xf dpdk-22.11.tar.xz
cd dpdk-22.07
```

```
meson  build
cd build
ninja
ninja install
```

执行完之后所有的库都安装在 /usr/local/lib/x86_64-linux-gnu/ 目录。
可执行程序和脚本都安装在 /usr/local/bin/ 目录。
要卸载只需执行 ninja uninstall 即可。

### 编译裁剪
默认编译时，会默认编译不需要的驱动，导致编译很慢。由于我们是在虚拟机中，很多驱动都可以不需要，因此可以做裁剪。具体在执行 meson build 之前，
修改 `drivers/meson.build` 文件，`subdirs` 里只保留 ‘bus’，‘mempool’，‘net’ 即可。

修改 `drivers/bus/meson.build` 文件，`drivers` 里只保留 ‘pci’，‘vdev’ 即可。

修改 `drivers/mempool/meson.build` 文件，`drivers` 里只保留 ‘ring’，‘stack’ 即可。

修改 `drivers/net/meson.build` 文件，drivers 里只保留 ‘e1000’ 即可【因为虚拟机网卡使用的是 e1000 驱动】。

接着执行 meson build
之后编译就很快了。

### 编译 igb uio 驱动
#### 下载源码
```
git clone http://dpdk.org/git/dpdk-kmods
```
需要修改dpdk-kmods/linux/igb_uio/igb_uio.c中的代码，跳过DPDK PCI检查
line 275
```c
// Original
pci_intx_mask_supported(udev->pdev)
// Change to
pci_intx_mask_supported(udev->pdev) || true)
```
#### 编译
```
cd dpdk-kmods/linux/igb_uio
make
```

验证环境是否OK
为了验证我们的环境是没问题的，先编译出 l2fwd 程序：

cd examples/l2fwd
make

### 加载 igb_uio 驱动：
```
cd dpdk-kmods/linux/igb_uio
modprobe uio
insmod igb_uio.ko intr_mode=legacy
```
注意： 加载驱动时要带着参数intr_mode=legacy，如果不加参数，将会有问题！

### 分配一些大页内存【这里是1G】：
```
echo 512 > /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages
```

### 绑定两个网卡【虚拟机有 4 个网卡，最后两个网卡供 DPDK 使用】：
```
ifconfig ens33 down
ifconfig ens34 down
dpdk-devbind.py -b igb_uio ens33 ens34
```

### 运行 l2fwd 程序：
```
dpdk-stable-22.11.4/build$ sudo meson configure -Dexamples=helloworld 
dpdk-stable-22.11.4/build$ sudo ninja
dpdk-stable-22.11.4/build$ cd examples
dpdk-stable-22.11.4/build/examples$ ./dpdk-helloworld 
././dpdk-l2fwd -l 0-1 -- -p 0x3 -T 1
```
如果看到以下信息，说明 DPDK 环境没问题！
EAL: Error enabling interrupts for fd 20 (Input/output error) 等错误可以忽略。

### 解绑网卡
两个网卡 ens34 和 ens35 被 DPDK 占用后，ifconfig 里是没有的，要恢复请进行如下操作。

首先查看两个网卡 pci 设备号：
```
root@ubuntu2204:~# lspci | grep Eth
02:00.0 Ethernet controller: Intel Corporation 82545EM Gigabit Ethernet Controller (Copper) (rev 01)
02:01.0 Ethernet controller: Intel Corporation 82545EM Gigabit Ethernet Controller (Copper) (rev 01)
02:02.0 Ethernet controller: Intel Corporation 82545EM Gigabit Ethernet Controller (Copper) (rev 01)
02:03.0 Ethernet controller: Intel Corporation 82545EM Gigabit Ethernet Controller (Copper) (rev 01)
```
可看到最后两个网卡设备号是 02:02.0 和 02.03.0。
然后解绑两个网卡的 igb_uio 驱动，绑定 e1000 驱动：
```
dpdk-devbind.py -u 02:02.0 02:03.0
dpdk-devbind.py -b e1000  02:02.0 02:03.0
```
最后将网卡up起来：
```
ifconfig ens34 up
ifconfig ens35 up
```

### 编译加载kni
进入 DPDK 源码目录，并创建 `build` 目录（如果已经创建，可以跳过这一步）：
```
cd dpdk-22.11/build
meson setup build
```


进入构建目录，并使用 `meson configure` 启用内核模块支持,配置完成后，使用 `ninja` 进行编译，编译完成后，可以使用insmod以下命令加载 KNI 内核模块
```
dpdk-22.11/build# meson configure -Denable_kmods=true
dpdk-22.11/build# ninja
dpdk-22.11/build/kernel/linux/kni# insmod ./rte_kni.ko
```
验证 KNI 模块是否正确加载
```
lsmod | grep kni
```



原文链接：https://blog.csdn.net/woay2008/article/details/126899996