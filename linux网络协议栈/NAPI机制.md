# **linux 内核协议栈 NAPI机制**

（代码基于linux5.4.34）

## 一、NAPI的通俗解释

当你的计算机通过网络接口接收数据时，网卡（网络设备）会把数据包发到内存中，然后操作系统会处理这些数据。传统的方法是，每当有数据包到来时，网卡都会立刻通过**中断**通知 CPU 来处理。这种方法在网络流量较小时效果不错，但如果数据包很多，CPU 会频繁地被中断，导致处理效率降低，因为 CPU 需要频繁切换任务。

NAPI 是为了解决这个问题而设计的。它的基本原理是**减少中断的次数，改为轮询机制**。

### 1. 具体怎么做的？

1. **先用中断通知**：当网络设备接收到第一个数据包时，依然会发出中断，告诉 CPU 有数据要处理。
2. **进入轮询模式**：一旦中断触发，系统就会关闭网卡的中断，开始进入轮询模式。这时，系统会定期查看网卡有没有新的数据，而不是每次都靠中断通知。
3. **批量处理数据包**：在轮询模式下，系统会批量处理一批数据包，而不是一个一个处理。这减少了频繁的中断，同时能更高效地处理网络数据。
4. **网络流量恢复正常后，恢复中断**：如果流量减小，数据包处理完了，系统会再次开启中断模式，等待下次有数据包到来时再触发中断。

### 2. NAPI 的优点

- **减少中断开销**：当流量很大时，通过关闭中断，减少 CPU 因处理中断而导致的额外开销。
- **批量处理数据**：一次处理多个数据包，提高了效率。
- **自适应**：当流量较小时，NAPI 依然可以回到传统的中断模式，确保及时处理每一个数据包。

### 3. 类比

想象你是个店员，负责处理客户订单。以前每次来一位顾客，你都得停下手上的事，立刻服务这个顾客（类似中断）。但如果顾客太多，你就忙不过来了。

NAPI 就好比你先接待一个顾客，然后告诉后面来的顾客稍等一下。你会在一个时间段内批量处理所有等待的顾客（轮询），这样可以更有效率地完成工作。

## 二、net_rx_action 代码详解

`net_rx_action()` 是 Linux 内核中用于处理网络接收软中断的函数，它处理从网络接口接收的数据包并调用相应的处理函数。这段代码实现了 NAPI（New API）机制的一部分，与我们之前讨论的 NAPI 轮询和批量处理相对应。

以下是对这段代码的分段解释及其实现的功能：

### 1. **初始化部分**

```c
struct softnet_data *sd = this_cpu_ptr(&softnet_data);
unsigned long time_limit = jiffies + usecs_to_jiffies(netdev_budget_usecs);
int budget = netdev_budget;
LIST_HEAD(list);
LIST_HEAD(repoll);
```

- `sd`：获取当前 CPU 上的 `softnet_data` 结构，保存网络接收相关的状态信息。
- `time_limit`：设定处理时间的上限，防止处理时间过长。这个时间限制是基于 `netdev_budget_usecs`。
- `budget`：设定一个处理包数量的预算，防止处理过多数据包。
- `list` 和 `repoll`：分别是用于保存要处理的 `napi_struct` 列表和需要重新调度的 `napi_struct` 列表。

### 2. **中断与队列的处理**

```c
local_irq_disable();
list_splice_init(&sd->poll_list, &list);
local_irq_enable();
```

- **关闭中断**：为了安全地操作 `sd->poll_list`，暂时关闭中断。
- **`list_splice_init()`**：将 `sd->poll_list` 中的所有 `napi_struct` 移到本地的 `list` 中，以便轮询处理。
- **重新启用中断**：在列表操作完成后重新启用中断。

这部分的操作确保处理中的 `napi_struct` 列表是安全的。

### 3. **进入轮询处理**

```c
for (;;) {
    struct napi_struct *n;

    if (list_empty(&list)) {
        if (!sd_has_rps_ipi_waiting(sd) && list_empty(&repoll))
            goto out;
        break;
    }

    n = list_first_entry(&list, struct napi_struct, poll_list);
    budget -= napi_poll(n, &repoll);

    if (unlikely(budget <= 0 || time_after_eq(jiffies, time_limit))) {
        sd->time_squeeze++;
        break;
    }
}
```

- **`for (;;) {}`**：进入一个无限循环，直到处理完所有的 NAPI 轮询任务或预算、时间耗尽。
- **`list_empty(&list)`**：检查 `list` 是否为空，如果没有任务且没有需要重新轮询的 `napi_struct`，则跳出循环，进入 `out` 处理。
- **`list_first_entry()`**：从 `list` 中获取第一个 `napi_struct` 来处理。
- **`napi_poll(n, &repoll)`**：调用 `napi_poll()`，该函数负责处理实际的网络数据包。处理过程中可能会耗尽预算或时间限制。
- **预算和时间检查**：如果预算用尽（`budget <= 0`）或时间超过设定的 `time_limit`，则退出循环。

这一部分对应于 NAPI 的轮询过程，它会批量处理网络数据包，直到达到处理预算或者时间限制。

### 4. **重新处理剩余任务**

```c
local_irq_disable();
list_splice_tail_init(&sd->poll_list, &list);
list_splice_tail(&repoll, &list);
list_splice(&list, &sd->poll_list);
if (!list_empty(&sd->poll_list))
    __raise_softirq_irqoff(NET_RX_SOFTIRQ);
```

- **关闭中断**：再一次关闭中断，以安全地操作 `sd->poll_list` 列表。
- **`list_splice_tail_init()`**：将 `sd->poll_list` 重新与当前的 `list` 拼接，准备后续的处理。
- **`list_splice_tail(&repoll, &list)`**：将 `repoll` 列表中的 `napi_struct` 加入 `list`，准备再次轮询。
- **`list_splice(&list, &sd->poll_list)`**：将拼接完成的 `list` 加回 `sd->poll_list` 中。
- **`__raise_softirq_irqoff(NET_RX_SOFTIRQ)`**：如果仍然有未处理的任务，则再次触发软中断，继续处理剩下的任务。

这部分确保如果任务没有完全处理完，它会继续调度 `napi_struct` 来进行下一轮处理。

### 5. **完成与资源清理**

```c
net_rps_action_and_irq_enable(sd);
out:
    __kfree_skb_flush();
```

- **`net_rps_action_and_irq_enable(sd)`**：执行一些与 RPS（Receive Packet Steering）相关的操作，并重新启用中断。
- **`__kfree_skb_flush()`**：释放缓存的 `skb`（socket buffer）。

这一部分是 NAPI 处理的最后步骤，确保处理完成后系统可以重新接收新的数据包。

### 总结

这段代码实现了网络接收中的软中断（`NET_RX_SOFTIRQ`）处理：

1. **初始化并准备轮询**：从 `softnet_data` 中获取待处理的任务，并准备执行。
2. **轮询并处理任务**：通过 `napi_poll()` 批量处理数据包，处理过程中检测时间和预算。
3. **重新安排任务**：如果有未处理完的任务，将它们重新加入队列，并触发下一轮软中断处理。
4. **清理资源并完成**：释放剩余的 `skb` 并重新启用中断。

## 三、napi_poll代码详解

`napi_poll()` 是 NAPI 机制中负责处理数据包的核心函数。通俗地来说，它的任务是把接收到的数据包从网络设备搬运到内核中，交给更高层的协议栈（如 IP 层）进行处理，同时避免频繁的中断。

以下是如何通过 `napi_poll()` 将数据包从网卡缓冲区处理到内核协议栈的步骤，逐步解释其中的过程：

### 1. **移除 `poll_list`：**

```c
list_del_init(&n->poll_list);
```

- 这一步从 `napi_struct` 的 `poll_list` 中移除当前的 `napi_struct` 实例，表明它现在开始处理数据，不再处于等待队列中。

### 2. **加锁：**

```c
have = netpoll_poll_lock(n);
```

- 获取锁，以确保在处理期间其他代码不会并发地修改或处理相同的 `napi_struct`。这是为了避免多个进程或线程同时访问同一个 `napi` 对象引发的竞争条件。

### 3. **调用驱动的 `poll()` 函数处理数据包：**

```c
if (test_bit(NAPI_STATE_SCHED, &n->state)) {
    work = n->poll(n, weight);
    trace_napi_poll(n, work, weight);
}
```

- `test_bit(NAPI_STATE_SCHED, &n->state)`：这一步检查 NAPI 是否被调度，以确保只有被正确调度的 `napi` 实例才会处理数据包。
- `n->poll(n, weight)`：这是核心部分。`n->poll` 是一个回调函数，由具体的网卡驱动实现。在这里，网卡驱动会将数据从网卡的接收队列中提取出来，搬到内核的缓冲区。这可能是以太网帧或其他网络协议帧。
    - **数据的搬运**：网卡中的数据包会被从硬件接收缓冲区移动到内核中的 `skb`（socket buffer）结构。`skb` 是 Linux 用来存储网络数据包的一个标准数据结构。
    - **工作量计算**：`work` 代表实际处理了多少个数据包，通常是 `weight` 的一部分。`weight` 是处理的最大数据包数量，防止一次性处理太多数据导致系统其他部分得不到处理时间。

### 4. **检查是否处理完所有包：**

```c
if (likely(work < weight))
    goto out_unlock;
```

- 如果处理的数据包数量小于 `weight`，说明这次的包已经处理完了，没有剩余的包需要处理，直接退出。如果包还没有处理完，就继续处理剩余的包。

### 5. **特殊情况处理：**

```c
if (unlikely(napi_disable_pending(n))) {
    napi_complete(n);
    goto out_unlock;
}
```

- 检查 `napi` 是否已经禁用（`napi_disable_pending()`），如果禁用了，就完成处理并退出，不再继续。

### 6. **处理 GRO 数据包：**

```c
if (n->gro_bitmask) {
    napi_gro_flush(n, HZ >= 1000);
}
gro_normal_list(n);
```

- 如果使用了 GRO（Generic Receive Offload），会调用 `napi_gro_flush()` 将 GRO 缓冲区中累积的数据包进行处理。GRO 是一种批量处理数据包的优化，可以将多个小数据包合并成一个大数据包，提高网络吞吐量。

### 7. **如果没有处理完，重新加入 `repoll` 列表：**

```c
list_add_tail(&n->poll_list, repoll);
```

- 如果没有处理完所有的数据包，`napi_struct` 会被重新加入到 `repoll` 列表中，以便稍后继续处理。

### 8. **解锁并返回：**

```c
netpoll_poll_unlock(have);
return work;
```

- 最后释放锁，返回处理的包的数量。

### 总结：

1. **数据包的搬运**：`napi_poll()` 从网络设备的硬件缓冲区中提取数据包，放入内核的 `skb` 结构中，准备让上层协议栈（如 IP 层、TCP 层）处理。
2. **批量处理**：通过 NAPI，系统避免每次有包到达都触发中断，而是批量处理数据包，这样可以减少 CPU 中断开销，提升系统性能。
3. **GRO 优化**：合并多个小数据包，减少上层处理次数，提高处理效率。

通过这些步骤，`napi_poll()` 实现了高效的网络数据包接收机制，避免了频繁的中断，同时批量处理数据包来提升性能。