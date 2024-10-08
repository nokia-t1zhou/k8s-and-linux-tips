在 IPv6 中，原本的 IP 报文不支持路由器进行分片。与 IPv4 不同，IPv6 的设计理念取消了由中间路由器进行分片的功能，分片完全交由发送端处理。这种设计简化了路由器的工作，提升了网络的转发效率。

#### IPv6 不支持路由器分片
IPv4 支持中间路由器对过大的数据包进行分片，并将分片后的数据包发送到目的地再进行重组。但是这种机制会增加路由器的负担，降低网络性能。

IPv6 取消了这种机制，不允许中间路由器对数据包进行分片。取而代之的是，IPv6 要求发送端在发送之前确定路径上的最大传输单元（MTU），并根据 MTU 对数据进行分片。

#### IPv6 的分片由源节点处理
在 IPv6 中，如果一个数据包超过了路径上的最大传输单元（PMTU），需要由发送端进行分片。发送端使用 分片扩展头（Fragment Extension Header） 来对数据包进行分片。

IPv6 报文的分片过程如下：

- 发送端在发包前，通过路径 MTU 发现（Path MTU Discovery，PMTUD）机制确定路径上的最小 MTU。
- 发送端根据这个最小 MTU 对数据进行分片，并为每个分片添加分片扩展头（包含分片信息，如分片偏移量、标识符等）。
- 接收端在接收到所有分片后，负责重新组装这些数据包。
#### 分片扩展头（Fragment Extension Header）
在需要分片的情况下，IPv6 使用分片扩展头来标识每个分片。该扩展头包含以下关键字段：

Identification（标识符）：标识属于同一数据包的所有分片，接收端可以根据这个字段将分片重组。
Fragment Offset（分片偏移量）：表示当前分片在原始数据包中的位置。
M标志（More Fragments Flag）：指示是否有更多的分片。最后一个分片的 M 标志位为 0，其他分片的 M 标志位为 1。
#### PMTUD（路径 MTU 发现）
由于 IPv6 不支持路由器分片，发送端需要通过 PMTUD 机制来确定路径的最小 MTU，并根据此 MTU 对数据进行适当的分片。PMTUD 的基本步骤如下：

发送端开始时假定路径的 MTU 为链路的 MTU。
如果在传输过程中某个路由器发现数据包超过了其支持的 MTU，它会丢弃数据包并返回一个 ICMPv6 "Packet Too Big" 消息，通知发送端实际可用的 MTU。
发送端根据该消息调整数据包大小，确保数据包不会超过路径的最小 MTU。
#### 总结
在 IPv6 中，路由器不进行分片，所有的分片工作由发送端完成。
发送端通过 PMTUD 机制确定路径的最小 MTU，并根据该 MTU 对数据进行分片。
分片信息通过分片扩展头携带，接收端负责将分片重新组装。
这种设计使得 IPv6 的路由转发更加高效，减少了路由器在处理分片方面的开销，同时将更多的控制权交给了端点设备。
### Path MTU Discovery, PMTUD
路径 MTU 发现（Path MTU Discovery, PMTUD）是 IPv6 和 IPv4 都使用的一种机制，旨在确定从源端到目的端路径上可以承载的最大传输单元（MTU）。PMTU 发现的目标是避免数据包在路径中因为过大而被丢弃，从而确保数据能够顺利到达目的地。IPv6 尤其依赖 PMTU 机制，因为 IPv6 不允许路由器对数据包进行分片。

#### PMTU 的基本原理
PMTU 发现过程通过探测路径上的 MTU，并动态调整数据包的大小，使其适应路径上所有链路的最大传输单元。这个过程依赖于 ICMP 消息，特别是 "Packet Too Big" 消息。

#### 详细过程
PMTU 发现的具体步骤如下：

（1）初始探测
在发送数据时，源节点假定路径的初始 MTU 是发送端网络接口的 MTU（通常是 1500 字节，但可能因链路类型不同而不同，如以太网的典型 MTU 为 1500 字节）。
源节点根据这个假设的 MTU 构建并发送数据包。
（2）路径上的 MTU检查
数据包沿着路径通过多个路由器。在每个路由器处，该路由器会检查数据包的大小。
如果数据包大小超过了路径上某个链路的 MTU（例如，该链路的 MTU 只有 1400 字节），此时路由器会丢弃数据包，并返回一个 ICMPv6 "Packet Too Big" 消息给源节点。该消息包含路径上该链路的实际 MTU 值（例如 1400 字节）。
（3）调整 MTU
源节点收到 "Packet Too Big" 消息后，会将数据包的大小调整为返回的 MTU（在这个例子中是 1400 字节）。
源节点根据新 MTU 分片或调整后，重新发送数据包。
（4）重复探测
在数据包传输的整个过程中，源节点可能会多次接收到 "Packet Too Big" 消息，因为路径上可能有多个链路的 MTU 不同。源节点会继续调整数据包的大小，直到适应路径上的最小 MTU。
一旦源节点发现了路径上的最小 MTU，就会将该值用于后续的数据包传输，直到路径发生变化。
（5）路径变化与重新探测
如果网络路径发生变化，可能会出现新的最小 MTU。这时新的路由器可能会发送 "Packet Too Big" 消息，源节点会再次调整 MTU。
PMTU 的探测是一个动态过程，当路径发生变化时，PMTU 机制会自动重新调整以适应新的路径条件。
#### PMTU 的关键机制
ICMPv6 "Packet Too Big" 消息：这是 PMTU 发现的核心机制。当数据包过大无法通过某个链路时，路由器会丢弃该数据包，并发送此消息给源节点。消息中包含了该链路的实际 MTU，源节点据此调整数据包大小。

动态调整 MTU：PMTU 机制会根据返回的 MTU 动态调整数据包大小，确保后续的数据包能顺利通过路径上的所有链路。

#### PMTU 的优势
减少分片开销：IPv6 中不允许中间路由器进行分片，因此通过 PMTU 发现可以确保数据包在路径上不需要进行分片，提升了网络性能。
适应网络变化：PMTU 机制能够自动适应路径变化，保证数据包能在不同网络条件下顺利传输。
#### 潜在问题：ICMP 消息被过滤
在一些网络中，ICMP 消息可能会被防火墙或安全策略过滤，这会导致 PMTU 发现失败。为了应对这种情况，许多应用程序会实现超时和重传机制，以避免由于 MTU 问题导致数据传输失败。