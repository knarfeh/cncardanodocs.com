---
layout: default
title: 网络传输层
permalink: /technical/protocols/network-transport/
group: technical-protocols
---
<!-- Reviewed at ef835a2334888eda7384da707c4077a8b576b192 -->

# 网络传输层

本指南适用于想为卡尔达诺结算层构建自己的客户端的开发人员。请阅读[卡尔达诺结算层实现概述](/technical/)了解更多信息。本指南涵盖了卡尔达诺结算层节点中使用的网络传输层。

传输层是一个位于 TCP 和应用程序级协议的层。原则上独立于应用程序协议（事实上，参考实现被具有不同应用程序级协议的多个不同应用程序使用）。

传输层的重点在于它提供了在单个 TCP 连接上复用的多个轻量级逻辑连接。每个轻量级连接都是单向的，并提供可靠的有序消息传输（即，它在 TCP 之上提供数据帧）

传输协议的属性：

- **单个 TCP 连接**。任何一对对等点之间一次只能使用一个 TCP 连接。这些连接可能是长期存在的。一旦建立了与对等节点的链接，它将用于发送/接收信息，知道 TCP连接被*明确*关闭或发生一些不可恢复的错误。

实现的属性：

- **报告网络故障**。网络故障不会从应用程序层隐藏。如果 TCP 连接意外断开，传输层应通知应用层。在卡尔达诺结算层中，策略是尝试重新连接，如果重新连接也失败，则只声明对等方无法访问。

## 概要

传输层的典型用途包括：

1. 监听来自对等点的 TCP 连接。
2. 建立到其他对等点的 TCP 连接。
3. 在建立的 TCP 连接上创建轻型连接。
4. 发送消息到对等节点（在一个或多个轻量级连接上）。
5. 接收来自对等节点（在一个或多个轻量级连接上）的消息。
6. 关闭轻量级连接
7. 关闭 TCP 连接。

在卡尔达诺结算层中，使用多个轻量级连接来支持应用程序级的消息传递协议。可以同时发送多个应用程序级别的消息，多个对话可以一次进行。大多数应用程序消息是在新创建的轻量级连接上发送的，如果需要的话，较大的应用程序级别消息被分解为多个传输级别消息以便于传输，其他应用程序级别的消息是作为一对单向轻量级连接组成的对话的一部分发送的。


## 概观

传输层的基本概念有：

- 传输
- 接入点
- 连接
- 事件
- 错误

传输指本文档描述的整个层和协议。

**传输**是指本文档描述的整个层和协议。一个传输实例指的是传输实现的配置和状态，特别是包括绑定到本地网络特定接口的TCP监听套接字，如 `192.168.0.1:3010`。

**接入点**是传输实例的逻辑端点。这意味着它又有一个地址，连接在端点之间。在实践中，它只是一个 TCP/IP 的简单抽象，通过主机名和端口进行寻址。

端口地址是具有结构如 `HOST:PORT:LOCAL_ID` 的二进制字符串，例如 `192.168.0.1:3010:0`。

注意，传输实例监听单个端口时，原则上在单个传输实例中可能有多个可寻址的接入点，这就是 `LOCAL_ID` 的作用，然而，卡尔达诺结算层目前还没有这个功能，所以它总是使用 `LOCAL_ID` 0。

**重量级连接**是指两个端点之前的 TCP 连接。两个接入点只使用一次 TCP 连接。

**连接**（或者更明确地说是*轻量级连接*）是端点之前的单向连接。端点之前的所有轻量级连接都在单个重量级连接（即单个TCP连接上）进行复用。

轻量级连接是在TCP之上分层的逻辑概念。每个连接都有一个整数ID。原则上可以在单个重量级TCP连接上复用数千个轻量级连接。

典型的操作方式是应用层希望建立到端点的轻量级连接，如果还没有重量级连接，则创建一个。同样，当最后一个轻量级连接关闭时，真正的 TCP 连接将被彻底关闭。

轻量级连接时单向的：轻量级连接上的消息仅在一个方向流动。但是，轻量级连接可以在任何方向上建立。同样的重量级连接用于双向的轻量级连接；哪个端点先建立重量级连接并不重要。

双向会话可以通过使用一对单向轻量级连接来建立。卡尔达诺结算层遵循这种模式。请查阅 `time-warp-nt` 获取详细信息。但请注意，这个传输层没有双向对话的特殊概念，只有单向连接的集合。

## 网络字节顺序

在一下对控制消息的描述中，所有整数都是按[网络字节顺序](https://en.wikipedia.org/wiki/Endianness#Networking)编码的。

下面的消息定义使用的 `Int32` 指的是32位的以网络字节顺序的整数值。

## 设置传输实例

每个传输实例都必须建立一个 TCP 监听套接字。使用的本地端口和端口号由使用传输的应用程序确定。

实现可以随时接收新的 TCP 连接（可能受限于资源策略），然后执行下面描述的新重量级连接的初始步骤。

## 建立重量级连接（初始化）

假设在接入点 A，B之前建立重量级连接，端点 A 发起连接。两个端点都有端点地址，如前所述，端点地址是这种形式：`HOST:PORT:LOCAL_ID`。

从 A 到 B 建立的一个重量级连接的过程如下。首先 A 必须必须在本地记录它正在初始化一个到 B 的重量级连接。在交叉连接请求的情况下（见下文）这是必须的，由端点 A 向端点 B 打开 `HOST` 和 `PORT` 连接。

端点 A 发送具有如下结构的**连接请求**消息：


    +-----------+-------------+--------------------+
    |   B-LID   |   A-EIDlen  |       A-EID        |
    +-----------+-------------+--------------------+
    |   Int32   |   Int32     |       bytes        |

其中：

-   `B-LID` - `B` 端点的本地 ID;
-   `A-EIDlen` - `A` 端点的地址;
-   `A-EID` - `A` 端口的地址。

因此 A 发送它希望连接的本地接入点 ID，它自身的地址来初始化节点。A 发送的地址应该是规范化的公共地址。主机部分可以是 IP 地址或 DNS 名称。它用于避免在端点之间建立多个 TCP 连接。在卡尔达诺结算层协议中，本地端点 ID 始终为0。

然后接入点 A 期望一个**连接请求响应**信息，它是下面的响应之一，一个简单的 `Int32` 编码。

当本地接入点 ID 所标识的端点不存在时，会返回 `ConnectionRequestInvalid` 响应。例如，如果 A 发送给 B，它希望连接到本地接入点 ID 1，那么只有 ID 0 存在时才会发生。在这种情况下，两个端点必须关闭 TCP 连接。

当端点 B 确定 A 与 B 之间或 B 与 A 之间，或两者同时有了一个 TCP 连接，会返回 `ConnectionRequestCrossed` 响应。在这种情况下，两个端点都必须关闭 TCP 连接。

## 建立重量级连接 (接收)

假设如前所述，在标记为 A 和 B 的端点之前建立重量级连接，并且端点 A 发起连接。我们现在从端点B的角度来考虑这个问题。

两个端点都有 `HOST:PORT:LOCAL_ID` 形式的接入点地址。具体来说，假设 B 只有一个接入点，其中 `LOCAL_ID` 为 0。

B 的传输实例在对应的接入点 IDs 上相应的 host 和 port 有监听套接字。它接受来自某个对等点的新的 TCP 连接。期望在该 TCP 连接上接收**连接请求**信息（以上述格式）。



Transport instance B must now respond with a **connection request response**
message (in the format described above), based on the following rules.

If the connection request asks for a local endpoint ID that does not exist (i.e.
anything other than 0 in our example), it must respond with
`ConnectionRequestInvalid` and close the TCP connection.

The rules for `ConnectionRequestCrossed` are described below in more detail.

Otherwise, when the endpoint ID is valid and there is no existing TCP
connection, it should reply with `ConnectionRequestAccepted` and record in its
local state that it now has an established heavyweight with A. It may then
proceed to the main part of the protocol.

## Crossed Connection Request

As mentioned previously, the protocol tries to ensure that only one TCP
connection is used between any two endpoints at once. The typical case is that
an endpoint can simply determine if it has an existing heavyweight connection to
a peer because it either initiated it or received it and it knows if any
existing TCP connection is still open. The hard case arises when two endpoints
initiate establishing heavyweight connections to each other *at the same time*
(in the usual distributed systems sense of "same time").

Each endpoint will have recorded in its local state that it is in the process of
initiating a heavyweight connection to the other endpoint. Each endpoint will
send the connection request message as usual. When each endpoint accepts an
incoming TCP connection, it checks the peer endpoint ID from the connection
request message.

The additional rule is that it must lookup in its local state to see if a
connection to the peer endpoint was either 1. already *being* established
outbound or 2. already fully established. In the first case then we are in the
crossed connection situation. The second case can also occur legitimately (i.e.
not a protocol violation) when one peer has discovered that the existing TCP
connection has failed (i.e. its end is closed) and is trying to establish a new
TCP connection, while the other peer has not yet discovered that the existing
TCP connection is dead.

### Crossed Connection Situation

In the crossed connection situation, thus far this is completely symmetric
between endpoints, but we must break the symmetry to resolve which of the two
TCP connections to use, and which to close. The solution the protocol uses to
break the symmetry is that the endpoint addresses can be ordered
(lexicographically in their binary string form). Thus the rule each node must
use to decide whether to accept or reject the incoming connection request is:
reply with `ConnectionRequestAccepted` if the peer's endpoint ID is less than
the local endpoint id, and otherwise reply with `ConnectionRequestCrossed` and
close the TCP connection.

### Connection Dead / Re-establish Situation

In the second case, where the endpoint handling the incoming TCP connection has
determined that an established connection already exists between the two
endpoints, the protocol is as follows. A `ConnectionRequestCrossed` reply is
sent and the TCP connection is closed. Additionally, the endpoint tries to
validate the liveness of the existing connection, with the purpose of either
validating that it is live or determining that it is not in order to close the
dead connection (which will then allow opening a new one).

To validate the liveness, the endpoint sends a **ProbeSocket** message. If a
**ProbeSocketAck** message is not received within an implementation-defined time
period then the endpoint should close the TCP connection and update its local
state accordingly to enable a new connection to be established by either
endpoint.

An endpoint that receives a ProbeSocket message should reply with a
ProbeSocketAck.

The encoding for these messages is simple:

    +-------------+
    | ProbeSocket |
    +-------------+
    |    Int32    |

    +----------------+
    | ProbeSocketAck |
    +----------------+
    |     Int32      |

where the value for the control message headers are 4 and 5 respectively.

## Main Protocol

Once a heavyweight connection has been established between two endpoints then
the main part of the protocol begins.

The main protocol between two endpoints consists of sending/receiving a series
of messages: control messages and data messages. Each has a header to identify
the message and a body appropriate to the message type. The messages for the
main protocol are control messages to create and close lightweight connections,
and data messages for sending data on a lightweight connection.

Lightweight connections are unidirectional. There are independent sets of
lightweight connections in each direction of the TCP connection. The lightweight
connections in each direction are managed by the *sending* side. The receiving
side has no direct control over the allocation of lightweight connections.

Lightweight connections are identified by a Lightweight connection ID, which is
a 32-bit signed integer. Lightweight connection IDs must be greater than 1024.
Lightweight connection ID numbers should be used sequentially.

The control messages to create or close a lightweight connection simply identify
the lightweight connection ID that they act on. Similarly, data messages
identify the ID of the lightweight connection that the data is being sent on.

Messages for different connection ID can be interleaved arbitrarily (enabling
the multiplexing of the different lightweight connections). The only constraints
are the obvious ones: for any connection ID the sequence of messages must be a
create connection message, any number of data messages and finally a close
connection message.

The format of these messages is as follows:

    +-----------+-----------+
    | CreateCon |   LWCId   |
    +-----------+-----------+
    |   Int32   |   Int32   |

    +-----------+-----------+
    |  CloseCon |   LWCId   |
    +-----------+-----------+
    |   Int32   |   Int32   |

    +-----------+-----------+-------------------+
    |   LWCId   |    Len    |       Data        |
    +-----------+-----------+-------------------+
    |   Int32   |   Int32   |     Len-bytes     |

where:

-   CreateCon control header is 0;
-   CloseCon control header is 1;
-   LWCId is the lightweight connection id, which is &gt;= 1024.

The header Int32 is aliased between the control message headers and the
lightweight connection IDs of the data messages, which is why connection ids
must be 1024 or greater.

The data messages consist of the lightweight connection ID and a length-prefixed
frame of data. Implementations of this protocol may wish to impose a maximum
size on these data frames, e.g. to ensure reasonable multiplexing between
connections or for resource considerations.

Note that there need be no direct correspondence between these message
boundaries and reads/writes on the TCP socket or packets. It may make sense for
performance or network efficiency to arrange for a connection open, small data
message and connection close to be sent in a single write.

## Closing Heavyweight Connections

Cleanly closing the heavyweight connection is not trivial. This is because the
heavyweight connection should only be closed once lightweight connections in
both directions are closed. Given that the allocation of lightweight connections
is controlled independently by each endpoint then some synchronization is
required for both endpoints to agree that there are no more lightweight
connections in either direction.

When one endpoint determines that it has no more outgoing lightweight
connections, and the set of incoming connections it knows of is empty, then it
may initiate the protocol to close the heavyweight connection. It does so by
sending a **CloseSocket** message. The message carries the maximum incoming
lightweight connection ID seen by the endpoint: i.e. the highest connection ID
that has been allocated by the remote endpoint that has so far been seen by the
local endpoint. The local endpoint now updates the state it uses to track the
remote endpoint to note that it is now in the process of closing. If the local
endpoint now receives a create connection message from the remote endpoint,
while it has the remote endpoint marked as being in the process of closing then
it resets the state back to the normal connection established state. This
happens if the remote endpoint opened a new lightweight connection before it
received the close socket message, and so the attempt to close the socket should
be abandoned.

When an endpoint receives a **CloseSocket** message it checks its local state to
check the number of outbound lightweight connections and the maximum lightweight
connection ID it has used for outgoing connections. If there are still outbound
connections then the close socket message is ignored. Additionally, if the
maximum outbound lightweight connection ID used thus far by the local node is
higher than the one received in the close socket message then the close socket
message is ignored. This case can happen even if the number of outbound
connections is currently zero, if an outbound connection was created and then
closed prior to the close socket message arriving. In both cases what has
happened is that the heavyweight connection has become active again while one
side was trying to close it due to inactivity, and so it is appropriate to
abandon the attempt to close it.

If on the other hand there are no outbound connections and the last new
connection ID seen by the remote endpoint is the same as that locally, then both
sides agree and the TCP connection should be closed.

The message structure is:

    +-------------+-----------+
    | CloseSocket |   LWCId   |
    +-------------+-----------|
    |    Int32    |   Int32   |

where:

-   `CloseSocket` - close connection control message, value `2`;
-   `LWCId` - maximum lightweight connection ID used thus far.

## Flow Control and Back-pressure

Lightweight connections do not provide any flow control over and above what is
provided by TCP. The protocol does not provide any facility to reject incoming
lightweight connections. Any such facility must be layered on top, in the
application layer or another intermediate layer.

Implementations should consider the problem of back-pressure and head of line
blocking. Head of line blocking is a problem common to many protocols layered on
top of TCP, such as HTTP 1.x where one large response can "block" other smaller
responses for other URLs because the responses are sent in order. This problem
is less severe in this transport protocol because connection are multiplexed, so
small messages need not be blocked by large messages. Nevertheless, it is still
the case that the multiplexed stream of data for all connections must be
received in order: it is not possible to push back on one lightweight connection
vs another, only on the whole heavyweight connection.
