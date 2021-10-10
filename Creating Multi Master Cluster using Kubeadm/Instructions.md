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

[root@sanchit-km1 ~]# cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
> br_netfilter
> EOF
br_netfilter

[root@sanchit-km1 ~]# cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
> net.bridge.bridge-nf-call-ip6tables = 1
> net.bridge.bridge-nf-call-iptables = 1
> EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1

[root@sanchit-km1 ~]# sudo sysctl --system
* Applying /usr/lib/sysctl.d/00-system.conf ...
net.bridge.bridge-nf-call-ip6tables = 0
net.bridge.bridge-nf-call-iptables = 0
net.bridge.bridge-nf-call-arptables = 0
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
* Applying /etc/sysctl.d/k8s.conf ...
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
* Applying /etc/sysctl.conf ...
```

#### Install and configure Docker
```
[root@sanchit-km1 ~]# curl -fsSL https://get.docker.com -o get-docker.sh
```

```
[root@sanchit-km1 ~]# sh get-docker.sh
# Executing docker install script, commit: 93d2499759296ac1f9c510605fef85052a2c32be
+ sh -c 'yum install -y -q yum-utils'
Package yum-utils-1.1.31-54.el7_8.noarch already installed and latest version
+ sh -c 'yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo'
Loaded plugins: fastestmirror
adding repo from: https://download.docker.com/linux/centos/docker-ce.repo
grabbing file https://download.docker.com/linux/centos/docker-ce.repo to /etc/yum.repos.d/docker-ce.repo
repo saved to /etc/yum.repos.d/docker-ce.repo
+ '[' stable '!=' stable ']'
+ sh -c 'yum makecache'
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
...
+ sh -c 'yum install -y -q docker-ce'
warning: /var/cache/yum/x86_64/7/docker-ce-stable/packages/docker-ce-20.10.9-3.el7.x86_64.rpm: Header V4 RSA/SHA512 Signature, key ID 621e9f35: NOKEY
Public key for docker-ce-20.10.9-3.el7.x86_64.rpm is not installed
Importing GPG key 0x621E9F35:
 Userid     : "Docker Release (CE rpm) <docker@docker.com>"
 Fingerprint: 060a 61c5 1b55 8a7f 742b 77aa c52f eb6b 621e 9f35
 From       : https://download.docker.com/linux/centos/gpg
+ version_gte 20.10
+ '[' -z '' ']'
+ return 0
+ sh -c 'yum install -y -q docker-ce-rootless-extras'
Package docker-ce-rootless-extras-20.10.9-3.el7.x86_64 already installed and latest version

================================================================================

To run Docker as a non-privileged user, consider setting up the
Docker daemon in rootless mode for your user:

    dockerd-rootless-setuptool.sh install

Visit https://docs.docker.com/go/rootless/ to learn about rootless mode.


To run the Docker daemon as a fully privileged service, but granting non-root
users access, refer to https://docs.docker.com/go/daemon-access/

WARNING: Access to the remote API on a privileged Docker daemon is equivalent
         to root access on the host. Refer to the 'Docker daemon attack surface'
         documentation for details: https://docs.docker.com/go/attack-surface/

================================================================================
```

```
[root@sanchit-km1 ~]# cat /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
```

```
[root@sanchit-km1 ~]# systemctl daemon-reload
[root@sanchit-km1 ~]# systemctl enable docker
Created symlink from /etc/systemd/system/multi-user.target.wants/docker.service to /usr/lib/systemd/system/docker.service.
```

```
[root@sanchit-km1 ~]# systemctl status docker
● docker.service - Docker Application Container Engine
   Loaded: loaded (/usr/lib/systemd/system/docker.service; enabled; vendor preset: disabled)
   Active: active (running) since Sat 2021-10-09 22:04:36 UTC; 7s ago
     Docs: https://docs.docker.com
 Main PID: 15404 (dockerd)
    Tasks: 9
   Memory: 32.5M
   CGroup: /system.slice/docker.service
           └─15404 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
```

#### Install kubeadm, kubectl and kubelet

1. Add the Google repository key.

```
[root@sanchit-km1 ~]# cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
> [kubernetes]
> name=Kubernetes
> baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
> enabled=1
> gpgcheck=1
> repo_gpgcheck=1
> gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
> exclude=kubelet kubeadm kubectl
> EOF
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
```

```
[root@sanchit-km1 ~]# sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
```

```
[root@sanchit-km1 ~]# sudo systemctl enable --now kubelet
Created symlink from /etc/systemd/system/multi-user.target.wants/kubelet.service to /usr/lib/systemd/system/kubelet.service.
```

```
[root@sanchit-km1 ~]# kubectl version
Client Version: version.Info{Major:"1", Minor:"22", GitVersion:"v1.22.2", GitCommit:"8b5a19147530eaac9476b0ab82980b4088bbc1b2", GitTreeState:"clean", BuildDate:"2021-09-15T21:38:50Z", GoVersion:"go1.16.8", Compiler:"gc", Platform:"linux/amd64"}
```

#### Installing the client tools

We will need two tools on the client machine: the Cloud Flare SSL tool to generate the different certificates

#### Installing cfssl
1. Download the binaries.
```
[root@sanchit-km1 ~]# wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
[root@sanchit-km1 ~]# wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
```

2. Add the execution permission to the binaries.
```
[root@sanchit-km1 ~]# chmod +x cfssl*
```

3. Move the binaries to /usr/local/bin.
```
[root@sanchit-km1 ~]# sudo mv cfssl_linux-amd64 /usr/local/bin/cfssl
[root@sanchit-km1 ~]# sudo mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
```

4. Verify the installation
```
[root@sanchit-km1 ~]# /usr/local/bin/cfssl version
Version: 1.2.0
Revision: dev
Runtime: go1.6
```

#### Generating the TLS certificates

1. Create the certificate authority configuration file.
```
[root@sanchit-km1 ~]# cat ca-config.json
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "kubernetes": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "8760h"
      }
    }
  }
}
```

2. Create the certificate authority signing request configuration file.
```
[root@sanchit-km1 ~]# cat ca-csr.json
{
  "CN": "Kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
  {
    "C": "IE",
    "L": "Cork",
    "O": "Kubernetes",
    "OU": "CA",
    "ST": "Cork Co."
  }
 ]
}
```

3. Generate the certificate authority certificate and private key.
```
[root@sanchit-km1 ~]# /usr/local/bin/cfssl gencert -initca ca-csr.json | /usr/local/bin/cfssljson -bare ca
2021/10/09 23:24:57 [INFO] generating a new CA key and certificate from CSR
2021/10/09 23:24:57 [INFO] generate received request
2021/10/09 23:24:57 [INFO] received CSR
2021/10/09 23:24:57 [INFO] generating key: rsa-2048
2021/10/09 23:24:57 [INFO] encoded CSR
2021/10/09 23:24:57 [INFO] signed certificate with serial number 563795634195758696450648701351519072206527597554
```

4.  Verify that the ca-key.pem and the ca.pem were generated.
```
[root@sanchit-km1 ~]# ls -al ca.pem ca-key.pem
-rw-------. 1 root root 1675 Oct  9 23:24 ca-key.pem
-rw-r--r--. 1 root root 1363 Oct  9 23:24 ca.pem
```

#### Creating the certificate for the Etcd cluster

1. Create the certificate signing request configuration file.
```
[root@sanchit-km1 ~]# cat kubernetes-csr.json
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
  {
    "C": "IE",
    "L": "Cork",
    "O": "Kubernetes",
    "OU": "Kubernetes",
    "ST": "Cork Co."
  }
 ]
}
```

2. Generate the certificate and private key.
```
[root@sanchit-km1 ~]# /usr/local/bin/cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -hostname=10.128.147.104,10.128.146.58,10.128.147.81,127.0.0.1,kubernetes.default -profile=kubernetes kubernetes-csr.json | /usr/local/bin/cfssljson -bare kubernetes
2021/10/09 23:34:12 [INFO] generate received request
2021/10/09 23:34:12 [INFO] received CSR
2021/10/09 23:34:12 [INFO] generating key: rsa-2048
2021/10/09 23:34:12 [INFO] encoded CSR
2021/10/09 23:34:12 [INFO] signed certificate with serial number 94575227083486979445698735726425610171271671902
```

3. Verify that the kubernetes-key.pem and the kubernetes.pem file were generated.
```
[root@sanchit-km1 ~]# ls -al kubernetes.csr kubernetes-key.pem
-rw-r--r--. 1 root root 1013 Oct  9 23:34 kubernetes.csr
-rw-------. 1 root root 1675 Oct  9 23:34 kubernetes-key.pem
```

4. Copy the certificate to each nodes.
```
[root@sanchit-km1 ~]# scp ca.pem kubernetes.pem kubernetes-key.pem centos@10.128.146.58:~
[root@sanchit-km1 ~]# scp ca.pem kubernetes.pem kubernetes-key.pem centos@10.128.147.81:~
```

#### Installing and configuring Etcd

1. Create a configuration directory for Etcd.
```
[root@sanchit-km1 ~]# sudo mkdir /etc/etcd /var/lib/etcd
```

2. Move the certificates to the configuration directory.
```
[root@sanchit-km1 ~]# sudo mv ~/ca.pem ~/kubernetes.pem ~/kubernetes-key.pem /etc/etcd
```

3. Download the etcd binaries.
```
[root@sanchit-km1 ~]# wget https://github.com/etcd-io/etcd/releases/download/v3.4.17/etcd-v3.4.17-linux-amd64.tar.gz
...
Saving to: ‘etcd-v3.4.17-linux-amd64.tar.gz’

100%[=====================================================================================================>] 17,414,995  31.8MB/s   in 0.5s

2021-10-09 23:47:12 (31.8 MB/s) - ‘etcd-v3.4.17-linux-amd64.tar.gz’ saved [17414995/17414995]
```

4. Extract the etcd archive.
```
[root@sanchit-km1 ~]# tar -xvf etcd-v3.4.17-linux-amd64.tar.gz
```

5. Move the etcd binaries to /usr/local/bin.
```
[root@sanchit-km1 ~]# mv etcd-v3.4.17-linux-amd64/etcd* /usr/local/bin/
```

6. Create an etcd systemd unit file.
```
[root@sanchit-km1 ~]# cat /etc/systemd/system/etcd.service
[Unit]
Description=etcd
Documentation=https://github.com/coreos


[Service]
ExecStart=/usr/local/bin/etcd \
  --name 10.128.147.104 \
  --cert-file=/etc/etcd/kubernetes.pem \
  --key-file=/etc/etcd/kubernetes-key.pem \
  --peer-cert-file=/etc/etcd/kubernetes.pem \
  --peer-key-file=/etc/etcd/kubernetes-key.pem \
  --trusted-ca-file=/etc/etcd/ca.pem \
  --peer-trusted-ca-file=/etc/etcd/ca.pem \
  --peer-client-cert-auth \
  --client-cert-auth \
  --initial-advertise-peer-urls https://10.128.147.104:2380 \
  --listen-peer-urls https://10.128.147.104:2380 \
  --listen-client-urls https://10.128.147.104:2379,http://127.0.0.1:2379 \
  --advertise-client-urls https://10.128.147.104:2379 \
  --initial-cluster-token etcd-cluster-0 \
  --initial-cluster 10.128.147.104=https://10.128.147.104:2380,10.128.146.58=https://10.128.146.58:2380,10.128.147.81=https://10.128.147.81:2380 \
  --initial-cluster-state new \
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5



[Install]
WantedBy=multi-user.target
```

7. Reload the daemon configuration.
```
[root@sanchit-km1 ~]# systemctl daemon-reload
```

8. Enable etcd to start at boot time.
```
[root@sanchit-km1 ~]# systemctl enable etcd
Created symlink from /etc/systemd/system/multi-user.target.wants/etcd.service to /etc/systemd/system/etcd.service.
```

9. Start etcd.
```
[root@sanchit-km1 ~]# systemctl start etcd

[root@sanchit-km1 ~]# systemctl status etcd
● etcd.service - etcd
   Loaded: loaded (/etc/systemd/system/etcd.service; enabled; vendor preset: disabled)
   Active: active (running) since Sun 2021-10-10 00:14:04 UTC; 8min ago
     Docs: https://github.com/coreos
 Main PID: 28676 (etcd)
    Tasks: 15
   Memory: 45.6M
   CGroup: /system.slice/etcd.service
           └─28676 /usr/local/bin/etcd --name 10.128.147.104 --cert-file=/etc/etcd/kubernetes.pem --key-file=/etc/etcd/kubernetes-key.pem --peer-cert-file=/etc/etcd/kubernetes.pem --peer-key-file=/etc/etcd/kubernetes-key.pem --trusted-...
```

10. Verify that the cluster is up and running.
```
[root@sanchit-km1 ~]# ETCDCTL_API=3 /usr/local/bin/etcdctl member list
1e332659602b35d8, started, 10.128.147.104, https://10.128.147.104:2380, https://10.128.147.104:2379, false
58e831dc56f2d417, started, 10.128.146.58, https://10.128.146.58:2380, https://10.128.146.58:2379, false
f61247f59d1a3ef1, started, 10.128.147.81, https://10.128.147.81:2380, https://10.128.147.81:2379, false
```

11. Install Keepalived on All Master Nodes

For providing load balancing from a virtual IP, The keepalived service provides a virtual IP managed by a configurable health check. Due to the way the virtual IP is implemented, all the hosts between which the virtual IP is negotiated need to be in the same IP subnet.
The keepalived configuration consists of two files: the service configuration file and a health check script which will be called periodically to verify that the node holding the virtual IP is still operational.


The keepalived configuration file is located at /etc/keepalived/keepalived.conf. In this file , you need to replace KEEPALIVED_AUTH_PASS with your own password and make the password identical on all Master nodes, and change the interface item value to the server’s proper network interface name.

1. Install keepalived
```
[root@sanchit-km1 ~]# yum install keepalived -y
```

2. Create keepalived configu file /etc/keepalived/keepalived.conf with the following content
```
[root@sanchit-km1 ~]# cat /etc/keepalived/keepalived.conf
global_defs {
   router_id LVS_k8s
}
vrrp_script CheckK8sMaster {
    script "curl -k https://10.128.147.104:6443"
    interval 3
    timeout 9
    fall 2
    rise 2
}
vrrp_instance VI_1 {
    state BACKUP
    interface eth0
    virtual_router_id 61
    priority 80
    advert_int 1
    mcast_src_ip 10.128.147.104
    nopreempt
    authentication {
          auth_type PASS
        auth_pass KEEPALIVED_AUTH_PASS
    }
    unicast_peer {
        10.128.146.58
        10.128.147.81
    }
    virtual_ipaddress {
        10.128.146.54/24
    }
    track_script {
        CheckK8sMaster
    }
}
```
```
[root@sanchit-km1 ~]# systemctl daemon-reload && systemctl enable keepalived && systemctl restart keepalived
Created symlink from /etc/systemd/system/multi-user.target.wants/keepalived.service to /usr/lib/systemd/system/keepalived.service.
```

VIP on 10.128.146.54 on sanchit-km1

```
[root@sanchit-km1 ~]# ip a
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether fa:16:3e:27:bf:ae brd ff:ff:ff:ff:ff:ff
    inet 10.128.147.104/23 brd 10.128.147.255 scope global dynamic eth0
       valid_lft 56626sec preferred_lft 56626sec
    inet 10.128.146.54/24 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::f816:3eff:fe27:bfae/64 scope link
       valid_lft forever preferred_lft forever
```

#### Initializing the master nodes

Initializing the Master node 10.128.147.104

1. Create the configuration file for kubeadm.
```
[root@sanchit-km1 ~]# cat kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1alpha1
kind: MasterConfiguration
api:
  advertiseAddress: 10.128.147.104
etcd:
  endpoints:
  - https://10.128.147.104:2379
  - https://10.128.146.58:2379
  - https://10.128.147.81:2379
  caFile: /etc/etcd/ca.pem
  certFile: /etc/etcd/kubernetes.pem
  keyFile: /etc/etcd/kubernetes-key.pem
networking:
  podSubnet: 10.244.0.0/16
apiServerCertSANs:
- 10.128.147.104
- 10.128.146.58
- 10.128.147.81
- 10.128.146.54
apiServerExtraArgs:
  endpoint-reconciler-type: lease
```

```
[root@sanchit-km1 ~]# systemctl stop etcd

[root@sanchit-km1 ~]# rm -rf /var/lib/etcd
```

```
[root@sanchit-km1 ~]# kubeadm init --config kubeadm-config.yaml
W1010 03:25:07.929694   10989 strict.go:55] error unmarshaling configuration schema.GroupVersionKind{Group:"kubeadm.k8s.io", Version:"v1beta3", Kind:"ClusterConfiguration"}: error unmarshaling JSON: while decoding JSON: json: unknown field "api"
[init] Using Kubernetes version: v1.22.2
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
...
...
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 10.128.147.104:6443 --token mwv95h.igkd634dlylc35v1 \
	--discovery-token-ca-cert-hash sha256:43758c51e1ffd5914f4e3cb4e804ebbce707d44d88100ff2f4bb65cd1c67303f
```

```
[root@sanchit-km1 ~]# mkdir -p $HOME/.kube
[root@sanchit-km1 ~]# sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
[root@sanchit-km1 ~]# sudo chown $(id -u):$(id -g) $HOME/.kube/config
[root@sanchit-km1 ~]# export KUBECONFIG=/etc/kubernetes/admin.conf
```

```
[root@sanchit-km1 ~]# scp -rp /etc/kubernetes/pki centos@10.128.146.58:~

[root@sanchit-km1 ~]# scp -rp /etc/kubernetes/pki centos@10.128.147.81:~
```
