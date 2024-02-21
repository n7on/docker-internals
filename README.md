# Docker Internals
Docker is a way to isolate a process from the rest of system using kernel features such as [namespaces](#namespaces), [cgroups](#cgroups), [capabilities](#capabilities) and [pivot_root](#pivot_root). When these features are used in conjunction to create an isolated environment, it's called a container. 

When a container is spawn using [Docker Engine](#docker-engine), `docker client` connects to `docker daemon` wich pulls a [Docker Image](#docker-image) and connects container to network using [Docker Networking](#docker-networking). The image is added to [Docker Filesystem](#docker-filesystem). The `image configuration` in the image is used by the [Docker Runtime](#docker-runtime) to create the process and the filesystem, and isolates it from the host.  


## namespaces
Namespaces are used to isolate processes. So that users, hostname, network, pid's etc only are visible from it's namespaces. This is the main concept of containers. Namespaces have 8 different types: 
* `net`. Network interfaces namespace.
* `mnt`. Mount namespace.
* `uts`. Hostname namespace.
* `pid`. Process namespace.
* `user`. User namespace (doesn't require privileged account).
* `time`. System time namespace.
* `ipc`. Inter-Process Communication namespace. Like shared memory, message queues and semaphores.
* `cgroup`. Cgroup namespace.  

Each namespace is held by at least 1 process. And a process can only belong to one namespace for each type at a given time. And all processes per default actually belong to one of each types in the `default namespaces`, which are held by the systems `PID 1`. So in that sense, `the host is also a container`. This can be visualized using `lsns`:

```bash
# list the init process namespaces
sudo lsns -p 1

> 4026531834 time       70   1 root /init
> 4026531835 cgroup     70   1 root /init
> 4026531837 user       70   1 root /init
> 4026531840 net        70   1 root /init
> 4026532266 ipc        70   1 root /init
> 4026532277 mnt        67   1 root /init
> 4026532278 uts        68   1 root /init
> 4026532279 pid        70   1 root /init

```
All other processes will inherit namespaces from it's parent process. So if we do same thing for the current shell process, we get exactly same namespace id's as with `init`: 

```bash
lsns -p $$ | awk '{print $1,$2}'

> 4026531834 time
> 4026531835 cgroup
> 4026531837 user
> 4026531840 net
> 4026532266 ipc
> 4026532277 mnt
> 4026532278 uts
> 4026532279 pid
```

We could start a new shell with new `uts` namespace using `unshare` command, which is a command that is a wrapper of the `syscall` with same name. And it's used in order to un-share a process from `default namespaces`. So we could start bash using `unshare` (to un-share from `uts` default namespace) and list it's new `uts` namespace id:  

> Note that `unshare` need root to create all types of namespaces except `user`. And this is also why Docker need root.  

```bash
sudo unshare --uts bash
# now running bash in new uts namespace

lsns -p $$

# and these namespaces is same as for init
> 4026531834 time       71     1 root /sbin/init
> 4026531835 cgroup     71     1 root /sbin/init
> 4026531837 user       71     1 root /sbin/init
> 4026531840 net        71     1 root /sbin/init
> 4026532266 ipc        71     1 root /sbin/init
> 4026532277 mnt        68     1 root /sbin/init
> 4026532279 pid        71     1 root /sbin/init

# but uts namespace have a new id
> 4026536218 uts         2 74238 root bash
```

Another way a process namespaces could be viewed is by exploring the `/proc` filesystem. Which are a `pseudo filesystem` provided by the kernel. For each process we have `/proc/<pid>/ns/<namespace>`, so we could list the namespaces of `init` (PID 1) using filesystem as well.

```bash
sudo ls -l /proc/1/ns

> lrwxrwxrwx 1 root root 0 Feb  4 16:26 cgroup -> 'cgroup:[4026531835]'
> lrwxrwxrwx 1 root root 0 Feb  4 16:26 ipc -> 'ipc:[4026532266]'
> lrwxrwxrwx 1 root root 0 Feb  4 16:26 mnt -> 'mnt:[4026532277]'
> lrwxrwxrwx 1 root root 0 Feb  4 16:26 net -> 'net:[4026531840]'
> lrwxrwxrwx 1 root root 0 Feb  4 16:26 pid -> 'pid:[4026532279]'
> lrwxrwxrwx 1 root root 0 Feb  7 18:56 pid_for_children -> 'pid:[4026532279]'
> lrwxrwxrwx 1 root root 0 Feb  4 16:26 time -> 'time:[4026531834]'
> lrwxrwxrwx 1 root root 0 Feb  7 18:56 time_for_children -> 'time:[4026531834]'
> lrwxrwxrwx 1 root root 0 Feb  4 16:26 user -> 'user:[4026531837]'
> lrwxrwxrwx 1 root root 0 Feb  4 16:26 uts -> 'uts:[4026532278]'

# or for specific namespace, like "mnt"
sudo readlink /proc/1/ns/mnt

```


## cgroups
Control Group (also called resource controllers), is a way to manage resources like memory, disk, CPU, network etc. So that resource limits can be added to a container, and usage can be extracted. `Cgroup` is structured in multiple separate hierarchies under `/sys/fs/cgroup`. Which contains each of it's subsystems. And a `cgroup` is isolated from host using it's `cgroup` namespace together with `cgroups` mounted from the host. When a container is started, [Docker Engine](#docker-engine) will create a new child group named `docker/<container id>` on the host under each subsystem. The host `cgroup` namespace will be copied, and if a limit is added it will be changed in the namespace. Following are some of the cgroup subsystem:

> Note that the `/sys` filesystem is just like `/dev` a pseudo filesystem provided by the kernel.

* `blkio`. Limits i/o on block devices.
* `cpu`. Limits CPU usage.
* `cpuacct`. Reports usage of CPU
* `cpuset`. Limits individual CPUs on multicore systems.
* `devices`. Allows or denies access to devices.
* `freezer`. Suspend or resumes processes.
* `memory`. Limits memory usage, and reports usage.
* `net_cls`. Tags network packages.
* `net_prio`. Sets priority on network traffic. 
* `ns`. Limit access to namespaces.
* `perf_event`. Identify cgroup membership of processes.


We could list all the cgroups that can be managed using `lscgroup`, which corresponds to the directories inside `/sys/fs/cgroup`.  

We could run a Docker container with some limit to explore:

```bash
# run sh in docker container named alpine using alpine image, with a 512mb memory limit
docker run --name alpine -it --rm --memory="512mb" alpine sh

# run docker stats to see it's limit
docker stats

# and we have a limit
> CONTAINER ID   NAME      CPU %     MEM USAGE / LIMIT   MEM %     NET I/O       BLOCK I/O   PIDS
> 12cf5d22a4a2   alpine    0.00%     536KiB / 512MiB     0.10%     1.16kB / 0B   0B / 0B     1

# now, on Docker host check it's max memory cgroup for the container
cat /sys/fs/cgroup/memory/docker/12cf5d22a4a2c381ed23629a5da3f221f951695f699ce9d415623a8d39e5e335/memory.limi
t_in_bytes

> 536870912

# And from inside container. 
cat /sys/fs/cgroup/memory/memory.limit_in_bytes

> 536870912
```


## capabilities

Capabilities are used by [Docker Engine](#docker-engine) to restrict permissions on a process running in a container. `containerd` runs as root with all capabilities (=ep). The capabilities a process currently have can be listed with `getpcaps`, so we can start up a new container and inspect:

```bash
docker run -d --name nginx nginx

# from where containerd is running
pid=$(ps aux | grep "nginx" | grep master | awk '{print $1}')

getpcaps $pid

>1426: cap_chown,cap_dac_override,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_net_bind_service,cap_net_raw,>cap_sys_chroot,cap_mknod,cap_audit_write,cap_setfcap=ep

```
So for instance, it has `cap_sys_chroot` which is needed by `pivot_root` to change root filesystem. It also has `cap_mknod` which is needed by some images to create special files in `/dev`. `cap_setuid` and `cap_setgid` is needed to map user and groups. In fact, many of it's capabilities are actually needed by [Docker Runtime](#docker-runtime) in order to initialize the container.


## pivot_root 

Used by [Docker Runtime](#docker-runtime) to change root filesystem to image filesystem. This is done in the process which starts up the container (PID 1) during container initialization. Following is an example how this is done:

```bash
# needed by pivot_root
mount --bind $fs_folder $fs_folder

# enter root filesystem
cd $fs_folder

# oldroot will be mounted by pivot_root
mkdir -p oldroot

# set new root
pivot_root . oldroot

# unmount oldroot, so it can be removed 
umount -l oldroot
# remove oldroot
rmdir oldroot

```
Doing that would make the root `/` point to the filesystem inside `$fs_folder`.

## Docker Engine
Docker Engine runs a daemon (service) called `dockerd` which is used by the Docker Client executable `docker`. `Dockerd` handles everything from creating networks to container management, but the actual containers run in another service called `containerd`. Normally the client connects to `dockerd` using the Docker UNIX socket file descriptor `/var/run/docker.sock`. When a container is started, an executable path within image is provided from client. And `containerd` uses a [Docker Runtime](#docker-runtime) called `runc` to isolate the process using namespaces, mount `/dev` & `/sys` filesystems, change root of filesystem using `pivot_root` and so forth. For example, running `docker run -it nginx bash` will connect to `dockerd`, which will connect to `containerd` and send command to run `bash` in `nginx:latest` image filesystem. `Containerd` will use `runc` to execute bash. And because of `-it` flags, a shared `TTY` device will be created by `containerd` that has `STDIN`, `STDOUT` & `STDERR` from bash connected to it. And it's `TTY` will be redirected by `containerd` to `docker` client, basically as a reverse shell.

## Docker Runtime
Docker containers are created by the Docker Runtime `runc`. And a container are simply an isolated environment where processes can run. So `runc` is basically a way to initialize a process with all it's [namespaces](#namespaces), [capabilities](#capabilities), [cgroups](#cgroups) and to [pivot_root](#pivot_root). `runc` need a filesystem and a `runtime configuration` in order to create a container. Or more correctly, `containerd` translates information from `OCI Image Manifest Specification` to `OCI runtime specification` and provides that to `runc`. So, we could create a base `OCI runtime specification` using `runc spec` that we could manually edit in similar way as `containerd`, and use `runc` to start up a container:

```bash
docker run --name ubuntu ubuntu
mkdir test; cd test
# export rootfs
docker export ubuntu > rootfs.tar
mkdir rootfs
tar -xf rootfs.tar -C ./rootfs

# create config.json
runc spec

# modify 
# add capabilities
#   * CAP_SETUID
#   * CAP_SETGID
# change root->readonly = false

# run container 
runc run containerid

```

> Note, that first process created inside a container is always `PID 1`. And in a Linux system it's usually `systemd` or `SysV init`. So a container doesn't do any bootstrap or management of user processes. All this is handled by [Docker Engine](#docker-engine) instead. And when `PID 1` is terminated, so is the container.


## Docker Filesystem
Root filesystems are part of an image and contains the executables needed together with all it's dependencies (userland). When an executable on this root filesystem runs in the [Docker Runtime](#docker-runtime), it's called a container. The default filesystem used by Docker is a union filesystem called `OverlayFS`, and it's just like the image format based upon layers. The top layer is where the container can make changes, and the layers below belong to the image which is immutable. So, if same image are used by multiple containers they all share the layers that belongs to the image. Both container and image layers are stored under `/var/lib/docker/overlay2`.

`OverlayFS` filesystem is part if the kernel. And it concists of following pieces:

* `LowerDir` - readonly layers.
* `UpperDir` - read/write layer.
* `MergedDir`- all layers merged.
* `WorkDir`  - used by `OverlayFS` to create `MergedDir`

> Note, if you're using Docker Desktop and WSL2, use following container to explore Docker Filesystem : `docker run -it --privileged --rm --pid=host debian nsenter -t 1 -m -u -i sh`. 

So we could inspect layers in an image and compare it to layers in a container. 
``` bash

# first, pull nginx image
docker pull nginx:latest

# and inspect it's layers
docker image inspect nginx | jq '.[0].GraphDriver.Data'
>{
>  "LowerDir": "/var/lib/docker/overlay2/9f8aa5926b47a7a07ba55cd2ce938ae1cfce32d08557bcd4a23086ef76560bef/diff:
>               /var/lib/docker/overlay2/49569d337c727a9d93a15b910c2a0fb5cb05996954a50a546002ca46231df3fd/diff:
>               /var/lib/docker/overlay2/8678c30b35e2393241ecb5288f0dbaab45e9e81213078793c05b62bf21ebfe97/diff:
>               /var/lib/docker/overlay2/856de74b0828e7523134b53f45de181a81e317e5eed3c6992ecd85fd281d0072/diff:
>               /var/lib/docker/overlay2/0c5253794034518627d1bce63c067171ef11c16767d5f5a77aa539a1b29d8f8f/diff:
>               /var/lib/docker/overlay2/a228042c51ce74cfbbae479fe7a7ceed26a45ba4a7dee392df059400202e92e6/diff",
>  "MergedDir":"/var/lib/docker/overlay2/5d6cb52f37dfbc060f91c708b38661558c22cbc522e232d087ef9009c9127f66/merged",
>  "UpperDir": "/var/lib/docker/overlay2/5d6cb52f37dfbc060f91c708b38661558c22cbc522e232d087ef9009c9127f66/diff",
>  "WorkDir":  "/var/lib/docker/overlay2/5d6cb52f37dfbc060f91c708b38661558c22cbc522e232d087ef9009c9127f66/work"
>}

# create nginx container
docker run --name nginx -d nginx:latest

# and inspect container layers
docker container inspect nginx | jq '.[0].GraphDriver.Data'

>{
>  "LowerDir": "/var/lib/docker/overlay2/ca852f913a6c93a9dd97a1219804e73c4e55d3639ab5198a97ac541aed9a2e87-init/diff:
>               /var/lib/docker/overlay2/5d6cb52f37dfbc060f91c708b38661558c22cbc522e232d087ef9009c9127f66/diff:
>               /var/lib/docker/overlay2/9f8aa5926b47a7a07ba55cd2ce938ae1cfce32d08557bcd4a23086ef76560bef/diff:
>               /var/lib/docker/overlay2/49569d337c727a9d93a15b910c2a0fb5cb05996954a50a546002ca46231df3fd/diff:
>               /var/lib/docker/overlay2/8678c30b35e2393241ecb5288f0dbaab45e9e81213078793c05b62bf21ebfe97/diff:
>               /var/lib/docker/overlay2/856de74b0828e7523134b53f45de181a81e317e5eed3c6992ecd85fd281d0072/diff:
>               /var/lib/docker/overlay2/0c5253794034518627d1bce63c067171ef11c16767d5f5a77aa539a1b29d8f8f/diff:
>               /var/lib/docker/overlay2/a228042c51ce74cfbbae479fe7a7ceed26a45ba4a7dee392df059400202e92e6/diff",
>  "MergedDir": "/var/lib/docker/overlay2/ca852f913a6c93a9dd97a1219804e73c4e55d3639ab5198a97ac541aed9a2e87/merged",
>  "UpperDir":  "/var/lib/docker/overlay2/ca852f913a6c93a9dd97a1219804e73c4e55d3639ab5198a97ac541aed9a2e87/diff",
>  "WorkDir":   "/var/lib/docker/overlay2/ca852f913a6c93a9dd97a1219804e73c4e55d3639ab5198a97ac541aed9a2e87/work"
>}
```
Ok, so if we look at LowerDir in container we see that it's same as with image, except it has 2 more layers on top of it with:

* `ca852f913a6c93a9dd97a1219804e73c4e55d3639ab5198a97ac541aed9a2e87-init` on top 
* `5d6cb52f37dfbc060f91c708b38661558c22cbc522e232d087ef9009c9127f66` below. Which is same as UpperDir on image.

And we can also see that:
`ca852f913a6c93a9dd97a1219804e73c4e55d3639ab5198a97ac541aed9a2e87` (without -init) is UpperDir in container.

Which all makes sense given that image is used in container, but readonly. And The `UpperDir` in container is where all changes are made.

So how does [Docker Runtime](#docker-runtime) make this behave like a normal filesystem? It mounts it all using the `overlay` mount type! So we could do same thing as [Docker Runtime](#docker-runtime), but mount it somewhere else:


```bash

mkdir -p /mnt/testing

mount -t overlay -o lowerdir=/var/lib/docker/overlay2/ca852f913a6c93a9dd97a1219804e73c4e55d3639ab5198a97ac541aed9a2e87-init/diff:\
/var/lib/docker/overlay2/5d6cb52f37dfbc060f91c708b38661558c22cbc522e232d087ef9009c9127f66/diff:\
/var/lib/docker/overlay2/9f8aa5926b47a7a07ba55cd2ce938ae1cfce32d08557bcd4a23086ef76560bef/diff:\
/var/lib/docker/overlay2/49569d337c727a9d93a15b910c2a0fb5cb05996954a50a546002ca46231df3fd/diff:\
/var/lib/docker/overlay2/8678c30b35e2393241ecb5288f0dbaab45e9e81213078793c05b62bf21ebfe97/diff:\
/var/lib/docker/overlay2/856de74b0828e7523134b53f45de181a81e317e5eed3c6992ecd85fd281d0072/diff:\
/var/lib/docker/overlay2/0c5253794034518627d1bce63c067171ef11c16767d5f5a77aa539a1b29d8f8f/diff:\
/var/lib/docker/overlay2/a228042c51ce74cfbbae479fe7a7ceed26a45ba4a7dee392df059400202e92e6/diff,\
upperdir=/var/lib/docker/overlay2/ca852f913a6c93a9dd97a1219804e73c4e55d3639ab5198a97ac541aed9a2e87/diff,\
workdir=/var/lib/docker/overlay2/ca852f913a6c93a9dd97a1219804e73c4e55d3639ab5198a97ac541aed9a2e87/work \
overlay /mnt/testing

```

And this is exact same filesystem that the nginx container uses. Which we can confirm:

```bash

echo "Hello!" > /mnt/testing/hello

docker exec -it nginx bash

# in container
cat /hello
> Hello!

# cleanup in host
umount /mnt/testing
rmdir /mnt/testing/
```


## Docker Image
Docker images implements the `OCI Image Manifest Specification` which basically is a manifest file that contains a list of `layers` bundled together with it's `image configuration` file. Each layer is built upon previous layers and it fits together perfectly with the default [Docker Filesystem](#docker-filesystem) called "OverlayFS". The image layers are usually located under `/var/lib/docker/overlay2/` and ech layer are represented as a folder.

> Layers could be thought of as tarballs, if these tarballs are extracted in correct order to disk, you'll get the image root filesystem.

The `image configuration` hold information about exposed ports, environment variables, which executable to run as default etc. Which is later used by the [Docker Engine](#docker-engine) to create the `OCI Runtime Specification` used by the [Docker Runtime](#docker-runtime).

To view where all layers in an image are located, and all other image related information, we can do following:

```bash
# inspect nginx image
docker image inspect nginx | jq

# layer information are found under `GraphDriver.Data`
```


## Docker Networking
> Note, if you're using Docker Desktop and WSL2, use following container to explore Docker Networking: `docker run -it --privileged --pid=host --rm ubuntu nsenter -t 1 -n bash`. Also, following packages are needed: `apt update; apt -y install iproute2 tcpdump iptables bridge-utils` 

[Docker Engine](#docker-engine) is responsible for the setup of networks. And Docker has four built-in network drivers:

* Bridge - The default network. With connectivity to an Docker bridge interface.
* Host - Allows access to same interfaces as the host.
* Macvlan - Allows for access to an interface on the host.
* Overlay - Allows for networks between different host running Docker, usually Docker Swarm clusters.

The default network bridge is `docker0`, and we can view some more information about it:

```bash
# first start a container
docker run --name nginx -p 80:80 -d nginx


brctl show docker0

>bridge name     bridge id               STP enabled     interfaces
>docker0         8000.02429f7dbd2f       no              veth0c35011

# and we see that it have one veth interfaces are attached to it.

# we can also see that this interface exist on the host
ip link show

>20: veth0c35011@if19: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP mode DEFAULT group default
>    link/ether ce:d7:73:6c:90:31 brd ff:ff:ff:ff:ff:ff link-netnsid 1

# and if we run iptables to see it's rules

iptables -L

# we'll see that it has this rule in the DOCKER chain

>Chain DOCKER (1 references)
>target     prot opt source               destination
>ACCEPT     tcp  --  anywhere             172.17.0.2           tcp dpt:http

# we can double check that this is in fact the container ip
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' nginx

>172.17.0.2

```
So what does this tell us? Well, `containerd` runs on the host and we can from this deduce that `containerd` creates the bridge `docker0`, and when we run a container, `containerd` creates a `veth` interface that belongs to the container network namespace. And the network is opened up to container ip using `iptable` rules in the `DOCKER` chain.
