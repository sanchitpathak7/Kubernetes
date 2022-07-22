# Pod to Pod Communication between Nodes with Calico

### 3 Node Cluster
```
$ kubectl get nodes
NAME             STATUS   ROLES    AGE   VERSION
10.128.146.244   Ready    master   91d   v1.20.15
10.128.146.71    Ready    master   91d   v1.20.15
10.128.147.235   Ready    master   91d   v1.20.15
```

#### Standalone pods deployed on 2 different nodes.
```
$ kubectl get pods -o wide
NAME                   READY   STATUS    RESTARTS   AGE     IP              NODE             
multi-container-pod    2/2     Running   0          4h42m   10.20.225.119   10.128.146.244
multi-container-pod2   2/2     Running   0          3m49s   10.20.127.150   10.128.146.71
```

#### For Pod multi-container-pod on Node 10.128.146.244
```
$ sudo docker ps | grep multi
eabab459f3f5        nginx                                              "/docker-entrypoint.…"   5 hours ago         Up 5 hours                              k8s_container-2_multi-container-pod_default_79243492-822f-49f4-9690-1f31a6a209b6_0
7070e27637e9        busybox                                            "/bin/sh -c 'sleep 1…"   5 hours ago         Up 5 hours                              k8s_container-1_multi-container-pod_default_79243492-822f-49f4-9690-1f31a6a209b6_0
954f6b2320f9        localhost:5100/pause:3.2                           "/pause"                 5 hours ago         Up 5 hours                              k8s_POD_multi-container-pod_default_79243492-822f-49f4-9690-1f31a6a209b6_0
```

### Inside the pod network namespace, an interface is created, and an IP address is assigned.
```
$ sudo docker inspect --format '{{ .State.Pid }}' eabab459f3f5
24009
$ sudo docker inspect --format '{{ .State.Pid }}' 7070e27637e9
23447
```

```
$ sudo nsenter -t 24009 -n ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: tunl0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN group default qlen 1000
    link/ipip 0.0.0.0 brd 0.0.0.0
4: eth0@if1293: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1440 qdisc noqueue state UP group default
    link/ether aa:83:b5:48:dd:82 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.20.225.119/32 brd 10.20.225.119 scope global eth0
       valid_lft forever preferred_lft forever
```

#### Other way to fetch same information

```
$ kubectl exec -it multi-container-pod -- ip a
Defaulting container name to container-1.
Use 'kubectl describe pod/multi-container-pod -n default' to see all of the containers in this pod.
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: tunl0@NONE: <NOARP> mtu 1480 qdisc noop qlen 1000
    link/ipip 0.0.0.0 brd 0.0.0.0
4: eth0@if1293: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1440 qdisc noqueue
    link/ether aa:83:b5:48:dd:82 brd ff:ff:ff:ff:ff:ff
    inet 10.20.225.119/32 brd 10.20.225.119 scope global eth0
       valid_lft forever preferred_lft forever
```

#### Lookup the corresponding host interface on host network namespace

```
$ ip link show | grep -A1 ^1293
1293: cali97e50e215bd@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1440 qdisc noqueue state UP mode DEFAULT group default
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 3
```

#### Network Plumbing
Each newly created pod on the node will be set up with a veth pair like this.

The pause process again, and this time it's holding the network namespace hostage. This pause container is responsible for creating and holding the network namespace. Just before the pod is deployed and container created, (among other things) it's the runtime responsibility to create the network namespace. Instead of running ip netns and creating the network namespace manually, the container runtime does this automatically. It contains very little code and instantly goes to sleep as soon as deployed. If one of the containers inside the pod crashes, the remaining can still reply to any network requests.

```
[centos@support-lts-node1 ~]$ sudo lsns -p 24009 | grep net
4026532565 net        7 23197 root /pause

[centos@support-lts-node1 ~]$ sudo lsns -p 23447 | grep net
4026532565 net        7 23197 root /pause
```

---

#### For Pod multi-container-pod2 on Node 10.128.146.71
```
$ sudo docker ps | grep multi
f28eff85e429        nginx                                              "/docker-entrypoint.…"   14 minutes ago      Up 14 minutes                           k8s_container-2_multi-container-pod2_default_85dda6df-b334-4ae5-ae13-dd1652aa4abb_0
0f6bc972101e        busybox                                            "/bin/sh -c 'sleep 1…"   15 minutes ago      Up 15 minutes                           k8s_container-1_multi-container-pod2_default_85dda6df-b334-4ae5-ae13-dd1652aa4abb_0
2a21d1c6b438        localhost:5100/pause:3.2                           "/pause"                 15 minutes ago      Up 15 minutes                           k8s_POD_multi-container-pod2_default_85dda6df-b334-4ae5-ae13-dd1652aa4abb_0
```

```
$ sudo docker inspect --format '{{ .State.Pid }}' f28eff85e429
25926
$ sudo docker inspect --format '{{ .State.Pid }}' 0f6bc972101e
25278
```

```
$ sudo nsenter -t 25926 -n ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: tunl0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN group default qlen 1000
    link/ipip 0.0.0.0 brd 0.0.0.0
4: eth0@if2454: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1440 qdisc noqueue state UP group default
    link/ether de:a6:c0:9e:41:0f brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.20.127.150/32 brd 10.20.127.150 scope global eth0
       valid_lft forever preferred_lft forever
```

```
$ ip link show | grep -A1 ^2454
2454: calie92f8848552@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1440 qdisc noqueue state UP mode DEFAULT group default
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 4
```

```
$ sudo lsns -p 25926 | grep net
4026532576 net        7 25132 root /pause
$ sudo lsns -p 25278 | grep net
4026532576 net        7 25132 root /pause
```

--
### Pod to Pod Communication across Nodes

#### Traffic Flow via Calico
The key thing is to remember is how Calico programs the chain. All traffic to the pod go through the chain (cali-tw-<cali**>). All traffic from the pod go through the chain (cali-fw-<cali**>). The core principle of calico networking is IP routing, and each container or virtual machine is assigned a workload-endpoint (wl).

The message sent from ConA to ConB is received by wl-A of nodeA, and is forwarded to nodeB after passing through various iptables rules according to the routing rules on nodeA.

Nodes exchange routing information through the BGP protocol. Each node runs a soft routing software bird and is set as a BGP Speaker to exchange routing information with other nodes through the BGP protocol.

#### Ping to Pod multi-container-pod2 from Pod multi-container-pod

```
$ kubectl exec -it multi-container-pod -- /bin/sh
/ # ping 10.20.127.150
PING 10.20.127.150 (10.20.127.150): 56 data bytes
64 bytes from 10.20.127.150: seq=0 ttl=62 time=7.987 ms
64 bytes from 10.20.127.150: seq=1 ttl=62 time=0.763 ms
64 bytes from 10.20.127.150: seq=2 ttl=62 time=0.799 ms
```

#### On Source Node 10.128.146.244

```
# sudo tcpdump -i cali97e50e215bd
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on cali97e50e215bd, link-type EN10MB (Ethernet), capture size 262144 bytes
15:45:50.305931 IP 10.20.225.119 > 10.20.127.150: ICMP echo request, id 32, seq 0, length 64
15:45:50.307031 IP 10.20.127.150 > 10.20.225.119: ICMP echo reply, id 32, seq 0, length 64
15:45:51.306151 IP 10.20.225.119 > 10.20.127.150: ICMP echo request, id 32, seq 1, length 64
15:45:51.306926 IP 10.20.127.150 > 10.20.225.119: ICMP echo reply, id 32, seq 1, length 64
15:45:52.306338 IP 10.20.225.119 > 10.20.127.150: ICMP echo request, id 32, seq 2, length 64
15:45:52.306851 IP 10.20.127.150 > 10.20.225.119: ICMP echo reply, id 32, seq 2, length 64
```

```
# watch -d iptables -L cali-fw-cali97e50e215bd -vn
Every 2.0s: iptables -L cali-fw-cali97e50e215bd -vn                                                                                   Fri Jul 22 15:42:13 2022

Chain cali-fw-cali97e50e215bd (2 references)
 pkts bytes target     prot opt in     out     source               destination
  246 15159 ACCEPT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* cali:9CJD_eoTe_bHnqNA */ ctstate RELATED,ESTABLISHED
    0     0 DROP       all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* cali:fSCGgtxkJrtUM8a- */ ctstate INVALID
   13   932 MARK       all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* cali:_Ri5Wk79FLqE8E14 */ MARK and 0xfffeffff
    0     0 DROP       udp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* cali:ypgKNhoJHo5LX3tX */ /* Drop VXLAN encapped packets originatin
g in workloads */ multiport dports 4789
    0     0 DROP       4    --  *      *       0.0.0.0/0            0.0.0.0/0            /* cali:aQFkVxyhUszk5VOr */ /* Drop IPinIP encapped packets originati
ng in workloads */
   13   932 cali-pro-kns.default  all  --  *	  *	  0.0.0.0/0            0.0.0.0/0            /* cali:yCbCS-hMRNuNYS_z */
   13   932 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* cali:GHSkegaO6C47A1Jl */ /* Return if profile accepted */ mark mat
ch 0x10000/0x10000
    0     0 cali-pro-ksa.default.default  all  --  *	  *	  0.0.0.0/0            0.0.0.0/0            /* cali:eQwFE1NLrzrh1rVj */
    0     0 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* cali:3JOLHovKa7Ew6dh_ */ /* Return if profile accepted */ mark mat
ch 0x10000/0x10000
    0     0 DROP       all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* cali:CMmm2WadvIezQ30g */ /* Drop if no profiles matched */
```

#### On Destination Node 10.128.146.71

```
# sudo tcpdump -i calie92f8848552
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on calie92f8848552, link-type EN10MB (Ethernet), capture size 262144 bytes
15:45:50.304528 IP 10.20.225.119 > 10.20.127.150: ICMP echo request, id 32, seq 0, length 64
15:45:50.304716 IP 10.20.127.150 > 10.20.225.119: ICMP echo reply, id 32, seq 0, length 64
15:45:51.304523 IP 10.20.225.119 > 10.20.127.150: ICMP echo request, id 32, seq 1, length 64
15:45:51.304573 IP 10.20.127.150 > 10.20.225.119: ICMP echo reply, id 32, seq 1, length 64
15:45:52.304563 IP 10.20.225.119 > 10.20.127.150: ICMP echo request, id 32, seq 2, length 64
15:45:52.304625 IP 10.20.127.150 > 10.20.225.119: ICMP echo reply, id 32, seq 2, length 64
15:45:55.311341 ARP, Request who-has 10.20.127.150 tell support-lts-node3.novalocal, length 28
15:45:55.311327 ARP, Request who-has 169.254.1.1 tell 10.20.127.150, length 28
15:45:55.311402 ARP, Reply 169.254.1.1 is-at ee:ee:ee:ee:ee:ee (oui Unknown), length 28
15:45:55.311411 ARP, Reply 10.20.127.150 is-at de:a6:c0:9e:41:0f (oui Unknown), length 28
```

```
# watch -d iptables -L cali-tw-calie92f8848552 -vn
Every 2.0s: iptables -L cali-tw-calie92f8848552 -vn                                                                                   Fri Jul 22 15:42:28 2022

Chain cali-tw-calie92f8848552 (1 references)
 pkts bytes target     prot opt in     out     source               destination
   73  5672 ACCEPT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* cali:FKbQyzLisXug7jeZ */ ctstate RELATED,ESTABLISHED
    0     0 DROP       all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* cali:EpTH33zsUUPwYPG2 */ ctstate INVALID
   23  1524 MARK       all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* cali:sdcYL2H02dKolK9V */ MARK and 0xfffeffff
   23  1524 cali-pri-kns.default  all  --  *	  *	  0.0.0.0/0            0.0.0.0/0            /* cali:bgRDmnDjxV0iWI1b */
   23  1524 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* cali:_m0cmojoecm3wXxC */ /* Return if profile accepted */ mark mat
ch 0x10000/0x10000
    0     0 cali-pri-ksa.default.default  all  --  *	  *	  0.0.0.0/0            0.0.0.0/0            /* cali:-esqbtJMKHJhMpC3 */
    0     0 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* cali:EZhXoak8D4Lu4i7Y */ /* Return if profile accepted */ mark mat
ch 0x10000/0x10000
    0     0 DROP       all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* cali:G-_fOCLusA5V69C6 */ /* Drop if no profiles matched */
```
