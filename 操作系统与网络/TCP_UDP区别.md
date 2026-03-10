# TCP vs UDP (TCP/UDP Differences)

## 核心区别

| 特性 | TCP (Transmission Control Protocol) | UDP (User Datagram Protocol) |
| :--- | :--- | :--- |
| **连接性** | 面向连接 (Connection-Oriented) | 无连接 (Connectionless) |
| **可靠性** | **可靠传输**：保证数据按序到达、不丢失、不重复 | **不可靠传输**：不保证顺序、不保证到达，可能丢包 |
| **流量控制** | 拥塞控制、流量控制 (Sliding Window) | 无流量控制，发送端可以任意速率发送 |
| **头部开销** | 大 (20 字节 + 选项)，复杂 | 小 (8 字节)，简单 |
| **传输模式** | 字节流 (Byte Stream) | 数据报 (Datagram) |
| **适用场景** | 文件传输、邮件、网页浏览 (HTTP/HTTPS) | 实时音视频、在线游戏 (FPS/MOBA)、广播/多播 |
| **拥塞控制** | 有 (慢启动、拥塞避免、快重传、快恢复) | 无 (丢包直接重传或忽略) |

---

## TCP 的可靠性机制

1.  **序列号 (Sequence Number)**：
    *   TCP 给每个字节的数据都分配一个序列号，保证接收端能按序重组数据。
2.  **确认应答 (ACK)**：
    *   接收方收到数据后，回送确认号（Ack Number = Seq + Length），告知发送方已成功接收。
3.  **重传机制 (Retransmission)**：
    *   **超时重传**：如果在 RTO (Retransmission Timeout) 时间内未收到 ACK，发送方重传数据。
    *   **快速重传**：如果收到 3 个相同的 ACK，说明中间有数据包丢失，立即重传该丢失的报文段。
4.  **流量控制 (Flow Control)**：
    *   利用**滑动窗口 (Sliding Window)** 机制，接收方告知发送方自己的接收窗口大小 (Advertised Window)，防止发送过快导致缓冲区溢出。
5.  **拥塞控制 (Congestion Control)**：
    *   **慢启动 (Slow Start)**：初始发送窗口较小，指数增长。
    *   **拥塞避免 (Congestion Avoidance)**：达到阈值后线性增长。
    *   **快重传 (Fast Retransmit)**：收到 3 个重复 ACK 时触发。
    *   **快恢复 (Fast Recovery)**：将拥塞窗口减半，然后线性增长。

---

## TCP 三次握手 (Three-Way Handshake)

建立连接的过程，确保双方的发送和接收能力正常。

1.  **SYN (Client -> Server)**:
    *   客户端发送 `SYN=1, Seq=x`。
    *   状态：`SYN_SENT`。
2.  **SYN-ACK (Server -> Client)**:
    *   服务端收到 SYN，回复 `SYN=1, ACK=1, Seq=y, Ack=x+1`。
    *   状态：`SYN_RCVD`。
3.  **ACK (Client -> Server)**:
    *   客户端收到 SYN-ACK，回复 `ACK=1, Seq=x+1, Ack=y+1`。
    *   状态：`ESTABLISHED`。
    *   服务端收到 ACK 后也进入 `ESTABLISHED` 状态。

**为什么是三次？**
*   防止已失效的连接请求报文段突然又传送到了服务端，产生错误连接。
*   确认双方的收发能力：
    *   第一次：Server 确认 Client 发送能力正常。
    *   第二次：Client 确认 Server 接收、发送能力正常。
    *   第三次：Server 确认 Client 接收能力正常。

---

## TCP 四次挥手 (Four-Way Wave)

断开连接的过程。

1.  **FIN (Client -> Server)**:
    *   客户端发送 `FIN=1, Seq=u`，表示数据发送完毕。
    *   状态：`FIN_WAIT_1`。
2.  **ACK (Server -> Client)**:
    *   服务端收到 FIN，回复 `ACK=1, Seq=v, Ack=u+1`。
    *   此时服务端可能还有数据未发送完，进入半关闭状态。
    *   状态：`CLOSE_WAIT` (Client 进入 `FIN_WAIT_2`)。
3.  **FIN (Server -> Client)**:
    *   服务端数据发送完毕，发送 `FIN=1, ACK=1, Seq=w, Ack=u+1`。
    *   状态：`LAST_ACK`。
4.  **ACK (Client -> Server)**:
    *   客户端收到 FIN，回复 `ACK=1, Seq=u+1, Ack=w+1`。
    *   状态：`TIME_WAIT` (等待 2MSL 时间后进入 `CLOSED`)。
    *   服务端收到 ACK 后进入 `CLOSED`。

**为什么是四次？**
*   因为 TCP 是全双工的，发送方发送 FIN 只是表示“我不再发送数据了”，但还可以接收数据。
*   服务端收到 FIN 后，可能还有数据需要处理和发送，所以先回一个 ACK，等数据处理完再发 FIN。

**TIME_WAIT 状态的作用 (2MSL)**
1.  确保最后一个 ACK 能到达服务端（如果丢失，服务端会重传 FIN，客户端需能再次回复 ACK）。
2.  防止“已失效的连接请求报文段”出现在本连接中（等待足够时间让网络中残留的报文段消失）。
