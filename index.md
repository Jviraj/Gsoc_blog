## **Tcp implemetation in Linux Kernel**  
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
              
