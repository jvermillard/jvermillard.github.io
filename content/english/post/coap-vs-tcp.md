+++
author = "Julien Vermillard"
title = "Transferring files over CoAP"
tags = [ "Internet-of-Things", "UDP", "CoAP", "LPWA"]
date = "2022-09-24"
+++

For constrained IoT applications, you want to limit the number of protocols and network stacks to implement. Having multiple stacks means more flash and memory usage, and protocols are infamous consumers of RAM buffers.

If you are using low power, low bandwidth IP transport, you probably settled on something UDP-based like [CoAP](https://www.rfc-editor.org/rfc/rfc7252). It is a pretty simple standard, replacing the need for TCP and HTTP in a single protocol. It can transfer payload larger than the network MTU (maximum transmission unit) using the blockwise transfer extension ([RFC7959](https://www.rfc-editor.org/rfc/rfc7959)), but, as we will see, this mechanism trade-off link optimization for simplicity.

First, let’s see how exactly CoAP block transfer works: It sends a block of a maximum of 1024 bytes and waits for the acknowledgment before transmitting the next one.

![Impact of network latency on CoAP blockwise transfer](/images/coapblock.png)

Imagine you are sending a 1Mbytes payCoAPload over a network with a roundtrip time of 100ms (0.1s) and a typical MTU of 1500bytes. With a CoAP payload of 1024 bytes per packet, we will need to send 1000 packets and, between each transmission, wait for the acknowledgment.

So we will need to wait for 1000*0.1s => 1 minute and 40 seconds for 1 Megabyte! It’s the equivalent of ~8kbits per second. We are far from being able to saturate the theoretical maximum bandwidth of NB-IoT (26kbps) or LTE-M (300kbps).

Now, let’s take a look at how TCP deals with transferring significant content on high latency networks: it uses a window system to control the number of data “in-flight.” It means the data or packets sent to the peers but not yet acknowledged. It helps TCP to send multiple packets without waiting for acknowledgment and helps to saturate the bandwidth.

![TCP slow start: the “in-flight” window progressively opens for saturating the bandwidth](/images/tcp.png)

Indeed, this method is more efficient in saturating the bandwidth, but this relies on the capacity of the receiving part to buffer and reorder the message, which consumes RAM and is often a problem for the most constrained microcontrollers.
