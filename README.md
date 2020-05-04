# Chaos

**Burning up the CPU of a single docker container**

*OBSERVATIONS*
<p align="justify">After running the chaos_cpu.sh script on app1 docker container in greencanary vm, I get the following latency and CPU load output on running siege load for 30 seconds with the benchmark option enabled. I have run it on the stackless endpoint to generate a workload. Now since one of the containers inside the greencanary vm is under a heavy cpu load and the reverse proxy which load balances the requests to one of the three containers does not take cpu load into account; it still continues tp route requests to the overloaded container. As that overloaded container is unable to serve the requests, the responses get delayed leading to an increase in latency for the greencanary vm which can be seen in the graph. But for the blue vm this is not the case as none of the containers are under any heavy load, so the requests are evenly load balanced among the three containers and requests are served as it would have been. If you see the CPU load graph, you can see that the green vm is perpetually under a high CPU load due to the chaos_cpu.sh script running, but for the blue vm, the cpu load shoots up when it is under siege. We can develop a method to fix this issue of increased latency observed on the green vm by taking the cpu load into account when routing requests to a server. This can improve the performance to a great extent. </p>

> *SIEGE*
<p align="center">
<img src ="https://media.github.ncsu.edu/user/12214/files/aa3ef280-8d7a-11ea-92f4-30754a097aa0" width="400" height="500">
</p>

> *LATENCY*
<p align="center">
<img src ="https://media.github.ncsu.edu/user/12214/files/5da6e780-8d79-11ea-98ba-39c918c238f5" width="700" height="400">
</p>

> *CPU LOAD*
<p align="center">
<img src ="https://media.github.ncsu.edu/user/12214/files/68617c80-8d79-11ea-849e-503cab991518" width="700" height="400">
</p>

**Network traffic**

*OBSERVATIONS*
<p align="justify">After running the chaos_network_corruption.sh script on the green vm which allows the emulation of random noise introducing an error in a random position for 50% of the packets, we see that the latency for /work workload endpoint for the green shoots up. This is because the packet corruption leads to retransmission of packets since the receiver will drop the corrupted packet and order for a retransmission. Since TCP ensures reliable transmission of data, packet corruption will lead to more retransmissions in case on the green vm. This in turn leads to an increased latency to serve a request or to accept the packets after the proper retransmission has occured without any corruption in the data. Latency takes into account the round trip time and also the time for retransmissions until the packet is finally sent over to the receiver without any corrupted or loss of data. The blue vm latency to serve the requests is normal as expected without any glitch. </p>

> *LATENCY*
<p align="center">
<img src ="https://media.github.ncsu.edu/user/12214/files/00636400-8d82-11ea-9dd7-7cbf3332a542" width="700" height="400">
</p>

**Killing and starting containers**

*OBSERVATIONS*
<p align="justify">After stopping the two containers of the green vm, I ran the siege command to induce load on both green and blue vms for two times on the /stackless endpoint. The observations for both the times remain the same which is the latency for green vm is less as compared to that of the blue vm. This is because of two reasons namely now the load balancing or scheduling algorithm does not have to work extra to decide which container to send the request to even though it was a simple round robin algorithm. This impacts the latency to a great extent as there is only one container active now in the green vm the requests can go directly to it without going through an extra logic of distributing the load. The next reason is that now since there is only one container the load on the resources is less for the green vm as compared to the blue vm. As containers share the underlying operating system kernel and the resources, the blue still has more load but the green vm has a reduced load on the resources which leads to a better performance in serving the requests and hence a lower latency is observed. </p>

> *Killing the containers*
<p align="center">
<img src ="https://media.github.ncsu.edu/user/12214/files/95685c00-8d86-11ea-8d54-818832367857" width="800" height="150">
</p>

> *LATENCY*
<p align="center">
<img src ="https://media.github.ncsu.edu/user/12214/files/75d13380-8d86-11ea-87a0-9186c9e70567" width="700" height="400">
</p>

**Squeeze Testing**

*OBSERVATIONS*
<p align="justify">After rerunning the 2 containers of the green vm with restricted cpu and memory of the host kernel, I ran the siege command to induce load on the green as well as the blue vm on /stackless endpoint. I observed a higher latency in the green vm which was expected because of the reduced cpu access. I set the cpu of app1 and app2 such that they would have access to the host cpu for every 10% of a second. Also the memory access was set as 1mb. So with the limited resource access mainly the limited cpu time that the containers now have, they were prone to a higher latency or a delay in serving the requests because they do not have unlimited cpu access anymore. They have to wait until the cpu becomes available to them so that they can serve the request. The blue vm's containers operating with unlimited cpu and memory gives performance according to the expected values.</p>

> *Limiting the resources*
<p align="center">
<img src ="https://media.github.ncsu.edu/user/12214/files/6654e900-8d8c-11ea-9452-be21ae0927b0" width="900" height="200">
</p>

> *LATENCY*
<p align="center">
<img src ="https://media.github.ncsu.edu/user/12214/files/7b317c80-8d8c-11ea-82b6-17d2c5e9a3f7" width="700" height="400">
</p>

**Filling disks**

> *Filling the disk and trying to access any container*
<p align="center">
<img src ="https://media.github.ncsu.edu/user/12214/files/9e116000-8d8f-11ea-8948-3b4ce7191f42" width="500" height="300">
</p>
<p align="center">
<img src ="https://media.github.ncsu.edu/user/12214/files/aec1d600-8d8f-11ea-85a8-9b050698c928" width="500" height="300">
</p>

<p align="justify">On filling the disk inside one of the containers, we see the output of df -h to be 100% for the overlay filesystem and /dev/sda1 which is the underlying common filesystem shared by all three containers. So when I try to execute any command or try to create a file on any of the three containers I get an error saying failed to run console socket because there is no space left on the device. This happens when I try to access any of the three containers. Now after killing the container and starting it back again, the disk is free again. The reason why I was not able to access any of the containers because the disk space /dev/sda1 filesystem is shared among the containers. So filling the disk space in of the containers, will leave us with no access to any of the containers because to get inside a containers it creates a file in /tmp folder but since the disk space is full it won't be able to do so leaving us with the only option to kill and start the container again which would clear up the disk space.</p>


