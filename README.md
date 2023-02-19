# Azure-Networking-Performance
Networking Performance and Best Practices that goes a little deeper on TCP/IP
# Intro
This article is going to discuss Azure networking performance across VPN and ExpressRoute. We will talk about what to expect, best practices, how to troubleshoot and tips. To summarize when talking about networking performance, you're only has fast as your lowest denominator. Taking Express Route as an example, you have the circuit bandwidth all up (Which includes both MSEEs ~VRFs), The gateway bandwidth, and the VMs from which you are running your workloads, sender and receiver. Any of those variables can be the bottleneck if peformance is not optimal. To summarize, the biggest factors that affect network performance are as follows:

•  Your RTT (Round Trip Time) This is the time it takes for a packet to go from sender to receiver and back to the sender, or simply latency. This variable cannot be changed unless you move one or both closer.

•  The workload or application. Is the application single or multi-threaded? By that I mean, does it kick off a operation and that TCP stream has to finish before kicking off another operation? If so, that means the receiver has to wait until the data tranfer finishes to recieve more data. In a networking trace (More on that later) it would show only one TCP stream. If your application is multi-threaded, that means it can initiate multiple TCP streams at once to the receiver without having to wait until the operation finsihes.

•  TCP Window Size, ie window scale factor. In the TCP/IP Stack, TCP options include what we call window scale factor. The receiver advertises to the sender how much data its allowed to recieve during transfer. After Server 2008, Windows TCP stack automatically adjusts the scale factor based on smooth RTT and packet loss. Most mahcines offer a scale factor of x 8 (More on that later).

•  The size of the data pipe, ie the bandwidth as we alluded above. Just because you have a 10GB circuit, does not mean you are going to get close to 10GB peformance. This also comes into play with the VMs and Gateways the workloads are running over. 

•  Packet loss -Generally in networking we say 1-2% packet loss is deemed acceptable. Anything beyond that is going to have issues because the sender is going to retransmit packets without ackknowledgements which causes delay in transfer.

Lets step through each of these variables and explain how they affect performance and what's normal and not normal.

# RTT
In this example, I simply went to www.bing.com with Wireshark and recorded the TCP handshake. That is, SYN--> SYN-ACK--->ACK. In That process Wireshark recorded the RTT or latency. You could also simply test thing by doing a Ping or TCPIng etc. Here from my client machine to bing, RTT was very low 0.0069. Wireshark also nicely displays this for us :)
![image](https://user-images.githubusercontent.com/55964102/219761757-58225b62-01fa-4fd9-9920-179744c0e2cd.png)
![image](https://user-images.githubusercontent.com/55964102/219762323-651fd92c-b1b4-401f-a654-000fd775a118.png)

The RTT should not change from one TCP stream to the next or from one data transfer to the next. If it does, that means the packets could be taking a diffrent path, there is packet loss, or a middleware device is intercepting the packets and that can be checked with the TTL (Time to live field) more on that later. As we mentioned above, latency cannot be changed as this is simply the physical distance across all the hops from sender to receiver. Unless you can move the sender or receiver closer, you are bound by the RTT which dramatically affects performance. In terms of VPN and ExR, VPN traverses the public internet so you're subject to your ISP and BGP internet backbone routers. For ExR, that is a private path over MPLS and that pipe should be more stable since you're using a Telco like Equinix and its a private dedicated path. Every time a packet traverses a router, the TTL is decremented by one. Windows starts with a TTL of 128 and Linux starts with a TTL of 64. The easiest way to check the number of hops is to simply do a traceroute or tracert to the destination IP. That will tell you how many routers the packet traversed to reach its destination.
![image](https://user-images.githubusercontent.com/55964102/219769185-e99ddab3-6144-4307-bee0-90625c724e44.png)

Request time out simply means the router discards ICMP packets. The number of hops should not change between tests. If it does, that means you likely have asymetric routing or another middleware device intercepting packets. In this case, there are 8 hops between my laptop and the server hosting bing.com


# Application Workload
There are a lot of scenarios I faced where an end user would complain about networking peformance using whatever application or transfer utility there were using. I would fire up Netmon or Wireshark and see only one TCP stream in the entire trace, meaning one TCP conversation. A filter that can be used is tcp.flags.syn==1. This will tell you how many TCP treams are in the trace file. Point in case, the quick capture I did on my machine, there were 5 TCPsteams:
![image](https://user-images.githubusercontent.com/55964102/219765752-fc70a20e-ffd6-47ea-86d5-079da99dd426.png)

Really, the only time you would want single threaded applications is for programming or debbugging and less overhead. Other then that, being able to kick off multiple TCP streams at once is going to yield a lot better performance. A simple way to measure that is using Iperf or NTttcp. If you run a test with only one thread (stream) you are going to get much poorer performance then running the test with 16, 32, or 64 parallel threads! The client is going to have to wait until the server acknowledges all the data and the TCP stream is closed, before a new process and TCP stream can be kicked off. Great link here on threading and how it plays a factor into peformance tunning: https://learn.microsoft.com/en-us/dotnet/standard/threading/threads-and-threading

# TCP Window Scaling (Scale Factor)

In TCP/IP networking the TCP Window size is the receivers window buffer size that accepts data. Think of it as a bucket, and inside that bucket you can only fill so many tennis balls. There is also a send buffer, but that is not visible in the trace file. On the initital TCP handshake, both sender and receier advertise to each other their TCP receive windows. What I mean by that if sender is not doing window scaling, but receiver is, then window scaling will NOT be used in the data transfer (tcp stream). Both sides have to agree on the supported TCP options. You can easily view the TCP options in the three way handshake. Taking a look at the same TCP handsahke earlier we used for bing.com:

Client Machine (My Laptop)

![image](https://user-images.githubusercontent.com/55964102/219774321-0fb580dc-9052-412b-9869-1628d1bb02e1.png)

Bing.com (Server or Receiver)

![image](https://user-images.githubusercontent.com/55964102/219776495-4141b1cd-224d-4c1f-a601-316674b7c62e.png)

Since my laptop supports a scale factor x 8 (multiplier of 256), that is the lowest denomiator and what will be used during data transfer instead of Bing, which uses a multiplier of 2048! If there was no window scale at all, it would be zero and the recieve buffer would only be 65K. You tend to see this on older machines, ie before 2008 or aging mainframe systems etc.

So both sender and reciever support TCP Window scaling in my example. In TCP, the default TCP Window size is 65K Bytes as mentioned above. What happens during data transfer, every time a client sends data, the reciever has to ack that that packet before it can take it out of its receiver buffer. If the buffer becomes full, ie the bucket with tennis balls fills up to quickly, that is a problem. This is where scale factor comes in, it's saying I can scale x 8 beyond the default 65k window size. If there is no scale factor, the TCP recieve buffer becomes totally full. That is called "Zero Window or TCP Window Full" and that will slow down data transfer. The client then has to sit there and wait until the recevier empties its buffer until more data can be transfered again. This is normally a peformance problem, ie the receiving machine does not have enough compute, ram and processing power to keep up with the sender sending data. It simply cannot empty is buffer fast enough and that will cause pauses in the data transfer and peformance delays. 

As long as there is no packet loss in the data transfer, the recieve buffer will continue to scale up to its max size until the transfer is over. If there is packet loss, the recieve window shrinks down into what is called congestion avoidance until those missing packets are then acknowledged again. Once the missing data is acknowledged the buffer will continue to expand again. Missing packets and duplicate acks are a problem during data transfer and slow the process. There are other TCP options like MSS (Maximum Segment Size) and SACK (Selective Acknowledgement) but those are beyond the scope of this article. Here is a great order article that is still relevant today on Window Scaling: https://learn.microsoft.com/en-us/previous-versions/technet-magazine/cc162519(v=msdn.10)?redirectedfrom=MSDN

# Dataplane Pipe Bandwidth

With VPN, you are bound by your gateway size and RTT, including encryption cipher suites which make a difference. In Azure, way say maximum VPN throughput you can get is 1.25Gbps per tunnel. If you use multiple tunnels the max you can get is 10Gbps, but that is using lower overhead cipher suites and encryption. For ExpressRoute, you have the gateway throughput and underlining circuit bandwidth. So, if you have 10Gbps circuit, but your ExR GW is only Standard SKU, your bound by your gateway throughput and that is the lowest denominator. For Expressroute, as long as the gateway is ultra sku, can you enable fastpath which bypasses the data for inbound flows for the dataplane. That would mean you're only bound by the circuit throughput and RTT. The common bottlenecks we see on the GW is PPS, TCP Flows and high CPU. Once the gateway becomes overloaded, it will start discarding packets. The same goes for the circuit, once its overloaded, it will eventually drop and discard packets. Another analogy, think of straw. If you have a tiny straw but are trying to push a ton of data, the straw is the limiter. You can also have a great big straw and underutilize the pipe. Fortunately, upgrading the GW and circuit is relatively easy as long as its not a direct port circuit. I also want to call out the VMs. The machines can also be the bottleneck, as we mentioned above with Window Scale. The other issue is that machines really have two network bottlenecks. Disk network throughput and overall NIC VM throughput. If the VM goes over the either, it will be throttled. We will throttle the disk in Azure (the IOs) and if network throughput is breached on the NIC, Azure fabric will throttle the VM. Ideally, all VMs should be running accelerated networkng (SR-IOV) which bypasses the virtual switch and sends the traffic directly to the physical NIC. Also, any machine today should be using receive side scaling or RSS. Windows Server 2012 and above uses RSS, but that can also be manually tunned with PowerShell. Public Articles on VM tunning and RSS:
https://learn.microsoft.com/en-us/windows-hardware/drivers/network/introduction-to-receive-side-scaling
https://learn.microsoft.com/en-us/azure/virtual-network/virtual-network-tcpip-performance-tuning

# Packet Loss

Generally in TCP/IP, any packet loss over 2% will start affecting performance. TCP is a connection oriented protocol, so it's designed to retransmit missing data. However, retransmissions take time, cause delays in large numbers. Once the path has loads of transmissions or duplicate ACKS, the peformance will take a hit. This is more of a problem on VPN as the path is over the internet as opposed to ExR, but both can happen. So as long as the packet loss is below that number, TCP should be able to handle this without major notice. You can check packet loss in Wireshark using tcp.analysis.retransmission or tcp.analysis.flags to check all bad TCP issues. 

# Formulas and Links to Check Performance
```bash
# Calculate Maximum TCP throughput
TCP-Window-Size-in-bits / Latency-in-seconds = Bits-per-second-throughput (bps)

# Calculate optimal TCP Window Size
Bandwidth-in-bits-per-second * Round-trip-latency-in-seconds = TCP window size in bits / 8 = TCP window size in bytes!

# Calculate maximum latency for desired throughput
TCP-window-size-bits / Desired-throughput-in-bits-per-second = Maximum RTT Latency!
```

Bandwidth Delay Product
The Bandwidth Delay Product (BDP) is a simple formula used to calculate the maximum amount of data that can exist on the network (referred to as bits or bytes in flight) based on a link’s characteristics:
• Bandwidth (bps) x RTT (seconds) = bits in flight
• Divide the result by 8 for bytes in flight = BDP
• So…….bps x RTT/8=BDP

If the BDP (in bytes) for a given network link exceeds the value of a session’s TCP window, then the TCP session will not be able to use all of the available bandwidth; instead, throughput will be limited by the receive window (assuming no other constraints, of course).
The BDP can also be used to calculate the maximum throughput (“bandwidth”) of a TCP connection given a fixed receive window size:
• Bandwidth(Throughput) = (window size *8)/RTT
Remember 1 mbps = 1,000,000 bps (bits per second)
Remember 1 gbps = 1,000 mbps (bits per second)

*If you want to use an automated third party site to check peformance, I recommend wintelguy:*
https://wintelguy.com/wanperf.pl

# Tips
• For ExpressRoute, always peer as closet to where your workloads will be running to reduce RTT
<br>
• For VPN and ExR Gateways, make sure you're monitoring bits in/out, PPS and CPU to make sure the gateway is correctly sized for your workloads. ExpressRoute scalable gateway is coming, but its not ready yet at the time of this article.
<br>
• For the circuit throughput, remember its both VRFs, so MSEE/MSEE2. You can always size up assuming there is sufficent capacity, but you CANNOT size down without a delete and re-create.
<br>
• Always run fastpath if you can for ExpressRoute gateways, as that eliminates the ingress path for the dataplane
<br>
• Make sure the VMs Disks are Ultra when possible and also make sure Accelerated Networking is enabled when possible to bypass the vSwitch
<br>
• Make sure both machines are using TCP Window Scaling, and receive side scaling is being used (RSS)
<br>
• If the application supports parallel threads and operations, use it as that will yeild much better peformance

# Other Useful Links
• https://infrastructuremap.microsoft.com/explore
<br>
• https://learn.microsoft.com/en-us/azure/networking/azure-network-latency
<br>
• https://learn.microsoft.com/en-us/azure/virtual-network/virtual-machine-network-throughput
<br>
• https://learn.microsoft.com/en-us/azure/virtual-network/virtual-network-tcpip-performance-tuning
<br>
• https://learn.microsoft.com/en-us/azure/expressroute/expressroute-about-virtual-network-gateways
<br>
• https://learn.microsoft.com/en-us/azure/vpn-gateway/vpn-gateway-about-vpngateways
<br>
• https://learn.microsoft.com/en-us/azure/expressroute/expressroute-circuit-peerings
<br>
• https://learn.microsoft.com/en-us/azure/expressroute/expressroute-erdirect-about


# Summary
From the above points, I wanted to touch on the various factors that affect network peformance in Azure. As we can see there are many factors that contribute to the peformance one would get over VPN and Expressroute. Some of them have knobs we can easily turn, ie updating the GW if its overloaded, updating the circuit bandwidth. The other knobs we cannot turn easily, changing the RTT or physical distance between sender and reciever, eliminating potential packet loss scenarios, Changing the path the pakets take unless we have the ability to add or remove routers along the way. You can also tune the applicatin if possible, or make sure you're following best practices in terms of enabling accelerated networking on the VM based on VM size. You also want to make sure your disks can handle the amount of IOs you pushing.





