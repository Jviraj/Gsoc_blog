---
layout: default
---
# **Tcp implemetation in Linux Kernel**  
## Some important files.   
     .   
    ├──linux  
    │   ├──net  
    │      ├──ipv4  
    │         ├──tcp.c                        #Layer between user and kernel space.  
    │         ├──tcp_output.c                 #TCP output engine. Handles out-going data and passes it to network layer  
    │         ├──tcp_input.c                  #TCP input engine. Handles incoming segments.  
    │         ├──tcp_timer.c                  #TCP timer handling.  
    │         ├──tcp_ipv4.c                   #IPv4 related functions, receives segments from network layer.  
    │         ├──tcp_cong.c                   #Congestion control handler, includes also TCP Reno implementation.  
    │         ├──tcp_[veno/westwood/vegas].c  #Congestion control algorithms, named as tcp NAME.c.  
    │   └──include  
           ├──net  
              ├──tcp.h                        #Main header files of TCP. struct tcp sock is defined here.  
           
## Packet Reception.
When a Packet arrives at _NIC_ (network interface controller), the _DMA_ (direct memory access) is invoked and the packet is placed in kernel memory via empty _sk_buffs_ in a ring buffer called _rx_ring_. An incoming packet is dropped when the ring buffer is full.  
After receiving the packet successfully, the NIC raises an interrupt to the CPU, which processes each incoming packet and passes it to the IP layer. The IP layer does its own processing and passes it on to the TCP layer if it is a TCP packet. The TCP process is then scheduled to handle received packets.   Finally, the packet is stored inside the TCP recv buffer. **A critical parameter** for tuning TCP is the size of recv buffer at the receiver. The number of packets a TCP sender is able to have outstanding (unacknowledged) is the minimum of the congestion window (cwnd) and the receiver’s advertised window (rwnd).  
If the size of the recv buffer is smaller than the _bandwidthdelay product_ (BDP) of the end-to-end path, the achievable throughput will be low. On the other hand, a large recv buffer allows a correspondingly large number of packets to remain outstanding, possibly exceeding the number of packets an end-to-end path can sustain. (The size of the recv buffer can be set by modifying the /proc/sys/net/ipv4/tcp_rmem variable).  

## Packet Transmission  
The maximum size of the congestion window is related to the amount of send buffer space allocated to the TCP socket. The send buffer holds all outstanding packets (for potential retransmission) as well as all data queued to be transmitted. Therefore, the congestion window can never grow larger than send buffer can accommodate. If the send buffer is too small, the congestion window will not fully open, limiting the throughput. On the other hand, a large send buffer allows the congestion window to grow to a large value. If not constrained by the _TCP_recv_buffer_, the number of outstanding packets will also grow as the congestion window grows, causing packet loss if the end-to-end path cannot hold the large number of outstanding packets. (The size of the send buffer can be set by modifying the /proc/sys/net/ipv4/tcp wmem variable, which also takes three different values, i.e., min, default, and max.)

## Data Structure
* struct tcp sock: (include/linux/tcp.h) is the core structure of TCP. It contains all the information and packet buffers for certain TCP connection.
* struct inet connection sock: (include/net/ inet connection sock) is a socket type one level down from the tcp sock. It contains information about protocol congestion state, protocol timers and the accept queue.
* struct inet sock: (include/net/inet sock.h). It has information about connection ports and IPaddresses.
* struct sock: It contains two of TCP’s three receive queues, sk receive queue and sk backlog, and also queue for sent data, used with retransmission.
