# Azure-Networking-Performance
Networking Performance and Best Practices that goes a lot deeper then anything you will find on Azure public docs.
# Intro
This article is going to discuss Azure networking performance across VPN and ExpressRoute. We will talk about what to expect, best practices, how to troubleshoot and tips. To summarize when talking about networking performance, you're only has fast as your lowest denominator. Taking Express Route as an example, you have the circuit bandwidth all up (Which includes both MSEEs ~VRFs), The gateway bandwidth, and the VMs from which you are running your workloads, sender and receiver. Any of those variables can be the bottleneck if peformance is not optimal. To summarize, the biggest factors that affect network performance are as follows:

1. Your RTT (Round Trip Time) This is the time it takes for a packet to go from sender to receiver and back to the sender, or simply the latency. This variable cannot be changed unless you move one or both closer.

2. The workload or application. Is the application single or multi-threaded? By that I mean, does it kick off a operation and that TCP stream has to finish before kicking off another operation? If so, that means the receiver has to wait until the data tranfer finishes to recieve more data. In a networking trace (More on that later) it would show only one TCP stream. If your application is multi-threaded, that means it can initiate multiple TCP streams at once to the receiver without having to wait until the operation finsihes.

3. TCP Window Size, ie scale factor. In the Windows TCP/IP Stack, TCP options include what we call scale factor. The receiver advertises to the sender how much data its allowed to recieve. After Server 2008, Windows TCP stack automatically adjusts the scale factor based on RTT and packet loss. Most mahcines offer a scale factor of x 8 (More on that later)

4. The size of the pipe, ie the bandwidth as we alluded above. Just because you have a 10GB circuit, does not mean you are going to get close to 10GB peformance. 

5. Packet loss -Generally in networking we say 1-2% packet loss is acceptable. Anything beyond that is going to have issues because the sender is going to retransmit packets which causes delay

Lets step through each of these variables and explain how they affect performance and what's normal and not normal.

# RTT
In this example, I simply went to www.bing.com with Wireshark and recorded the TCP handshake. That is, SYN--> SYN-ACK--->ACK. In That process Wireshark recorded the RTT or latency. You could also simply test thing by doing a Ping or TCPPIng etc. Here from my client machine to bing, RTT was very low 0.0069. Wireshark also nicely displays this for us :)
![image](https://user-images.githubusercontent.com/55964102/219761757-58225b62-01fa-4fd9-9920-179744c0e2cd.png)
![image](https://user-images.githubusercontent.com/55964102/219762323-651fd92c-b1b4-401f-a654-000fd775a118.png)

The RTT should not change from one TCP stream to the next or from one data transfer to the next. If it does, that means the packets could be taking a diffrent path, there is packet loss, or a middleware device is intercepting the packets and that can be checked with the TTL (Time to live field) more on that later. As we mentioned above, latency cannot be changed as this is simply the physical distance across all the hops from sender to receiver. Unless you can move the sender or receiver closer, you are bound by the RTT which dramatically affects performance. In terms of VPN and ExR, VPN traverses the public internet so you're subject to your ISP and BGP internet backbone routers. For ExR, that is a private path over MPLS and that pipe should be more stable since you're using a Telco like Equinix and its a private dedicated path. Every time a packet traverses a router, the TTL is decremented by one. Windows starts with a TTL of 128 and Linux starts with a TTL of 64. The easiest way to check the number of hops is to simply do a traceroute or tracert to the destination IP. That will tell you how many routers the packet traversed to reach its destination.
![image](https://user-images.githubusercontent.com/55964102/219769185-e99ddab3-6144-4307-bee0-90625c724e44.png)

Request time out simply means the router discards ICMP packets. The number of hops should not change between tests. If it does, that means you likely have asymetric routing or another middleware device intercepting packets. In this case, there are 8 hops between my laptop and the server hosting bing.com


# Application Workloads
There are a lot of scenarios I faced where an end user would complain about networking peformance using whatever application or transfer utility there were using. I would fire up Netmon or Wireshark and see only one TCP stream in the entire trace, meaning one TCP conversation. A filter that can be used is tcp.flags.syn==1. This will tell you how many TCP treams are in the trace file. Point in case, the quick capture I did on my machine, there were 5 steams:
![image](https://user-images.githubusercontent.com/55964102/219765752-fc70a20e-ffd6-47ea-86d5-079da99dd426.png)

Really, the only time you would want single threaded applications is for programming or debbugging and less overhead. Other then that, being able to kick off multiple TCP streams at once is going to yield a lot better performance. A simple way to measure that is using Iperf or NTttcp. If you run a test with only one thread (stream) you are going to get much poorer performance then running the test with 16, 32, or 64 threads! The client is going to have to wait until the server acknowledges all the data and the TCP stream is closed, before a new process and TCP stream can be kicked off. Great link here on threading and how it plays a factor into peformance tunning: https://learn.microsoft.com/en-us/dotnet/standard/threading/threads-and-threading

# TCP Window Scaling (Scale Factor)

In TCP/IP networking the TCP Window size is the receivers window that accepts data. Think of it as a bucket, and inside that bucket you can only fill so many tennis balls. There is also a send buffer, but that is not visible in the trace file. On the initital TCP handshake, both sender and receier advertise to each other their TCP receive windows. Just like a elluded above, both have to be the same. What I mean by that if sender is not doing window scaling, but receiver is, then window scaling will NOT be used in the data transfer (tcp stream). Both sides have to agree on the TCP options. You can easily view the TCP options in the three way handshake. Taking a look at the same TCP handsahke earlier we used for bing.com:

Client Machine (My Laptop)

![image](https://user-images.githubusercontent.com/55964102/219774321-0fb580dc-9052-412b-9869-1628d1bb02e1.png)

Bing.com (Server or Receiver)

![image](https://user-images.githubusercontent.com/55964102/219776495-4141b1cd-224d-4c1f-a601-316674b7c62e.png)

Since my laptop supports a scale factor x 8 (multiplier of 256), that is the lowest denomiator and what will be used during data transfer instead of Bing, which uses a multiplier of 2048! 

So both sender and reciever support TCP Window scaling. In TCP, the default TCP Window size is 65K Bytes. What happens during data transfer, every time a client sends data, the reciever has to ack tha that packet before it can take it out of its receiver buffer. If the buffer becomes full, ie the bucket with tennis balls fills up to quickly, the buffer becomes totally full. This is where scale factor comes in, its saying I can scale x 8 beyond the default 65k window size. If there is no scale factor, the TCP recieve buffer becomes totally full. That is called Zero Window or TCP Window Full and that will slow down data transfer. The client then has to sit there and wait until the sender empties its reciever buffer until more data can be transfered again. This is normally a peformance problem, ie the machine does not have enough compute, ram and processing power to keep up with the sender sending data. It simply cannot empty is buffer fast enough and that will cause pauses in the data transfer and peformance delays. 

As long as there is no packet loss in the data transfer, the recieve buffer will continue to scale up to its max size until the transfer is over. If there is packet loss, the recieve window shrinks down into what is called congestion avoidance until those missing packets are then acknowledged again. Again, packet loss and duplicate ACKS slow down data transfer. There are other TCP options like MSS (Maximum Segment Size) and SACK (Selective Acknowledgement) but those are beyond the scope of this article. Here is a great order article that is still relevant today on Window Scaling: https://learn.microsoft.com/en-us/previous-versions/technet-magazine/cc162519(v=msdn.10)?redirectedfrom=MSDN




