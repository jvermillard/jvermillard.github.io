+++
author = "Julien Vermillard"
title = "CoAP and Lightweight M2M challenges with NAT gateways"
tags = [ "Internet-of-Things", "UDP", "CoAP", "LPWA"]
date = "2022-09-07"
+++
With the rise of low power wide area (LPWA) networks, we see a rising interest in UDP-based protocols, but trading TCP for UDP has its own class of challenges.

After reading this pretty nice explanation from Tailscale on how they deal with NAT traversal for their VPN service (see: https://tailscale.com/blog/how-nat-traversal-works/) I decided to take the time to write down this little explanation on the specific case of CoAP and NAT gateways.

First, let’s focus on what NAT gateways are and what they are doing. They are allowing a device in a private network to communicate even without a public address.

![The NAT gateway receives the packet](/images/udp1.png)

![The NAT gateway saves a session and rewrites the message with its public IP](/images/udp2.png)



The NAT replaces the private source IP with its public IP. So, the target (server) will not see the original private IP but the IP of the NAT gateway.

![The NAT gateway can send back the response using the session](/images/udp3.png)

Now, the core of the issue: to process the response from the server, the NAT gateway sets some sessions to remember the origin of the request.

When the answer packet comes back, the NAT gateway finds the session and changes the destination of the packet to send it to the private IP of the client.

Now why use this complex system? First, to avoid assigning a public IPv4 address to each communicating device which is a scarce resource. Also, to a lesser extent, to prevent your device from being directly addressable by any device on the Internet, either for security reasons or for selling you a public IP for a premium.

# The impact of UDP/CoAP protocols?

For now, nothing here is specific to CoAP or UDP. And it’s true, the problem is more related to how the gateway handles TCP and UDP differently.

Indeed, session timeouts are configured differently for TCP and UDP. For TCP, on a typical mobile network, the inactivity timeout is around 20 minutes. For UDP it’s about 30 to 180 seconds! So if there is no communication for this amount of time, the session will expire and the server will not be able to reach the original client.

This definitively causes issues when you try to achieve low server-to-device latency.

## The solution to server-to-client latency

One obvious solution is to have the device send heartbeat messages frequently enough to keep the NAT session alive. But transmitting a packet every 30 seconds or so can have a huge price in bandwidth and power cost.

The other category of solution is to remove the NAT from the picture. One way to achieve that is to move the server and the client to the same private network. It’s what you usually do in a mobile network by using a private APN (Access Point Network). If you integrate your server within the APN you will have direct client addressing from the server and you will not suffer from the NAT timeout. Another way to remove the NAT is to use IPv6 where NAT should not exists. I have little direct experience with this kind of setup but in theory, it should remove the NAT issue.

Then there is more tradeoff you can do. One is to embrace the server-to-client latency by queuing any server-to-device message and waiting for the client to reconnect cyclically. Of course, the latency will not improve but the system will work. Finally, you could also switch to a TCP-based version of CoAP or another protocol, but you will lose the benefits of it (power consumption, lightness of the embedded stack).

## More optimization: DTLS session re-establishment

The UDP NAT timeout has another impact, but this time on DTLS. This security layer, used on top of UDP, identifies the session of a packet by using the source IP + port. So when a NAT session timeout, you don’t have any guarantee to get the same source port assigned by the NAT gateway. So that does mean, if after 30 seconds of inactivity a client wants to send a packet to the server, it must restart the session by doing a handshake (ideally an abbreviated one).

Again, this has an impact on your data and power consumption. One lesser known, and lesser implemented, DTLS extension solves this issue by adding a unique session identifier in the packet (see RFC9146 https://datatracker.ietf.org/doc/rfc9146/) in place of relying on the source port and IP address. A similar solution is used by QUICC/HTTP3 which are also UDP-based protocols.








