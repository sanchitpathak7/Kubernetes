#### Container Runtime: 
Software that's responsible for pulling and running container images.

Kubernetes 1.5 introduced an internal plugin API named Container Runtime Interface (CRI) to provide easy access to different container runtimes.

Different popular runtimes are:
1. Docker
2. Containerd
3. CRI-O


#### What is Containerd?
It is a high-level container runtime. It is a daemon that manages the complete container lifecycle on a single host: creates, starts, stops containers, pulls and stores images, configures mounts, networking, etc.

Containerd content store: /var/lib/containerd/io.containerd.content.v1.content

### ContainerD with Docker
Docker => DockerD => Containerd => OCI Spec/Runc => Containers

When you run a container with docker, you're actually running it through the Docker daemon, containerd, and then runc.

### Docker as runtime with Kubernetes
Kubectl => Kubelet => (CRI) => dockershim [over Unix Socket and using grpc framework i.e. 2 services image and runtime] => docker => Containerd => Containers

Kubernetes includes a component called dockershim that allows it to support Docker.

Kubernetes prefers to run containers through any container runtime which supports its Container Runtime Interface (CRI).

### Why Dockershim is needed?
Provides a layer of abstaraction to the Docker Engine.

Docker doesn't support CRI API. Thus, dockershim was a temporary solution to use Docker as Container Runtime.

Kubelet Endpoints:
    -image-service-endpoint
    -container-runtime-endpoint
    (Default: unix:///var/run/dockershim.sock)


### Problem with Docker as Runtime
Kubernetes is deprecating Docker as container runtime after v1.20.
Docker containers are still supported, but the dockershim/docker, the layer between Kubernetes and containerd is deprecated and will be removed from version 1.22+.

### Do we keep using Dockerfiles?
Docker is still useful in many ways. The image that Docker produces isn't really a Docker specific image - it's an OCI (Open Container Initiative) image.
Any OCI complaint image, regardless of the tool one uses to built it, will look the same to Kubernetes.

### Change from Docker Shim to Containerd CRI

No extra layer of abstraction

Kubelet communicates with containerd over unix socket.

change kubelet parameters
change containerd configuration

1. Enable CRI plugin 


/etc/containerd/config.toml

disabled_plugins = ["cri"]

# cat /etc/containerd/config.toml | grep disabled
disabled_plugins = ["cri"]

2.
kubelet paramters

--container-runtime=remote
--container-runtime-endpoint=unix:///run/containerd/containerd.sock

```
$ less kubelet.INFO | grep -e container-runtime-endpoint -e container-runtime
I1011 23:33:17.743347 3362862 flags.go:59] FLAG: --container-runtime="docker"
I1011 23:33:17.743350 3362862 flags.go:59] FLAG: --container-runtime-endpoint="unix:///var/run/dockershim.sock"
```

### CONTAINERD CLIENTs

ctr (Shipped as part of containerd package)
not compatible with docker CLI, not friendly to users

nerdctl
Docker comptabile CLI for containerd. Output scheme is also similar to docker.

nerdctl is a Docker-compatible CLI for containerd. Its output scheme is also similar to docker. nerdctl supports most of the docker cli such as nerdctl run, nerdctl exec, nerdctl inspect and so on. All the support cli’s: command-reference `

Note: There are a few docker-cli flags like —filter, —format etc. we use that are not yet present in nerdctl. 

More information on nerdctl: containerd/nerdctl 



supports most of the docker CLI.

All k8s containers are in k8s.io containerd namespace.

Location: /opt/pf9/pf9-kube/bin/nerdctl



Both clients run via unix sockets



 crictl only works with CRI compatible container runtimes.
 helpful to inspect and debug container runtimes and applications on nodes.

 incomptabile with Docker CLI, not friendly to users


kubectl get nodes -o wide







devicemapper v/s overlay2

cgroup driver: cgroupfs vs systemd




/opt/pf9/pf9-kube/bin/nerdctl -n k8s.io ps

/opt/pf9/pf9-kube/bin/crictl -r unix:///run/containerd/containerd.sock exec -it <ID> /bin/sh

- - + 
DOCKER

```
$ cat /etc/docker/daemon.json
{
    "bridge": "none",
    "graph": "/var/lib/docker",
    "group": "pf9group",
    "live-restore": true,
    "log-driver": "json-file",
    "log-opts": {
        "max-size": "10m",
        "max-file": "10"
    },
    "storage-driver": "",
    "storage-opts": [ ],
    "debug": false,
    "registry-mirrors": ["https://dockermirror.platform9.io/"]
}
```



```
curl --request POST --url https://cse.platform9.io/qbert/v4/b855dca4be144f089b68dcff7c39f820/clusters/e3f58795-5805-49cf-81fb-768fd6d686d6/upgrade\?type\=minor --header "X-Auth-Token: $TOKEN" --header 'Content-Type: application/json' --header 'cache-control: no-cache' -d '{ "containerRuntime": "containerd" }'
OK%
```

```
[2021-10-27 04:33:37] Generating config.toml
--- /opt/pf9/pf9-kube/master_scripts/045-start_container_runtime.sh start at 2021-10-27 04:33:39 ---
[2021-10-27 04:33:40] docker configuration: http proxy configuration not detected
[2021-10-27 04:33:40] DockerHub credentials not available. You may hit the DockerHub image pull rate limit.
--- /opt/pf9/pf9-kube/master_scripts/050-etcd_configure.sh start at 2021-10-27 04:33:40 ---
[2021-10-27 04:33:41] Ensuring etcd data is stored on host
[2021-10-27 04:33:41] time="2021-10-27T04:33:41Z" level=fatal msg="no such object etcd"
[2021-10-27 04:33:41] Skipping; etcd container does not exist
--- /opt/pf9/pf9-kube/master_scripts/055-etcd_run.sh start at 2021-10-27 04:33:42 ---
[2021-10-27 04:33:42] Node endpoint is ddu-master2
[2021-10-27 04:33:42] Deriving local etcd environment
[2021-10-27 04:33:42] Ensuring container 'etcd' is destroyed
[2021-10-27 04:33:42] gcr.io/etcd-development/etcd:v3.4.14: resolving      |ESC[32mESC[0m--------------------------------------|
[2021-10-27 04:33:42] elapsed: 0.1 s                        total:   0.0 B (0.0 B/s)
[2021-10-27 04:33:42] gcr.io/etcd-development/etcd:v3.4.14: resolving      |ESC[32mESC[0m--------------------------------------|
[2021-10-27 04:33:42] elapsed: 0.2 s                        total:   0.0 B (0.0 B/s)
[

...
...
[2021-10-27 04:33:47] 0ba9932dd9967f6a5d4b6dcf29509d90eb4ed933fccbcfd9404b88b46a5f92d6
[2021-10-27 04:33:58] https://localhost:4001 is healthy: successfully committed proposal: took = 20.676423ms
--- /opt/pf9/pf9-kube/master_scripts/060-network_configure.sh start at 2021-10-27 04:33:58 ---
--- /opt/pf9/pf9-kube/master_scripts/065-cni_configure.sh start at 2021-10-27 04:33:59 ---
--- /opt/pf9/pf9-kube/master_scripts/070-auth_webhook.sh start at 2021-10-27 04:34:00 ---
[2021-10-27 04:34:01] unpacking docker.io/platform9/bouncer:1.2.0 (sha256:7f4fa6798a15bdd6ca77f3f4c75660bd3988c961fcf5e9508ff1d8bfa10d7891)...d
one
[2021-10-27 04:34:01] unpacking overlayfs@sha256:7f4fa6798a15bdd6ca77f3f4c75660bd3988c961fcf5e9508ff1d8bfa10d7891 (sha256:7f4fa6798a15bdd6ca77f
3f4c75660bd3988c961fcf5e9508ff1d8bfa10d7891)...done
[2021-10-27 04:34:01] Ensuring container 'bouncer' is destroyed
[2021-10-27 04:34:01] 3d508e2c7c3db15b0cb3eab14b77d23a05ccd411b45c37be53d999a0d61eeb10
--- /opt/pf9/pf9-kube/master_scripts/075-misc_scripts.sh start at 2021-10-27 04:34:02 ---
[2021-10-27 04:34:02] utils::write_cloud_provider_config: Env var KUBELET_CLOUD_CONFIG is empty, not writing a file
--- /opt/pf9/pf9-kube/master_scripts/080-kubelet_configure_start.sh start at 2021-10-27 04:34:02 ---
[2021-10-27 04:34:03] Node endpoint is ddu-master2
[2021-10-27 04:34:03] inactive
[2021-10-27 04:34:03] Generating runtime systemd unit for kubelet
[2021-10-27 04:34:03] kubelet configuration: http proxy configuration not detected
[2021-10-27 04:34:03] Starting kubelet
--- /opt/pf9/pf9-kube/master_scripts/090-kube_proxy_start.sh start at 2021-10-27 04:34:05 ---
```

```
$ cat /etc/containerd/config.toml | grep disabled
disabled_plugins = []
```

```
$ less kubelet.INFO | grep -e container-runtime-endpoint -e container-runtime
I1027 04:35:20.616072 1541827 flags.go:59] FLAG: --container-runtime="remote"
I1027 04:35:20.616089 1541827 flags.go:59] FLAG: --container-runtime-endpoint="unix:///run/containerd/containerd.sock"
```

```
$ sudo /opt/pf9/pf9-kube/bin/nerdctl -n k8s.io ps
CONTAINER ID    IMAGE                                                                      COMMAND                   CREATED           STATUS    PORTS    NAMES
00e4238ede6c    k8s.gcr.io/kube-scheduler:v1.21.3                                          "kube-scheduler --co…"    42 minutes ago    Up
0ba9932dd996    gcr.io/etcd-development/etcd:v3.4.14                                       "/usr/local/bin/etcd"     42 minutes ago    Up                 etcd
```

```
$ sudo /opt/pf9/pf9-kube/bin/crictl -r unix:///run/containerd/containerd.sock ps
CONTAINER           IMAGE               CREATED             STATE               NAME                      ATTEMPT             POD ID
20532b652f259       50b52cdadbcf0       33 minutes ago      Running             calico-node               2                   c9d976f3f619f
37801cb77625e
```

```
$ sudo ctr -n k8s.io containers list
CONTAINER                                                           IMAGE                                                                                                          RUNTIME
00e4238ede6cfb204962644322986966ce69cb0997990d49e3942d0816e08dbe    k8s.gcr.io/kube-scheduler:v1.21.3                                                                              io.containerd.runc.v2
0ba9932dd9967f6a5d4b6dcf29509d90eb4ed933fccbcfd9404b88b46a5f92d6    gcr.io/etcd-development/etcd:v3.4.14                                                                           io.containerd.runc.v2
```

### Image Management
When we download an image from the OCI registry, the image and all its content are loaded into the content store. Containerd has content store, under containerd's local storage space, for example, on a standard Linux installation at /var/lib/containerd/io.containerd.content.v1.content

```
# tree /var/lib/containerd/io.containerd.content.v1.content/blobs/sha256/
/var/lib/containerd/io.containerd.content.v1.content/blobs/sha256/
├── 020336b75c4893f1849758800d6f98bb2718faf3e5c812f91ce9fc4dfb69543b
├── 0a02d75339eaca89fcca3a8f39b69afba2cff13964c6d3a6a470e508ab4b43e4
├── 0c96386ac8cc1801fe7dfb7e4d41df94436490b9a11fae9b44db55e45a99df27
├── 0f938ef24c3b0163d5036348401191c8a33108c0e685541534b867fc6a6de78b
```

More information on Content Flow:  https://github.com/containerd/containerd/blob/main/docs/content-flow.md 

### Namespaces

Containerd supports namespaces at the container runtime level. These namespaces are completely different from the kubernetes namespaces. Containerd namespaces are used to provides isolation to different applications that might be using containerd like docker, kubelet, etc. Well-known namespaces -

k8s.io : contains all containers started from cri plugin by kubelet irrespective of their namespace in kubernetes

moby : contains all containers started by docker

Since containerd allows different apps to use different namespaces, we must provide k8s.io as namespace when interacting with containerd directly to manage containers.