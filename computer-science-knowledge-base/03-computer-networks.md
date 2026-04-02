# Computer Networks — Graduate-Level Reference

> 结构：概念 → 原理 → 代码/伪代码 → 复杂度 → 应用

---

## 1. OSI 七层 & TCP/IP 四层

| OSI 层 | TCP/IP 层 | PDU | 协议示例 | 功能 |
|--------|----------|-----|---------|------|
| 7 应用 | 应用 | 数据 | HTTP, DNS, SMTP, FTP, SSH | 为应用提供服务 |
| 6 表示 | 应用 | 数据 | TLS, JPEG, ASCII | 数据格式、加密 |
| 5 会话 | 应用 | 数据 | RPC, NetBIOS | 会话管理 |
| 4 传输 | 传输 | 段(Segment)/数据报 | TCP, UDP, QUIC | 端到端通信 |
| 3 网络 | 网际 | 包(Packet) | IP, ICMP, ARP | 路由寻址 |
| 2 数据链路 | 网络接口 | 帧(Frame) | Ethernet, PPP, VLAN | 相邻节点传输 |
| 1 物理 | 网络接口 | 比特(Bit) | 光纤, 双绞线 | 比特流传输 |

---

## 2. 物理层

### 2.1 编码方式

| 编码 | 原理 | 时钟恢复 | 带宽效率 |
|------|------|---------|---------|
| NRZ | 1=高, 0=低 | ❌（长连续位） | 1 bit/baud |
| Manchester | 中间跳变：↑=1, ↓=0 | ✅ | 0.5 bit/baud |
| 4B/5B | 4位数据→5位编码 | ✅ | 0.8 bit/baud |
| 8B/10B | 8位→10位，DC平衡 | ✅ | 0.8 bit/baud |
| 64B/66B | 64位+2位同步头 | ✅ | 0.97 bit/baud |

### 2.2 复用技术

| 技术 | 原理 | 应用 |
|------|------|------|
| FDM | 频分复用，不同频带 | 有线电视、4G LTE |
| TDM | 时分复用，不同时隙 | 电话网(E1) |
| WDM | 波分复用，不同波长 | 光纤通信 |
| CDM | 码分复用，正交编码 | CDMA, GPS |

**CDM 编码**：每个站分配唯一 m 位码片序列（chip sequence），发送 1 用原码片，发送 0 取反。接收方内积解码：`S·T/m = ±1`（有数据）或 `0`（无数据）。

---

## 3. 数据链路层

### 3.1 以太网帧 (IEEE 802.3)

```
┌─────────┬─────────┬─────────┬──────────────┬───────┬───────┐
│ 前导码   │ SFD     │ 目的MAC  │ 源 MAC       │ 类型   │ 数据   │ FCS   │
│ 7B      │ 1B      │ 6B      │ 6B           │ 2B    │46-1500B│ 4B   │
└─────────┴─────────┴─────────┴──────────────┴───────┴───────┘
最小帧长 64B，最大 1518B（不含前导码）
```

### 3.2 CSMA/CD（载波侦听多路访问/冲突检测）

```
1. 侦听信道空闲 → 发送
2. 发送中持续检测冲突
3. 检测到冲突 → 发送干扰信号 (jam) → 所有站停止
4. 二进制指数退避：
   第 k 次冲突：从 [0, 2^k - 1] 中随机选 r，等待 r × 2τ
   k > 10 后不再增加，k = 16 放弃
```

**最小帧长限制**：帧发送时间 ≥ 2τ（往返传播延迟），确保发送时能检测到冲突。

### 3.3 CRC 校验

**原理**：多项式除法，将数据视为多项式，除以生成多项式 G(x)，余数为校验码。

```
CRC-32: G(x) = x^32 + x^26 + x^23 + x^22 + x^16 + x^12 + x^11 + x^10 + x^8 + x^7 + x^5 + x^4 + x^2 + x + 1
```

```python
def crc32(data, poly=0xEDB88320):
    crc = 0xFFFFFFFF
    for byte in data:
        crc ^= byte
        for _ in range(8):
            crc = (crc >> 1) ^ poly if crc & 1 else crc >> 1
    return crc ^ 0xFFFFFFFF
```

**检错能力**：检测所有单/双位错、所有奇数位错、所有 ≤ 32 位的突发错误。

### 3.4 VLAN (802.1Q)

```
在以太网帧中插入 4 字节 tag：
┌──────────┬──────┬────────┬──────────┐
│ TPID(2B) │ TCI  │ 类型    │ 数据      │
│ 0x8100   │ VID  │         │          │
└──────────┴──────┴────────┴──────────┘
VID: 12位，最多 4094 个 VLAN
```

### 3.5 ARP (Address Resolution Protocol)

```
主机 A 要发数据给同网段主机 B (知道 B 的 IP)：
1. A 检查 ARP 缓存
2. 缓存未命中 → 广播 ARP Request（谁有 IP_B？告诉我 MAC_A）
3. B 回复 ARP Reply（MAC_B → A 的单播）
4. A 缓存 IP_B → MAC_B 映射（TTL 通常 20min）
```

---

## 4. 网络层

### 4.1 IPv4 头部

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
┌─────────┬───┬───────┬──────────────────────────────────────────┐
│Version  │IHL│DSCP/ECN│ Total Length                             │
├─────────┴───┴───────┼─────────────┬────────────────────────────┤
│ Identification       │Flags│Fragment│ Time to Live  │ Protocol │
│                      │     │Offset  │               │          │
├──────────────────────┴─────────────┴───────────────┴──────────┤
│ Header Checksum                                               │
├───────────────────────────────────────────────────────────────┤
│ Source IP Address                                              │
├───────────────────────────────────────────────────────────────┤
│ Destination IP Address                                         │
└───────────────────────────────────────────────────────────────┘
```

**关键字段**：
- TTL：每经路由器 -1，减到 0 丢弃（防环路）
- Fragment Offset：分片偏移（8 字节为单位）
- Protocol：6=TCP, 17=UDP, 1=ICMP

### 4.2 IPv6

```
固定 40 字节头部（简化）：
- 地址 128 位 → 2^128 个地址
- 取消校验和（交给上层）
- 无分片（路由器不分片，源端 PMTU 发现）
- Flow Label：标识数据流
```

### 4.3 子网划分 (CIDR)

```
IP: 192.168.1.100/26
网络地址 = IP & 子网掩码
192.168.1.100 & 255.255.255.192 = 192.168.1.64
主机范围: 192.168.1.65 ~ 192.168.1.126 (62 台主机)
广播地址: 192.168.1.127

CIDR 聚合（路由汇总）：
192.168.0.0/24 + 192.168.1.0/24 → 192.168.0.0/23
```

### 4.4 路由协议

| 协议 | 类型 | 算法 | 度量 | 适用 |
|------|------|------|------|------|
| RIP | IGP, 距离向量 | Bellman-Ford | 跳数(≤15) | 小型网络 |
| OSPF | IGP, 链路状态 | Dijkstra | 代价(带宽) | 大型企业 |
| IS-IS | IGP, 链路状态 | Dijkstra | 代价 | ISP 骨干 |
| BGP | EGP, 路径向量 | 路径属性 | 策略 | 互联网 |

**RIP**：每 30s 广播路由表，最大 15 跳，16 视为不可达。收敛慢，可能计数到无穷。

**OSPF**：
- 链路状态通告 (LSA) 泛洪，每台路由器构建完整拓扑图
- 以自己为根运行 Dijkstra 得到最短路径树
- 分区域（Area）层次化设计，减少 LSA 泛洪范围
- Hello 间隔 10s，死亡间隔 40s

**BGP**：
- 基于 TCP (端口 179)，路径向量协议
- AS 间路由决策基于属性：Weight > Local Preference > AS-Path 长度 > Origin > MED > EBGP>IBGP > IGP 度量
- 防环：AS-Path 包含自身 AS 号则拒绝

### 4.5 ICMP

```
类型 8/0: Echo Request/Reply (ping)
类型 3:   Destination Unreachable
类型 5:   Redirect
类型 11:  Time Exceeded (traceroute 利用此)
```

**Traceroute 原理**：发送 TTL=1,2,3... 的 UDP 包，每个路由器 TTL 超时返回 ICMP Time Exceeded，最终到达目的返回 Port Unreachable。

### 4.6 NAT (Network Address Translation)

```
内网 10.0.0.x → NAT 网关 → 公网 IP

SNAT (源地址转换): 内网→公网，替换源 IP:Port
DNAT (目的地址转换): 公网→内网，替换目的 IP:Port (端口转发)
PAT (端口地址转换): 多内网主机共享一个公网 IP，靠端口号区分
```

**NAT 表项**：
```
内网 IP:Port        → 外网 IP:Port
10.0.0.5:12345     → 203.0.113.1:54321
```

### 4.7 VPN

**IPsec**：
- AH (认证头)：完整性验证，无加密
- ESP (封装安全载荷)：加密 + 完整性
- 两种模式：传输模式（仅载荷加密）、隧道模式（整个原始包加密）
- IKE (Internet Key Exchange)：密钥协商，基于 Diffie-Hellman

**GRE**：通用路由封装，轻量级隧道协议，不加密。

---

## 5. 传输层

### 5.1 TCP

#### 三次握手

```
Client                          Server
  │──── SYN (seq=x) ────────────│  → SYN_RCVD
  │←─── SYN+ACK (seq=y,ack=x+1)│
  │──── ACK (ack=y+1) ─────────│  → ESTABLISHED
  │                              │
```

**为什么三次？**：防止历史重复 SYN 导致服务器误建连接。

#### 四次挥手

```
Client                          Server
  │──── FIN (seq=u) ────────────│  → CLOSE_WAIT
  │←─── ACK (ack=u+1) ─────────│
  │                              │
  │←─── FIN (seq=w) ────────────│  → LAST_ACK
  │──── ACK (ack=w+1) ─────────│  → TIME_WAIT (2MSL)
```

**TIME_WAIT (2MSL)**：确保最后一个 ACK 到达；等待网络中残留报文消失。MSL 通常 30s~2min。

#### TCP 状态机

```
CLOSED → SYN_SENT → ESTABLISHED → FIN_WAIT_1 → FIN_WAIT_2 → TIME_WAIT → CLOSED
                                          ↑
CLOSED → LISTEN → SYN_RCVD → ESTABLISHED → CLOSE_WAIT → LAST_ACK → CLOSED
```

#### 流量控制（滑动窗口）

```
发送方：
┌──────────┬──────────────┬──────────────────────────┐
│ 已确认    │  窗口内已发送  │  窗口内未发送              │  窗口外
└──────────┴──────────────┴──────────────────────────┘

接收方通过 ACK 中的 Window Size 告知可用缓冲区大小。
窗口探测 (Zero Window Probe)：当窗口为 0 时，发送方定期发 1 字节探测。
```

**糊涂窗口综合征**：避免发送小报文。Nagle 算法：只有收到前一个 ACK 或积累足够数据才发送。

#### 拥塞控制

```
                  ┌─────────── 拥塞避免 (线性增长 +1/cwnd/RTT)
                  │
1 ── 慢启动(指数) ─┤ ─── ssthresh
  cwnd × 2/RTT   │
                  └───→ 超时：ssthresh = cwnd/2, cwnd = 1, 重启慢启动
                       3次重复ACK：快重传 + 快恢复
                         ssthresh = cwnd/2
                         cwnd = ssthresh + 3
                         → 进入拥塞避免
```

**BBR (Bottleneck Bandwidth and Round-trip propagation time)**：
- 不基于丢包判断拥塞，而是测量瓶颈带宽 BtlBw 和最小 RTT
- 发送速率 = BtlBw × pacing_gain
- 周期性探测带宽增加（1.25×）和减少（0.75×）
- 在高延迟/有丢包的网络中比 CUBIC 性能好很多

### 5.2 UDP

**特点**：无连接、不可靠、无序、无流量/拥塞控制。头部仅 8 字节。

```
┌──────┬──────┬──────┬──────┐
│ 源端口│目端口│ 长度  │ 校验和│
│ 2B   │ 2B   │ 2B   │ 2B   │
└──────┴──────┴──────┴──────┘
```

**适用**：实时音视频、DNS、游戏、IoT。

### 5.3 QUIC

**原理**：基于 UDP 的可靠传输协议（HTTP/3 底层）。

**特性**：
- **0-RTT 连接建立**：首次 1-RTT，后续复用密钥 0-RTT
- **连接迁移**：用 Connection ID 而非四元组标识连接（WiFi→4G 无需重连）
- **多路复用无队头阻塞**：每条流独立，一条丢包不影响其他
- **内置 TLS 1.3**：协议原生加密
- **用户态拥塞控制**：可快速迭代

---

## 6. 应用层

### 6.1 DNS

**查询过程**（递归 + 迭代）：
```
浏览器 → 本地 DNS 服务器（递归查询）
       → 根域名服务器 → .com TLD 服务器 → 权威域名服务器（迭代查询）
```

**记录类型**：

| 类型 | 用途 | 示例 |
|------|------|------|
| A | IPv4 地址 | `example.com → 93.184.216.34` |
| AAAA | IPv6 地址 | `example.com → 2606:2800:220:1:...` |
| CNAME | 别名 | `www.example.com → example.com` |
| MX | 邮件服务器 | `example.com → mail.example.com` |
| NS | 域名服务器 | `example.com → ns1.dnsprovider.com` |
| TXT | 文本记录 | SPF、DKIM、验证 |
| SRV | 服务定位 | `_sip._tcp.example.com → ...` |

**缓存 TTL**：每条记录有 TTL，过期需重新查询。

### 6.2 HTTP

#### HTTP/1.0 → HTTP/1.1

```
HTTP/1.0: 每个请求新建 TCP 连接
HTTP/1.1:
  - 持久连接 (Connection: keep-alive)
  - 管线化 (pipelining): 不等响应连续发请求（但队头阻塞）
  - Host 头：虚拟主机
  - 分块传输 (Transfer-Encoding: chunked)
```

#### HTTP/2

```
特性：
- 二进制帧 (Frame) 替代文本
- 流 (Stream): 一个 TCP 连接上多个并发流
- 多路复用: 无队头阻塞（应用层，TCP 层仍有）
- 头部压缩 (HPACK): 静态表 + 动态表 + Huffman 编码
- 服务器推送 (Server Push)
- 流优先级 (Stream Priority)
```

#### HTTP/3

```
基于 QUIC (UDP):
- 解决 TCP 层队头阻塞
- 0-RTT 连接恢复
- QPACK 头部压缩（类似 HPACK，适配 QUIC 的有序流模型）
```

#### 常见状态码

```
200 OK, 201 Created, 204 No Content
301 Moved Permanently, 302 Found, 304 Not Modified
400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found
500 Internal Server Error, 502 Bad Gateway, 503 Service Unavailable
```

### 6.3 TLS 1.3

**握手过程（1-RTT）**：
```
Client                                         Server
  │─── ClientHello ─────────────────────────────│
  │    (supported_versions, key_share,           │
  │     signature_algorithms, cipher_suites)     │
  │←── ServerHello + Extensions ────────────────│
  │    (key_share, cipher_suite)                 │
  │←── Certificate + CertificateVerify ─────────│
  │←── Finished ────────────────────────────────│
  │─── Finished ────────────────────────────────│
  │═══ Application Data (encrypted) ════════════│
```

**0-RTT（会话恢复）**：
```
客户端保存 PSK (Pre-Shared Key)，在 ClientHello 中携带：
  │─── ClientHello + early_data ────────────────│
  │═══ 0-RTT Application Data ══════════════════│  ← 立即发送数据
```

**密码套件**：TLS 1.3 精简为 5 个：
- TLS_AES_128_GCM_SHA256
- TLS_AES_256_GCM_SHA384
- TLS_CHACHA20_POLY1305_SHA256
- TLS_AES_128_CCM_SHA256
- TLS_AES_128_CCM_8_SHA256

**密钥交换**：仅支持 (EC)DHE，强制前向保密。

### 6.4 WebSocket

```
升级握手：
GET /chat HTTP/1.1
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13

响应：
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
  (= SHA-1(key + "258EAFA5-E914-47DA-95CA-5AB5DC85B11B"))
```

### 6.5 SMTP / POP3 / IMAP

```
SMTP (TCP 25/587): 发送邮件
  MAIL FROM, RCPT TO, DATA
  支持 STARTTLS 升级为加密

POP3 (TCP 110/995): 下载邮件（通常下载后删除）
  USER, PASS, LIST, RETR, DELE, QUIT

IMAP (TCP 143/993): 在线管理邮件（服务器端操作）
  SELECT, FETCH, SEARCH, MOVE, EXPUNGE
```

---

## 7. Socket 编程

### 7.1 TCP Server-Client

```python
# TCP Server
import socket

server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
server.bind(('0.0.0.0', 8080))
server.listen(128)  # backlog 队列
conn, addr = server.accept()
data = conn.recv(4096)
conn.sendall(b'HTTP/1.1 200 OK\r\n\r\nHello')
conn.close()

# TCP Client
client = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
client.connect(('127.0.0.1', 8080))
client.sendall(b'Hello Server')
response = client.recv(4096)
client.close()
```

### 7.2 UDP Server-Client

```python
# UDP Server
server = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
server.bind(('0.0.0.0', 9090))
data, addr = server.recvfrom(4096)
server.sendto(b'Response', addr)

# UDP Client
client = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
client.sendto(b'Hello', ('127.0.0.1', 9090))
data, addr = client.recvfrom(4096)
```

### 7.3 epoll 事件循环

```python
import select
import socket

server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
server.setblocking(False)
server.bind(('0.0.0.0', 8080))
server.listen(1024)

epoll = select.epoll()
epoll.register(server.fileno(), select.EPOLLIN)

connections = {}
try:
    while True:
        events = epoll.poll(1)  # timeout 1ms
        for fd, event in events:
            if fd == server.fileno():
                conn, addr = server.accept()
                conn.setblocking(False)
                connections[conn.fileno()] = conn
                epoll.register(conn.fileno(), select.EPOLLIN)
            elif event & select.EPOLLIN:
                data = connections[fd].recv(1024)
                if data:
                    connections[fd].send(data)  # echo
                else:
                    epoll.unregister(fd)
                    connections[fd].close()
                    del connections[fd]
finally:
    epoll.unregister(server.fileno())
    epoll.close()
    server.close()
```

---

## 8. 网络安全

### 8.1 防火墙

| 类型 | 工作层 | 检查内容 |
|------|--------|---------|
| 包过滤 | 网络层 | IP、端口、协议 |
| 状态检测 | 网络层+传输层 | 连接状态表 |
| 应用网关 (WAF) | 应用层 | HTTP 内容、SQL 注入、XSS |

**iptables/Netfilter 链**：
```
PREROUTING → INPUT → [本地进程] → OUTPUT → POSTROUTING
                 ↓                                ↑
              FORWARD ────────────────────────────┘
```

### 8.2 DDoS 攻击与防御

| 攻击类型 | 原理 | 防御 |
|---------|------|------|
| SYN Flood | 大量 SYN 耗尽半连接队列 | SYN Cookie、增加队列、防火墙 |
| UDP Flood | 大量 UDP 包耗尽带宽 | 限流、黑洞路由 |
| DNS 放大 | 伪造源 IP 发 DNS 查询 | BCP38（源地址验证）、限速 |
| HTTP Flood | 大量合法 HTTP 请求 | WAF、验证码、CDN 分流 |

**SYN Cookie 原理**：
```
不分配资源保存半连接状态，而是将状态编码到 SYN-ACK 的序列号中：
ISN = f(sip, dip, sport, dport, secret, time)
客户端回复 ACK 时验证序列号合法性 → 合法则建连接
```

### 8.3 中间人攻击 (MITM)

**攻击方式**：ARP 欺骗、DNS 欺骗、伪造 CA、SSL 剥离。

**防御**：
- HSTS（强制 HTTPS）
- Certificate Pinning
- DNSSEC
- TLS 1.3（简化握手，减少攻击面）

### 8.4 常见加密算法

| 类型 | 算法 | 用途 |
|------|------|------|
| 对称加密 | AES-128/256-GCM, ChaCha20 | 数据加密 |
| 非对称加密 | RSA-2048, ECDSA (P-256), Ed25519 | 密钥交换、签名 |
| 哈希 | SHA-256, SHA-3, BLAKE3 | 完整性 |
| 密钥交换 | ECDHE (P-256, X25519) | 前向保密 |
| MAC | HMAC-SHA256, Poly1305 | 消息认证 |

---

## 附录：关键公式速查

**带宽延迟积 (BDP)**：
```
BDP = Bandwidth × RTT
例：100Mbps × 50ms = 6.25 MB → TCP 窗口需 ≥ 6.25MB 才能充分利用带宽
```

**以太网效率**：
```
效率 = 帧有效数据 / (帧长 + 帧间隙)
     = 1500 / (1518 + 12 + 8) ≈ 97.5% (最大帧)
     = 46 / (64 + 12 + 8) ≈ 38.1% (最小帧)
```

**TCP 吞吐量（拥塞避免阶段）**：
```
W(t) = W(0) + t/RTT  (线性增长)
吞吐量 = W × MSS / RTT
```

**香农定理**：
```
C = B × log₂(1 + S/N)
C = 信道容量 (bps), B = 带宽 (Hz), S/N = 信噪比
```

**奈奎斯特定理**：
```
最大数据速率 = 2B × log₂V
B = 带宽, V = 离散信号电平数
```
