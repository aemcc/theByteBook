# 2.2 HTTPS 请求优化分析

优化 HTTPS 请求之前，我们得先清楚一个 HTTPS 请求有哪些阶段，以及各个阶段延迟如何计算。

## 2.2.1 请求阶段分析

一个完整、未复用连接的 HTTPS 请求需要经过以下 5 个阶段：**DNS 域名解析、TCP 握手、SSL 握手、服务器处理、内容传输**。

如图 2-1 请求阶段分析所示，这些阶段共需要 5 个 RTT（Round-Trip Time，往返时间）[^2] = 1 RTT（DNS Lookup，域名解析）+ 1 RTT（TCP Handshake，TCP 握手）+ 2 RTT（SSL Handshake，SSL 握手）+ 1 RTT（Data Transfer，HTTP 内容请求传输）。

:::center 
  ![](../assets/http-process.png)<br/>
  图 2-1 HTTPS（使用 TLS1.2 协议）请求阶段分析 [图片来源](https://blog.cloudflare.com/a-question-of-timing)
:::

RTT 是评估本地主机与远程主机间网络延迟的重要指标之一。

举个例子，北京到美国洛杉矶的 RTT 延迟为 190 ms，那么从北京访问美国洛杉矶的服务延迟就是： 4 * 190（ms） + 后端业务处理时间（ms）。其中“4”代表 HTTPS 请求的 4 个 RTT。

由于 RTT 反映的是因物理距离带来的延迟，而 SSL 阶段又包含大量的加密/解密计算消耗。因此，优化工作应该集中在减少 RTT 和降低 SSL 计算量上。

## 2.2.2 各阶段耗时分析

HTTPS 请求的各个阶段可以使用 curl 命令进行详细的延迟分析。

curl 命令提供了 -w 参数，该参数支持 curl 按照指定的格式打印与请求相关的信息，部分信息可以用特定的变量表示，例如 status_code、size_download、time_namelookup 等等。因为我们要进行耗时分析，所以只关注和请求延迟有关的变量（以 time_ 开头的变量）。往文本文件 curl-format.txt 写入下面的内容：

```bash
$ cat curl-format.txt
    time_namelookup:  %{time_namelookup}\n
       time_connect:  %{time_connect}\n
    time_appconnect:  %{time_appconnect}\n
      time_redirect:  %{time_redirect}\n
   time_pretransfer:  %{time_pretransfer}\n
 time_starttransfer:  %{time_starttransfer}\n
                    ----------\n
         time_total:  %{time_total}\n
```

上述的变量的含义，如表 2-2 所示。

:::center
表 2-2 curl 支持的与请求延迟有关的变量
:::

| 变量名称 | 变量释义 |
|:--|:--|
| time_namelookup | 从请求开始到域名解析完成的耗时 |
| time_connect | 从请求开始到 TCP 三次握手完成的耗时 |
| time_appconnect | 从请求开始到 TLS 握手完成的耗时 |
| time_pretransfer | 从请求开始到向服务器发送第一个 GET/POST 请求开始之前的耗时 |
| time_redirect | 重定向耗时，包括到内容传输前的重定向的 DNS 解析、TCP 连接、内容传输等时间 |
| time_starttransfer | 从请求开始到内容传输前的耗时 |
| time_total | 从请求开始到完成的总耗时 |


我们先看看一个简单的请求，如下所示：

```bash
$ curl -w "@curl-format.txt" -o /dev/null -s 'https://www.iq.com/'
// curl 打印的与耗时有关的信息（单位秒）
time_namelookup=0.025021
time_connect=0.033326
time_appconnect=0.071539
time_redirect=0.000000
time_pretransfer=0.071622
time_starttransfer=0.088528
time_total=0.088744
```
上述命令各个参数的意义是：
- -w：从文件中读取要打印信息的格式。
- -o /dev/null：把响应的内容丢弃。我们并不关心 HTTPS 的返回内容，只关心请求的耗时情况。
- -s：不输出请求的进度条。

不过，得注意 curl 打印的各个耗时都是从请求发起的那一刻开始计算，我们得将其转换成 HTTPS 各阶段耗时，例如域名解析耗时、TCP 建立耗时、TTFB 耗时[^3]等。

表 2-3 为 curl 内各个步骤耗时与 HTTPS 指标计算关系。

:::center
表 2-3 HTTPS 请求各阶段耗时计算
:::
| 耗时 | 说明 |
|:--|:--|
| 域名解析耗时 = time_namelookup | 域名 NS 及本地 LocalDNS 解析耗时 |
| TCP 握手耗时 = time_connect - time_namelookup | 建立 TCP 连接时间 |
| SSL 耗时 = time_appconnect - time_connect | TLS 握手以及加解密处理 |
| 服务器处理请求耗时 = time_starttransfer - time_pretransfer | 服务器响应第一个字节到全部传输完成耗时 |
| TTFB  = time_starttransfer - time_appconnect | 服务器从接收请求到开始到收到第一个字节的耗时 |
| 总耗时 = time_total | 整个 HTTPS 的请求耗时|


## 2.2.3 HTTPS 的优化总结

了解 HTTPS 请求的各个阶段以及相应的延迟计算后，我们可以针对性地采取以下优化措施：

- 域名解析优化：减少域名解析产生的延迟。例如，提前获取域名解析结果备用，那么后续的 HTTPS 连接就能减少一个 RTT。
- 对传输内容进行压缩：传输数据的大小与耗时成正比，压缩传输内容是降低请求耗时最有效的手段之一。
- SSL 层优化：升级 TLS 算法和 HTTPS 证书，例如升级 TLS 1.3 协议，可将 SSL 握手的 RTT 从 2 个减少到 1 个。
- 传输层优化：升级拥塞控制算法以提高网络吞吐量。将默认的 Cubic 升级为 BBR 对于大带宽、长链路的弱网环境尤其有效。
- 网络层优化：使用商业化的网络加速服务，通过路由优化数据包，实现动态服务加速。
- 使用更现代的 HTTP 协议：升级至 HTTP/2，进一步升级到基于 QUIC 协议的 HTTP/3。

接下来，笔者将逐一介绍上述优化手段的原理和实践效果。

[^1]: 参见 https://blog.cloudflare.com/a-question-of-timing/
[^2]: RTT（Round-Trip Time）一个网络数据包从起点到目的地然后再回到起点所花费的时长。
[^3]: TTFB（Time To First Byte，首字节时间）指从浏览器请求页面到接收来自服务器发送的信息的第一个字节的时间。

