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
           
* * *

### Packet Reception.
When a Packet arrives at _NIC_ (network interface controller), the _DMA_ (direct memory access) is invoked and the packet is placed in kernel memory via empty _sk_buffs_ in a ring buffer called _rx_ring_. An incoming packet is dropped when the ring buffer is full.  
After receiving the packet successfully, the NIC raises an interrupt to the CPU, which processes each incoming packet and passes it to the IP layer. The IP layer does its own processing and passes it on to the TCP layer if it is a TCP packet. The TCP process is then scheduled to handle received packets.   Finally, the packet is stored inside the TCP recv buffer. **A critical parameter** for tuning TCP is the size of recv buffer at the receiver. The number of packets a TCP sender is able to have outstanding (unacknowledged) is the minimum of the congestion window (cwnd) and the receiver’s advertised window (rwnd).  
If the size of the recv buffer is smaller than the _bandwidthdelay product_ (BDP) of the end-to-end path, the achievable throughput will be low. On the other hand, a large recv buffer allows a correspondingly large number of packets to remain outstanding, possibly exceeding the number of packets an end-to-end path can sustain. (The size of the recv buffer can be set by modifying the /proc/sys/net/ipv4/tcp_rmem variable).  

* * *

### Packet Transmission  
The maximum size of the congestion window is related to the amount of send buffer space allocated to the TCP socket. The send buffer holds all outstanding packets (for potential retransmission) as well as all data queued to be transmitted. Therefore, the congestion window can never grow larger than send buffer can accommodate. If the send buffer is too small, the congestion window will not fully open, limiting the throughput. On the other hand, a large send buffer allows the congestion window to grow to a large value. If not constrained by the _TCP_recv_buffer_, the number of outstanding packets will also grow as the congestion window grows, causing packet loss if the end-to-end path cannot hold the large number of outstanding packets. (The size of the send buffer can be set by modifying the /proc/sys/net/ipv4/tcp wmem variable, which also takes three different values, i.e., min, default, and max.)

* * *

### Data Structure
* **struct tcp sock:**
(include/linux/tcp.h) is the core structure of TCP. It contains all the information and packet buffers for certain TCP connection.
* **struct inet connection sock:**
(include/net/ inet connection sock) is a socket type one level down from the tcp sock. It contains information about protocol congestion state, protocol timers and the accept queue.
* **struct inet sock:**
(include/net/inet sock.h). It has information about connection ports and IPaddresses.
* **struct sock:**
It contains two of TCP’s three receive queues, sk receive queue and sk backlog, and also queue for sent data, used with retransmission.

* **Data Queues:**
The socket will be marked as being in use to avoid conflicts. However, incoming segments must be saved even when the socket is in use. Therefore, socket has several queues for incoming data, receive queue, pre-queue, and backlog queue. out-of-order queue is used as temporary storage for segments arriving out of order.
  1. Receive queue: when segment arrives but the user is not waiting for the data, the data is immediately processed and copied to the receive queue. Data will be copied to user’s buffer when application reads data from the socket.
  2. Pre-queue: If user is using blocking IO (_With blocking I/O, when a client makes a request to connect with the server, the thread that handles that   connection is blocked until there is some data to read, or the data is fully written. Until the relevant operation is complete that thread can do nothing else but wait._) and the receive queue does not have as many bytes as requested, will the socket be marked as waiting for data. When the new segment arrives, it will be put to pre-queue and waiting process will be awakened. Then the data will be handled and copied to user’s buffer.
  3. Backlog queue: If user is handling segments (socket is marked as being in use) at the same time when we receive a new one, it will be put to the backlog queue, and user context will handle the segment after it has handled all earlier segments from other queues.

* **struct sk_buff:**
(located in include/linux/skbuff.h). It is a socket buffer containing one slice of the data we are sending or receiving. If the data does not fit then it will be fragmented to smaller segments and saved inside struct skb_shared_info that lives at the end of data (at the end of the pointer).

One large segment cannot hold all the data therefore, we must have several socket buffers to be able to handle major amounts of data and to resend the data that was lost during transmission to receiver. In Linux these queues are implemented as double linked ringlists of sk_buff structures. Each socket buffer has a pointer to the previous and next buffers. There is also special data structure to represent the whole list, known as struct sk buff head, that is used to indicate the first and the last members of ring list.

* **Hash tables:**
  A data structure that is used to map a given key to the corresponding value.  
Sockets are located in kernel’s hash table from where them are fetched when a new segment arrives or socket is otherwise needed.  
Main hash structure is struct inet_hashinfo (include/net/inet hashtables.h), and TCP uses it as a type of global variable tcp_hashinfo located in net/ipv4/tcp ipv4.c  
struct inet_hashinfo has three main hash tables: One for sockets with full identity, one for bindings and one for listening sockets.  

* **Other data structures and features**
  1. struct proto: (include/net/sock.h) is a general structure presenting transmission layer to socket layer. It contains function pointers that are set to TCP specific functions in net/ipv4/ tcp-ipv4.c, and applications function calls are eventually, through other layers, mapped to these.  
  2. struct tcp_info: is used to pass information about socket state to user. Structure will be filled in function tcp_get_info(). It contains values for connection state (Listen, Established, etc), congestion control state (Open, Disorder, CWR, Recovery, Lost), receiver and sender MSS, rtt and various counters.  
  3. struct_inet_connection_sock: Retransmit, delayed ack and zero window probe timers are located in struct inet_connection_sock.  

* * *

### Socket Initialization
TCP functions are set to struct proto in tcp_ipv4.c. This structure will be held in struct inet_protosw in af_inet.c, from where it will be fetched and set to sk->sk_prot when user does socket() call. During socket creation in the function inet_create() function sk->sk_prot->init() will be called, which points to tcp_v4_init_sock(). From there the real initialization function tcp_init_sock() will be called.  
Address-family independent initialization of TCP socket occurs in tcp_init_sock() (net/ipv4/tcp.c). The function will be called when socket is created with socket() system call.  
* **Connection Socket:** Next step is to call connect(). In case of TCP, it maps to function inet_stream_ connect(), from where sk- >sk_prot->connect() is called. It maps to TCP function tcp_v4_connect().
tcp_v4_connect() validates end host connection by using ip_route_connnect() function. After that inet_hash_connect() will be called. Inet_hash_connect() selects the source port for our socket, if not set, and adds the socket to hash tables. If everything is fine then sequence number will be fetched from secure_tcp_sequence_number() and the socket is passed to tcp_connect().
tcp_connect() calls first tcp_connect_init(), that will initialize parameters used with TCP connection, such as maximum segment size (MSS) and TCP window size. After that tcp_connect() will reserve memory for socket buffer, add buffer to sockets write queue and passes buffer to function tcp_transmit_skb(), that builds TCP headers and passes data to network layer. Before returning tcp connect() will start retransmission timer for the SYN packet. When SYN-ACK packet is received, state of socket is modified to ESTABLISHED, ACK is sent and communication between nodes may begin.
 
* **Listening Socket:** bind() must be called to pick up port what will be listened to. Then listen() must be called. Bind() maps to inet_bind(). This function validates port number and socket, and then tries to bind the wanted port. If everything goes right, 0 will be returned otherwise error code indicating the problem will be returned. Listen() will become to function inet_listen(). This function will perform a few checks, and then calls function inet_csk_listen_start(), which allocates memory for the socket accept queue, sets socket to TCP_LISTEN and sets socket to TCP_LISTEN and adds socket to TCP hash table to wait for incoming connections.
