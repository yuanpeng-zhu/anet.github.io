```cpp
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/ioctl.h>
#include <linux/if.h>
#include <linux/if_tun.h>
#include <rte_eal.h>
#include <rte_ethdev.h>
#include <rte_mbuf.h>

#include <rte_ether.h>
#include <rte_ip.h>
#include <arpa/inet.h> 

#include <time.h>

#define NUM_MBUFS 8192
#define MBUF_CACHE_SIZE 250
#define BURST_SIZE 32
#define TUN_DEVICE "/dev/net/tun"

static const uint16_t RX_RING_SIZE = 128;
static const uint16_t TX_RING_SIZE = 512;
static struct rte_mempool *mbuf_pool;


struct packet_processing_args {
    uint16_t port_id;
    int tun_fd;
};


int create_tun_device(const char *dev_name) {
    struct ifreq ifr;
    int fd = open("/dev/net/tun", O_RDWR);
    if (fd < 0) {
        perror("Opening TUN device");
        exit(EXIT_FAILURE);
    }

    memset(&ifr, 0, sizeof(ifr));
    ifr.ifr_flags = IFF_TUN | IFF_NO_PI; // 配置为 TUN 设备，无附加协议头
    strncpy(ifr.ifr_name, dev_name, IFNAMSIZ); // 设置设备名

    if (ioctl(fd, TUNSETIFF, &ifr) < 0) {
        perror("Configuring TUN device");
        close(fd);
        exit(EXIT_FAILURE);
    }

    printf("Created TUN device: %s\n", dev_name);

    system("sudo ip addr add 192.168.10.2/24 dev tun0");
    system("sudo ip link set tun0 up");

    return fd;
}


void init_dpdk(int argc, char **argv, uint16_t port_id) {
    // 初始化 EAL
    if (rte_eal_init(argc, argv) < 0) {
        rte_exit(EXIT_FAILURE, "EAL initialization failed\n");
    }

    // 创建内存池
    mbuf_pool = rte_pktmbuf_pool_create("MBUF_POOL", NUM_MBUFS, MBUF_CACHE_SIZE, 0, RTE_MBUF_DEFAULT_BUF_SIZE, rte_socket_id());
    if (!mbuf_pool) {
        rte_exit(EXIT_FAILURE, "Cannot create mbuf pool\n");
    }

    // 配置以太网设备
    struct rte_eth_conf port_conf = {0};
    if (rte_eth_dev_configure(port_id, 1, 1, &port_conf) < 0) {
        rte_exit(EXIT_FAILURE, "Ethernet device configuration failed\n");
    }

    // 设置 RX 和 TX 队列
    if (rte_eth_rx_queue_setup(port_id, 0, RX_RING_SIZE, rte_eth_dev_socket_id(port_id), NULL, mbuf_pool) < 0) {
        rte_exit(EXIT_FAILURE, "RX queue setup failed\n");
    }
    if (rte_eth_tx_queue_setup(port_id, 0, TX_RING_SIZE, rte_eth_dev_socket_id(port_id), NULL) < 0) {
        rte_exit(EXIT_FAILURE, "TX queue setup failed\n");
    }

    // 启动设备
    if (rte_eth_dev_start(port_id) < 0) {
        rte_exit(EXIT_FAILURE, "Device start failed\n");
    }
    printf("Initialized DPDK on port %u\n", port_id);

    rte_eth_promiscuous_enable(port_id);
}


void parse_packet(struct rte_mbuf *pkt) {
    // 获取以太网帧头部
    struct rte_ether_hdr *eth_hdr;
    eth_hdr = rte_pktmbuf_mtod(pkt, struct rte_ether_hdr *);

    // 获取源 MAC 地址
    struct rte_ether_addr src_mac = eth_hdr->src_addr;
    printf("Source MAC: %02X:%02X:%02X:%02X:%02X:%02X\n",
           src_mac.addr_bytes[0], src_mac.addr_bytes[1], src_mac.addr_bytes[2],
           src_mac.addr_bytes[3], src_mac.addr_bytes[4], src_mac.addr_bytes[5]);

    // 获取目的 MAC 地址
    struct rte_ether_addr dst_mac = eth_hdr->dst_addr;
    printf("Destination MAC: %02X:%02X:%02X:%02X:%02X:%02X\n",
           dst_mac.addr_bytes[0], dst_mac.addr_bytes[1], dst_mac.addr_bytes[2],
           dst_mac.addr_bytes[3], dst_mac.addr_bytes[4], dst_mac.addr_bytes[5]);


    // 解析以太网类型 (IPv4 或 IPv6)
    if (eth_hdr->ether_type == rte_cpu_to_be_16(RTE_ETHER_TYPE_IPV4)) {
        // 解析 IPv4 头部
        struct rte_ipv4_hdr *ipv4_hdr;
        ipv4_hdr = (struct rte_ipv4_hdr *)(eth_hdr + 1);

        // 获取源 IP 地址
        char src_ip[INET_ADDRSTRLEN];
        inet_ntop(AF_INET, &ipv4_hdr->src_addr, src_ip, sizeof(src_ip));
        printf("Source IP: %s\n", src_ip);

        // 获取目的 IP 地址
        char dst_ip[INET_ADDRSTRLEN];
        inet_ntop(AF_INET, &ipv4_hdr->dst_addr, dst_ip, sizeof(dst_ip));
        printf("Destination IP: %s\n", dst_ip);
    } else if (eth_hdr->ether_type == rte_cpu_to_be_16(RTE_ETHER_TYPE_IPV6)) {
        // 解析 IPv6 头部（如需要）
        // 这里可以按照 IPv6 格式处理
    }
}

// void process_packets(uint16_t port_id, int tun_fd){
void process_packets(void *arg) {
    struct packet_processing_args *args = (struct packet_processing_args *)arg;
    uint16_t port_id = args->port_id;
    int tun_fd = args->tun_fd;

    struct rte_mbuf *bufs[BURST_SIZE];

    clock_t while_time1 = clock();
    clock_t while_time2;
    while (1) {
        // 从网卡接收数据包
        clock_t start_time = clock();
        uint16_t nb_rx = rte_eth_rx_burst(port_id, 0, bufs, BURST_SIZE);
        for (int i = 0; i < nb_rx; i++) {
            struct rte_mbuf *pkt = bufs[i];
            char *pkt_data = rte_pktmbuf_mtod(pkt, char *);
            uint16_t pkt_len = rte_pktmbuf_pkt_len(pkt);

            printf("-------------receive--------------------\n");
            parse_packet( pkt );

            // 以太网头部长度
            const size_t eth_hdr_len = sizeof(struct rte_ether_hdr);

            // 确保报文长度足够
            if (pkt_len > eth_hdr_len) {
                char *ip_packet = pkt_data + eth_hdr_len;
                size_t ip_len = pkt_len - eth_hdr_len;

                // 将 IP 报文写入 TUN 设备
                if (write(tun_fd, ip_packet, ip_len) < 0) {
                    perror("Write to TUN");
                }
            }

            rte_pktmbuf_free(pkt);
        }

        clock_t time1 = clock();

        // 从 TUN 读取数据
        char tun_buf[2048];
        ssize_t len = read(tun_fd, tun_buf, sizeof(tun_buf));
        if (len > 0) {
            struct rte_mbuf *tx_pkt = rte_pktmbuf_alloc(mbuf_pool);
            if (!tx_pkt) {
                fprintf(stderr, "Failed to allocate TX packet\n");
                continue;
            }

            // 获取缓冲区地址，并准备写入
            char *tx_data = rte_pktmbuf_mtod(tx_pkt, char *);

            // 构造以太网头部
            struct rte_ether_hdr *eth_hdr = (struct rte_ether_hdr *)tx_data;
            struct rte_ether_addr src_mac, dst_mac;

            // 获取网卡 MAC 地址
            if (rte_eth_macaddr_get(port_id, &src_mac) < 0) {
                fprintf(stderr, "Failed to get source MAC address\n");
                rte_pktmbuf_free(tx_pkt);
                continue;
            }

            // 设置目标 MAC 地址（根据具体需求设置）
            rte_ether_unformat_addr("00:0c:29:c5:a9:17", &dst_mac);

            // 填充以太网头部
            rte_ether_addr_copy(&src_mac, &eth_hdr->src_addr);
            rte_ether_addr_copy(&dst_mac, &eth_hdr->dst_addr);
            eth_hdr->ether_type = rte_cpu_to_be_16(RTE_ETHER_TYPE_IPV4);

            // 添加 IP 数据到以太网帧后
            memcpy(tx_data + sizeof(struct rte_ether_hdr), tun_buf, len);
            tx_pkt->data_len = len + sizeof(struct rte_ether_hdr);
            tx_pkt->pkt_len = len + sizeof(struct rte_ether_hdr);

            printf("-------------send--------------------\n");
            parse_packet( tx_pkt );

            // 发送到网卡
            if (rte_eth_tx_burst(port_id, 0, &tx_pkt, 1) < 1) {
                rte_pktmbuf_free(tx_pkt);
                fprintf(stderr, "TX failed\n");
            }
        }

        clock_t end_time = clock();

        double cpu_time_used1 = ((double) (time1 - start_time)) / CLOCKS_PER_SEC;
        double cpu_time_used2 = ((double) (end_time - time1)) / CLOCKS_PER_SEC;

        while_time2 = clock();
        double cpu_time_used3 = ((double) (while_time2 - while_time1)) / CLOCKS_PER_SEC;
        while_time1 = while_time2;

        printf("程序运行时间: %f 秒 %f 秒, 距离上次 %f 秒\n", cpu_time_used1, cpu_time_used2, cpu_time_used3);

    }
}





int main(int argc, char **argv) {
    const char *tun_name = "tun0";
    uint16_t port_id = 0;

    // 初始化 TUN 设备
    int tun_fd = create_tun_device(tun_name);

    // 初始化 DPDK
    init_dpdk(argc, argv, port_id);

    struct packet_processing_args args;
    args.port_id = port_id;
    args.tun_fd = tun_fd;
    rte_eal_remote_launch((lcore_function_t *)process_packets, &args, 3);

    // 处理数据包
    // process_packets(port_id, tun_fd);

    rte_eal_mp_wait_lcore();

    // 释放资源
    close(tun_fd);
    return 0;
}


```