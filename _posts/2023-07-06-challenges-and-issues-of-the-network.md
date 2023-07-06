---
layout: post
title: Challenging and issues of the network
subtitle: The issues that Network developers have to face in networked games
tags: [latency, jitter, packet loss]
comments: true
---

When developing an online multiplayer video game, it is necessary to understand the problems related to the network that need to be addressed.

## Latency:
The first challenge is latency, which is defined as the time it takes for information to be transmitted from the client's application layer to the server's application layer [26]. Latency is unidirectional, considering only the one-way path. If both the outgoing and return paths are taken into account, it refers to another term called round-trip time (RTT). This value is commonly known as "ping." Latency depends on several factors. Firstly, it is affected by the number of intermediate network nodes and the transmission speed of each node. Each time a packet passes through an intermediate node, it is affected by a nodal delay (D_nodal), which is the sum of various delays [27, 28]:

- Transmission delay (*D<sub>trans</sub>*): It is the time it takes for an originating node to send all the bits of a packet (consisting of its header *H*, and payload *p*) through the network. This time depends on the size of the packet H+p and the transmission speed *b*, commonly known as bandwidth, of the originating node, measured in bits per second (bps). ![Transmission delay formula](https://latex.codecogs.com/svg.image?D_{trans}=\frac{(H&plus;p)}{b}).

- Propagation delay (D_prop): It is the time it takes for information to travel from the originating node to the destination node. This time depends on the distance (d) the signal has to travel between the origin and destination and the propagation speed (s) of the medium. D_prop = d⁄s.

- Processing delay (D_proc): It is the time it takes for a node to process the packet. This time depends on the processing speed of the node.

- Queueing delay (D_queue): It is the time a packet spends in a buffer waiting to be processed by a node. This time depends on how busy the node is at that moment.

As a result, the nodal delay of an intermediate node can be calculated using the formula D_nodal = D_trans + D_prop + D_proc + D_queue.

Additionally, the packet size can also affect latency. The transmission delays of intermediate nodes will increase with larger packet sizes. Another issue with large packets is packet fragmentation. Each intermediate node has a maximum packet size it can accept, known as the Maximum Transmission Unit (MTU). When a node receives a packet larger than its MTU, it has two options: (i) fragment the packet or (ii) discard it [29]. In IPv4, this depends on the "Don't Fragment" bit in the packet header [30]. If fragmentation is required, multiple fragments are created, which are not reassembled until they reach the destination host. When a packet is fragmented, each resulting fragment contains a partial copy of the original IP datagram header, as shown in Illustration 17. This copy is partial because each fragment modifies certain fields within its copy, such as the Fragment Offset.

![IPv4 Packet Header](/assets/img/IPv4-packet-header.png)
Illustration 17 - IPv4 Packet Header

Packet fragmentation introduces several problems. Firstly, the more fragments generated, the higher the chance that one of them will be lost. If a fragment is lost and doesn't reach the destination, the entire packet is discarded. Secondly, the process of fragmentation incurs additional delays since the fragmentation and reassembly operations take physical time. Moreover, the fragments may arrive out of order, requiring them to be reordered at the destination host, thus increasing the assembly time.

Packet fragmentation only occurs in IPv4, as in IPv6, the packet is directly discarded. In IPv6, the source host must perform the fragmentation process to avoid future issues.

## Jitter:
Another network problem is known as latency fluctJitter:
Another network problem is known as latency fluctuation or jitter, which is defined as the variation in delays between packets [26]. This can be caused by different queueing delays resulting from network congestion. It can also be caused by the phenomenon of Bufferbloat [31], where a node in the network has an excessively large buffer to store packets. This phenomenon can be addressed by using Active Queue Management policies [32]. Illustration 18 shows a graphical representation of jitter and the formula to calculate it [26].

![Graphical representation of jitter](/assets/img/jitter-graphic-representation.jpg)
Illustration 18 - Graphical representation of jitter

Jitter results in increased latency and, in some cases, packet loss. This is because, in many instances, the variation has been so significant that the packet arrives too late and becomes obsolete (e.g., because a newer packet has arrived earlier). This can be addressed by using a Playout Delay Buffer.

## Packet Loss:
Packet loss is another problem that can be caused by various factors. As mentioned earlier, due to the large packet size, packet fragmentation increases the likelihood of losing fragmented packets. Additionally, as a result of jitter, obsolete packets may be automatically discarded by the destination host. Another possible cause is congestion. Congestion can cause intermediate node buffers to become saturated, leading to incoming packets being automatically discarded due to lack of capacity. This can occur not only when the buffer is full but also because of the Active Queue Management policies of the nodes, where some packets may be discarded before the buffer fills up.

Since packet loss is an inevitable problem, many multiplayer games implement techniques such as client-side Entity Extrapolation to hide these losses.