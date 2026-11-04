# 用 eBPF 抓 TCP 重传，卡在 ring buffer 了

起因是线上有个服务偶尔超时，tcpdump 抓了几十兆什么都看不出来。想着干脆上个 eBPF，只抓重传事件，省事。

## 能跑通的第一步

`tcp:tcp_retransmit_skb` 这个 tracepoint 是现成的，bpftrace 一行就能看：

```bash
sudo bpftrace -e '
tracepoint:tcp:tcp_retransmit_skb
{
  printf("%s:%d -> %s:%d retrans\n",
    str(args->saddr), args->sport, str(args->daddr), args->dport);
}
'
```

跑起来确实能看到重传，但只有五元组，没有重传次数、没有 RTT，也不知道是第几次重传。要判定是不是真的网络问题还差得远。

## 改用 libbpf 自己写

想法很简单：挂 kprobe 到 `tcp_retransmit_skb()`，用 BPF map 按四元组累计次数，用户态每秒读一次。

先用了 hash map，能跑，但用户态读的时候要遍历整个 map，连接一多就很慢。

于是换 ringbuf：

```c
struct {
  __uint(type, BPF_MAP_TYPE_RINGBUF);
  __uint(max_entries, 1 << 24);
} events SEC(".maps");
```

提交侧：

```c
struct event *e = bpf_ringbuf_reserve(&events, sizeof(*e), 0);
if (!e) return 0;
e->sport = sport;
e->dport = dport;
e->retrans = 1;
bpf_ringbuf_submit(e, 0);
```

## 卡住的地方

用户态 `ring_buffer__new()` 之后 poll 一直返回 0，事件一条都收不到。

`bpftool map dump` 看 map 是有数据的，说明内核侧提交了，用户态没读出来：

```
$ sudo bpftool map dump id 47 | head
key: 01 00 00 00 ...  value: 03 00 00 00 00 00 00 00
```

试过几个方向，都没解决：

- 换成 `ring_buffer__poll(rb, 100)` 显式超时，还是 0
- 确认编译带了 `-g`，BTF 在，`bpftool btf dump` 能看到自己的 struct
- 怀疑是 reserve 的大小对齐问题，把 sizeof 打印出来是 48 字节，对齐没问题
- 换回 perf buffer 能收到事件，说明 kprobe 挂对了，问题只出在 ringbuf 这一侧

现在的结论是：内核 5.10，理论上 ringbuf 5.8 就支持，但 reserve/submit 在某些 backport 内核上行为和上游不一致。下一步准备看 `bpf_ringbuf_query()` 的返回值，确认消费者位置有没有推进。

暂时还是 perf buffer 顶着的，虽然性能差一点，但至少能出数据。

## 还没做的

- 加上 cwnd 和 snd_cwnd 的采样，看是不是拥塞窗口掉到底
- 按进程聚合，而不是只看五元组
- 接个 Prometheus exporter，出个看板
