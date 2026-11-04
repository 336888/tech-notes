# 半夜的 502：一次 nginx upstream 排查

凌晨两点被告警叫起来，一个接口 502，但只有零星几笔，不是全挂。

## 先看日志

```
2026-10-28 02:14:03 [error] 1841#0: *88213 upstream prematurely closed connection while reading response header from upstream,
request: "POST /api/order/callback HTTP/1.1", upstream: "http://127.0.0.1:9000/api/order/callback"
```

关键词是 `upstream prematurely closed connection`，不是 timeout，是后端主动断的。

## 排除了几个常见方向

- `client_max_body_size`：回调 body 只有几百字节，排除
- `proxy_read_timeout`：默认 60s，接口正常 200ms 返回，排除
- 后端进程数：`ps -ef | grep node` 看到 4 个 worker 都在，没崩

## 真正的线索在应用日志

翻后端错误日志，同一时刻有：

```
Error: aborted
    at IncomingMessage.onAborted (node:internal/http:1:1)
```

`aborted` 说明是客户端先断的。但 nginx 说是 upstream 先断，两边互相甩锅。

后来想明白了：支付网关的回调超时是 **3 秒**。我们的处理逻辑里有一步要查订单状态，那个查询偶尔要 4-5 秒（慢查询），于是网关先超时断开，nginx 收到 RST，就把这个连接标记成 upstream 异常，日志写成 prematurely closed。

所以根因不在 nginx，是回调接口太慢，把支付方的耐心耗尽了。

## 改法

1. 回调改成「先落库、立即返回 200，异步处理」
2. 慢查询加索引，从 4.5s 降到 80ms
3. nginx 加 `proxy_ignore_client_abort on;`，至少日志不再误导人

改完之后一周没再出现。

## 留下的教训

- `prematurely closed connection` 不一定是后端崩，也可能是**对端**等不及了
- 两边日志要对着时间戳看，单看一边永远在互相甩锅
- 第三方回调的超时时间一定要写进文档，不然排查全靠猜

## 一个没搞明白的

`proxy_ignore_client_abort` 加上之后，客户端断开时 nginx 还会继续等后端返回，这对长任务友好，但会占着 worker 连接。我们量小无所谓，量大了要不要开，还没想清楚。
