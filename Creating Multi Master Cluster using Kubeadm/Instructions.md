## Creating Highly Available Multi Master Kubernetes Cluster using Kubeadm

When setting up a production cluster, high availability (the cluster's ability to remain operational even if some control plane or worker nodes fail) is a must. Redundancy of control plane nodes incl. etcd needs to be considered for when planning and setting up a cluster.

### Server Information

Master Nodes: 
```
sanchit-km1 [10.128.147.104]
sanchit-km2 [10.128.146.58]
sanchit-km3 [10.128.147.81]
```
Worker Node:
```
sanchit-kw [10.128.147.213]
```
### Preparing the servers
#### Disable SELinux 
```
[root@sanchit-km1 ~]# setenforce 0
[root@sanchit-km1 ~]# sed -i 's/SELINUX=permissive/SELINUX=disabled/' /etc/sysconfig/selinux
```
#### Disable swap
```
[root@sanchit-km1 ~]# swapoff -a
[root@sanchit-km1 ~]# sed -i 's/^.*swap/#&/' /etc/fstab
```
#### Enable Traffic Forwarding
```
[root@sanchit-km1 ~]# iptables -P FORWARD ACCEPT

[root@sanchit-km1 ~]# sysctl -w net.ipv4.ip_forward=1
net.ipv4.ip_forward = 1

[root@sanchit-km1 ~]# sysctl -w vm.swappiness=0
vm.swappiness = 0

[root@sanchit-km1 ~]# sysctl --system
* Applying /usr/lib/sysctl.d/00-system.conf ...
* Applying /usr/lib/sysctl.d/10-default-yama-scope.conf ...
kernel.yama.ptrace_scope = 0
* Applying /usr/lib/sysctl.d/50-default.conf ...
kernel.sysrq = 16
kernel.core_uses_pid = 1
kernel.kptr_restrict = 1
net.ipv4.conf.default.rp_filter = 1
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.accept_source_route = 0
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.promote_secondaries = 1
net.ipv4.conf.all.promote_secondaries = 1
fs.protected_hardlinks = 1
fs.protected_symlinks = 1
* Applying /etc/sysctl.d/99-sysctl.conf ...
* Applying /etc/sysctl.conf ...
```


For providing load balancing from a virtual IP, The keepalived service provides a virtual IP managed by a configurable health check. Due to the way the virtual IP is implemented, all the hosts between which the virtual IP is negotiated need to be in the same IP subnet.
The keepalived configuration consists of two files: the service configuration file and a health check script which will be called periodically to verify that the node holding the virtual IP is still operational.