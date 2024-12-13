## DPDK+TUN 将报文交付内核协议栈处理

## 实验环境搭建
![alt text](../picture/tun{E3451474-1D68-44C2-AC5D-28734F40927F}.png)
其中，node1与node2为wmware workstation16上安装的ubuntu20的虚拟机。
网卡配置如上图所示。

## 实验目标
node2可以ping通node1的TUN网卡。
node2可以与node1绑定TUN网卡的应用通过tcp进行通信。

## 初始代码如下
[ dpdk源码 ](dpdk_sourcecode.md)
其中注意，tun是三层虚拟网卡，接受的与发送的是完整的ip数据报文。
dpdk从网卡获取的是完整的mac报文，接收并发到tun必须将mac的头部去掉，同样从tun中读取并从网卡发送时，需要将mac头部加上。

## 测试调试与分析

### node2 ping node1不通
node2 ping node1的192.168.10.2 ip
1. node2 进行tcpdump抓包，未发现从ens35发送的icmp报文。
    查看node2的ip route，发现缺少发往192.168.10.0/24的路由
    添加路由：`sudo ip route add 192.168.10.0/24 via 192.168.1.129`
    查看node2的ip neigh：
    ![alt text](../picture/tun{69E0B243-C222-41C2-871B-5A1113869A05}.png)
    发现发往192.168.1.129的arp表项失效了，手动更新新的表项：
    ```
    sudo ip neigh del 192.168.1.129 dev ens35
    sudo ip neigh add 192.168.1.129 dev ens35 lladdr 00:0C:29:4A:36:96
    ```
    目前路由表与arp表项均已经正常。
    ![alt text](../picture/tun{40B9FA08-4860-4D52-AE4E-A00AD78FC7E3}.png)
    ![alt text](../picture/tun{7D89D346-DC80-422B-9A6C-0D31A90F04A5}.png)

2. node2 进行tcpdump抓包，发现从ens35发送的icmp报文只有request报文。
    查看dpdk程序输出，发现只有receive报文，没有send报文。
    ![alt text](../picture/tun{93AF37C5-4400-4733-98E2-7BFC63BF6297}.png)
    用tcpdump对node1的tun0进行抓包，发现仅有icmp request报文
    ![alt text](../picture/tun{FEA01A13-6ED0-4899-9D3B-9B947A54B91D}.png)
    此时说明，dpdk从网卡中获取到了icmp报文，通过write写入到了tun0设备。
    查找tun0的路由表，发现没有到192.168.1.0/24的路由表。
    添加路由表项：`sudo ip route add 192.168.1.0/24 dev tun0`
    为了方便，在`int create_tun_device(const char *dev_name)`添加如下代码，避免每一次运行dpdk程序都要重复添加路由表。
    `system("sudo ip route add 192.168.1.0/24 dev tun0");`

3. 此时发现node1 tun0发送了icmp reply报文
    ![alt text](../picture/tun{1839C287-9922-4E2A-BD4A-376485628C33}.png)
    但是reply报文时连续接收的。
    查看node2 ping的输出，发现延迟、丢包均非常严重
    ![alt text](../picture/tun{2FB34717-BCB6-4BB0-B4E1-54AACFBCC9C3}.png)
    node1 dpdk的程序输出显示，连续地接收了若干报文后，也连续地发送了若干报文。中间间隔时间非常大。
    上述现象说明，dpdk的while循环时执行了的，但是执行速度过慢。
    通过在while中加入计时器，发现一次循环cpu运行时间只有不到0.00001秒。但是每次循环实际时间大概3-5分钟。猜测执行到某些函数，dpdk主动放弃了运行，进入了等待状态，
    此时进一步猜测write、read为阻塞操作。
    为此，将dpdk接收、发送数据包从原来地一个while循环中，分别拆分到两个函数中。
    ```cpp
    rte_eal_remote_launch((lcore_function_t *)receive_packets, &args, 3);
    rte_eal_remote_launch((lcore_function_t *)send_packets, &args, 4);
    ```
    此时经过编译与运行，ping输出正常！
    ![alt text](../picture/tun{4434E8D5-D500-4E74-AD99-436F78157D7C}.png)
    在node1与node2进行tcp测试，结果正常
    ![alt text](../picture/tun{3C7C86AB-D5DB-4990-92DB-1FB868AF9B64}.png)

    终版源码如下
    [ dpdk源码 ](dpdk_sourcecode2.md)





