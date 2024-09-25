
## 一、三次握手的过程

![alt text](<Pasted image 20240925093358.png>)

第一次握手：
客户端将TCP报文标志位SYN置为1，随机产生一个序号值seq=J，保存在TCP首部的序列号(Sequence Number)字段里，指明客户端打算连接的服务器的端口，并将该数据包发送给服务器端，发送完毕后，客户端进入SYN_SENT状态，等待服务器端确认。

第二次握手：
服务器端收到数据包后由标志位SYN=1知道客户端请求建立连接，服务器端将TCP报文标志位SYN和ACK都置为1，ack=J+1，随机产生一个序号值seq=K，并将该数据包发送给客户端以确认连接请求，服务器端进入SYN_RCVD状态。

第三次握手：
客户端收到确认后，检查ack是否为J+1，ACK是否为1，如果正确则将标志位ACK置为1，ack=K+1，并将该数据包发送给服务器端，服务器端检查ack是否为K+1，ACK是否为1，如果正确则连接建立成功，客户端和服务器端进入ESTABLISHED状态，完成三次握手，随后客户端与服务器端之间可以开始传输数据了。

## 二、三次握手的实现

### 2.1 源码
现有cpp实现的`server.c` 以及 `client.c` 的源码如下：
`server.c` :
```cpp
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>
 
#define PORT 8080
#define BUF_SIZE 1024

int main() {
    int server_fd, new_socket;
    struct sockaddr_in address;
    int addrlen = sizeof(address);
    char buffer[BUF_SIZE] = {0};
    
    // 创建套接字 (TCP Socket)
    if ((server_fd = socket(AF_INET, SOCK_STREAM, 0)) == 0) {
        perror("socket failed");
        exit(EXIT_FAILURE);
    }

    // 配置服务器地址
    address.sin_family = AF_INET; // IPv4
    address.sin_addr.s_addr = INADDR_ANY; // 接受任何 IP 地址
    address.sin_port = htons(PORT); // 端口号

    // 绑定 Socket 到指定端口
    if (bind(server_fd, (struct sockaddr *)&address, sizeof(address)) < 0) {
        perror("bind failed");
        close(server_fd);
        exit(EXIT_FAILURE);
    }

    // 监听传入连接
    if (listen(server_fd, 3) < 0) {
        perror("listen failed");
        close(server_fd);
        exit(EXIT_FAILURE);
    }

    printf("Server is listening on port %d...\n", PORT);

    // 接受客户端连接
    if ((new_socket = accept(server_fd, (struct sockaddr *)&address, (socklen_t*)&addrlen)) < 0) {
        perror("accept failed");
        close(server_fd);
        exit(EXIT_FAILURE);
    }

    // 读取客户端数据
    int valread = read(new_socket, buffer, BUF_SIZE);
    printf("Received from client: %s\n", buffer);

    // 发送响应给客户端
    char *response = "Message received by server";
    send(new_socket, response, strlen(response), 0);
    printf("Response sent to client\n");

    // 关闭套接字
    close(new_socket);
    close(server_fd);
    return 0;
}
```

 `client.c`:
```cpp
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>

#define PORT 8080
#define BUF_SIZE 1024

int main() {
    int sock = 0;
    struct sockaddr_in serv_addr;
    char *message = "Hello from client!";
    char buffer[BUF_SIZE] = {0};

    // 创建套接字 (TCP Socket)
    if ((sock = socket(AF_INET, SOCK_STREAM, 0)) < 0) {
        printf("\n Socket creation error \n");
        return -1;
    }

    // 配置服务器地址
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_port = htons(PORT);

    // 将地址转换为二进制并设置为 localhost (127.0.0.1)
    if (inet_pton(AF_INET, "127.0.0.1", &serv_addr.sin_addr) <= 0) {
        printf("\nInvalid address/ Address not supported \n");
        return -1;
    }

    // 连接到服务器
    if (connect(sock, (struct sockaddr *)&serv_addr, sizeof(serv_addr)) < 0) {
        printf("\nConnection Failed \n");
        return -1;
    }

    // 发送消息到服务器
    send(sock, message, strlen(message), 0);
    printf("Message sent to server: %s\n", message);

    // 读取服务器的响应
    int valread = read(sock, buffer, BUF_SIZE);
    printf("Server response: %s\n", buffer);

    // 关闭套接字
    close(sock);
    return 0;
}

```


### 2.2 连接过程分析

#### a. 创建套接字
不论是客户端还是服务端，都首先通过`server_fd = socket(AF_INET, SOCK_STREAM, 0))`创建了套接字描述符。
```c
int socket(int domain, int type, int protocol)
```
参数设置如下：
- domain：参数表示套接字要使用的协议簇，协议簇的在“linux/socket.h”里有详细定义，常用的协议簇：
	- AF_UNIX（本机通信）
	- AF_INET（TCP/IP – IPv4）
	- AF_INET6（TCP/IP – IPv6）
- type：“type”参数指的是套接字类型，常用的类型有：
	- SOCK_STREAM（TCP流）
	- SOCK_DGRAM（UDP数据报）
	- SOCK_RAW（原始套接字）
- protocol：协议，通常设为 0 以选择默认协议，或者可以指定具体协议（如 `IPPROTO_TCP` 或 `IPPROTO_UDP`）。

#### b. 服务端：绑定指定端口
通过`bind(server_fd, (struct sockaddr *)&address, sizeof(address))`，绑定指定`ip`、端口号的端口。
```c
int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
```
参数设置如下：
- sockfd：即socket描述字，它是通过socket()函数创建了，唯一标识一个socket。bind()函数就是将给这个描述字绑定一个名字。
- addr：一个const struct sockaddr *指针，指向要绑定给sockfd的协议地址。如ipv4对应的是：
```c
struct sockaddr_in {
    sa_family_t    sin_family; /* address family: AF_INET */
    in_port_t      sin_port;   /* port in network byte order */
    struct in_addr sin_addr;   /* internet address */
};
```
- addrlen：对应的是地址的长度。
#### c. 服务端：监听端口
```c
int listen(int sockfd, int backlog);
```
listen函数的第一个参数即为要监听的socket描述字，第二个参数为相应socket可以排队的最大连接个数。socket()函数创建的socket默认是一个主动类型的，listen函数将socket变为被动类型的，等待客户的连接请求。

#### d. 客户端：连接
```c
int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
```
[connect函数]的第一个参数即为客户端的socket描述字，第二参数为服务器的socket地址，第三个参数为socket地址的长度。客户端通过调用connect函数来建立与TCP服务器的连接。

#### e. 服务端：接受请求
```c
int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
```
accept函数的第一个参数为服务器的socket描述字，第二个参数为指向struct sockaddr *的指针，用于返回客户端的协议地址，第三个参数为协议地址的长度。如果accpet成功，那么其返回值是由内核自动生成的一个全新的描述字，代表与返回客户的TCP连接。


### 三、三次握手的底层实现

#### 3.1 `socket`
调用系统调用
```
SYSCALL_DEFINE3(socket, int, family, int, type, int, protocol)
{
    return __sys_socket(family, type, protocol);
}
```
```c
int __sock_create(struct net *net, int family, int type, int protocol,
             struct socket **res, int kern)

{
    ......
    sock = sock_alloc();
	......
    err = pf->create(net, sock, protocol, kern);
    ......
}
```
在`sock_alloc`函数中，申请了⼀个`struct socket_alloc`内核对象。
```
struct socket_alloc {
    struct socket socket;
    struct inode vfs_inode;
};
```
`socket_alloc`内核对象将`socket`和`inode`信息关联了起来。
`sock_alloc`最终调用`socket_alloc_inode`申请`socket_alloc`对象。

#### 3.2 `bind`
```c
SYSCALL_DEFINE3(bind, int, fd, struct sockaddr __user *, umyaddr, int, addrlen)
{
	return __sys_bind(fd, umyaddr, addrlen);
}
```

```c
int __sys_bind(int fd, struct sockaddr __user *umyaddr, int addrlen)
{
	struct socket *sock;
	struct sockaddr_storage address;
	int err, fput_needed;

        // 这里通过入参的fd寻找到socket结构体数据
	sock = sockfd_lookup_light(fd, &err, &fput_needed);
	if (sock) {
    // 把addr数据从用户空间copy到内核空间，转变为struct sockaddr_storage结构体
		err = move_addr_to_kernel(umyaddr, addrlen, &address);
		if (err >= 0) {
            // SELInux相关的处理
			err = security_socket_bind(sock,
						   (struct sockaddr *)&address,
						   addrlen);
			if (!err)
                // 如果是SOCK_STREAM的话调用的是inet_bind()
				err = sock->ops->bind(sock,
						      (struct sockaddr *)
						      &address, addrlen);
		}
		fput_light(sock->file, fput_needed);
	}
	return err;
}
```
bind实现功能如下:
1.通过fd找到对应的socket实例
2.进行(地址+端口)的参数校验
3.对socket实例中的成员赋值

#### 3.3 `listen`
此时，服务端正在等待客户端的请求连接。
`listen`调用系统调用进入内核态，系统调用如下：
```c
SYSCALL_DEFINE2(listen, int, fd, int, backlog)
{
    return __sys_listen(fd, backlog);
}

int __sys_listen(int fd, int backlog)
{
    struct socket *sock;
    int err, fput_needed;
    int somaxconn;
    sock = sockfd_lookup_light(fd, &err, &fput_needed);
    if (sock) {
        somaxconn = sock_net(sock->sk)->core.sysctl_somaxconn;
        if ((unsigned int)backlog > somaxconn)
            backlog = somaxconn;
        err = security_socket_listen(sock, backlog);
        if (!err)
            err = sock->ops->listen(sock, backlog);
        fput_light(sock->file, fput_needed);
    }
    return err;
}
```
`sock_net(sock->sk)->core.sysctl_somaxconn`为内核参数，限制半连接队列的长度。
实际上，半连接队列长度与`sysctl_somaxconn`与 `backlog`有关。
`sock->ops->listen(sock, backlog)`指向的对象为`inet_listen`函数。
```c
int inet_listen(struct socket *sock, int backlog)
{
    ......
    if (old_state != TCP_LISTEN) {
        ......
        err = inet_csk_listen_start(sk, backlog);
        ......
    }
    ......
}
```
核心函数：`inet_csk_listen_start(sk, backlog);`，启动socket监听。
```c
int inet_csk_listen_start(struct sock *sk, int backlog)
{
    ......
    reqsk_queue_alloc(&icsk->icsk_accept_queue);
    ......
    return err;

}
```
**重点来了**：此处为接收队列申请了对象（`inet_connection_sock`）并进行了初始化。
`inet_connection_sock`中包含`struct request_sock_queue icsk_accept_queue;`。
`icsk_accept_queue`中保存了全连接以及半连接队列。
```
struct request_sock_queue {
    ......
    struct request_sock *rskq_accept_head;
    struct request_sock *rskq_accept_tail;
    truct listen_sock *listen opt;
};
```
![alt text](<Pasted image 20240925104149.png>)
`listen`最主要的⼯作就是申请和初始化接收队列，包括全连接队列和半连接队列。其中全连接队列是⼀个链表，⽽半连接队列由于需要快速地查找，所以使⽤的是⼀个哈希表（其实半连接队列更准确的叫法应该叫半连接哈希表）。
半连接队列长度为`sysctl_somaxconn`与 `backlog`最小值+1向上取整到2的N次幂。
半连接队列长度为`sysctl_somaxconn`与 `backlog`最小值。
#### 3.4 `connect`
客户端发起握手请求
`connect`调用系统调用
```c
SYSCALL_DEFINE3(connect, int, fd, struct sockaddr __user *, uservaddr,
        int, addrlen)
{
    return __sys_connect(fd, uservaddr, addrlen);
}
```
```c
int __sys_connect(int fd, struct sockaddr __user *uservaddr, int addrlen)
{
    struct socket *sock;
    sock = sockfd_lookup_light(fd, &err, &fput_needed);
    ......
    err = sock->ops->connect(sock, (struct sockaddr *)&address, addrlen,
                 sock->file->f_flags);
	......
}
```
`sock->ops->connect`对应`inet_stream_connect`函数
`inet_stream_connect` 根据当前socket对象选择`sk->sk_prot->connect`并进入，进行处理
`sk->sk_prot->connect`指向`tcp_v4_connect`
`tcp_v4_connect`选择端口并发送SYN报文。
`__inet_hash_connect`用于选择可用端口，选择规则：源ip：目的ip：源端口：目的端口四元组不重复。
SYN会通过`tcp_connect`-> `tcp_transmit_skb`发送出去。

#### 3.5 `accept`
服务端接收ACK报文,第三次握手。
```c
int tcp_v4_do_rcv(struct sock *sk, struct sk_buff *skb)
{
    struct sock *rsk;
    if (sk->sk_state == TCP_ESTABLISHED) { /* Fast path */
        ......
    }
    if (tcp_checksum_complete(skb))
        goto csum_err;
    if (sk->sk_state == TCP_LISTEN) {
        struct sock *nsk = tcp_v4_cookie_check(sk, skb);
        if (!nsk)
            goto discard;
        if (nsk != sk) {
            if (tcp_child_process(sk, nsk, skb)) {
                rsk = nsk;
                goto reset;
            }
            return 0;
        }
    } else
        ......
    }
}
```
`tcp_v4_do_rcv`根据`sk->sk_state`分别进行不通的处理。
`tcp_v4_cookie_check`寻找半连接对象，将该对象从半连接队列中清除，创建新的socket并将其加入全连接队列。

`accept`调用系统调用
```c
SYSCALL_DEFINE3(accept, int, fd, struct sockaddr __user *, upeer_sockaddr,
        int __user *, upeer_addrlen)
{
    return __sys_accept4(fd, upeer_sockaddr, upeer_addrlen, 0);
}
```
`__sys_accept4`调用`sock->ops->accept`
最终调用`inet_csk_accept`,从已经建立好的全连接队列中取出一个并返回。


## 参考
深入理解linux网络
[传输控制协议 - 维基百科，自由的百科全书 (wikipedia.org)](https://zh.wikipedia.org/wiki/%E4%BC%A0%E8%BE%93%E6%8E%A7%E5%88%B6%E5%8D%8F%E8%AE%AE)
[一文彻底搞懂 TCP三次握手、四次挥手过程及原理 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/108504297)
