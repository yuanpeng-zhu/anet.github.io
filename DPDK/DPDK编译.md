### 准备工作
#### 加载 igb_uio 驱动：
```
cd dpdk-kmods/linux/igb_uio
modprobe uio
insmod igb_uio.ko intr_mode=legacy
```
注意： 加载驱动时要带着参数intr_mode=legacy，如果不加参数，将会有问题！

#### 分配一些大页内存【这里是1G】：
```
echo 512 > /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages
```

#### 绑定两个网卡【虚拟机有 4 个网卡，最后两个网卡供 DPDK 使用】：
```
ifconfig ens33 down
ifconfig ens34 down
dpdk-devbind.py -b igb_uio ens33 ens34
```


### 1. 加入源码文件

### 2. 配置`meson.build`
示例：
```
project('l3fwd', 'c')

# 启用 DPDK 的实验性 API（如果需要）
allow_experimental_apis = true

# 查找 DPDK 依赖
dpdk = dependency('libdpdk', version : '>=22.11')

# 定义源文件列表
sources = files(
    'l3fwd.c',
)

# 创建可执行文件
executable('l3fwd_app', sources,
    dependencies: dpdk,
    install: true)
```

### 3. export路径
```
export PKG_CONFIG_PATH=/usr/local/lib64/pkgconfig/
```
可通过`find /usr/local/ -name 'libdpdk.pc'寻找路径


### 4. 编译并运行
```
meson setup build 
ninja -C build
```


### lib not found
```
root@ubuntu:/home/zyp/Desktop/l3fwd# ./build/l3fwd_app
./build/l3fwd_app: error while loading shared libraries: librte_hash.so.23: cannot open shared object file: No such file or directory
root@ubuntu:/home/zyp/Desktop/l3fwd# find / -name "librte_hash.so.23"
/usr/local/lib64/librte_hash.so.23
/home/zyp/Desktop/dpdk-22.11/build/lib/librte_hash.so.23
find: ‘/run/user/1000/doc’: Permission denied
find: ‘/run/user/1000/gvfs’: Permission denied
root@ubuntu:/home/zyp/Desktop/l3fwd# export LD_LIBRARY_PATH=/usr/local/lib64/:$LD_LIBRARY_PATH
```