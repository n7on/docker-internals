# Docker Internals
Docker is a way to isolate a process from the rest of system using kernel features such as [namespaces](#namespaces), [cgroups](#cgroups), [capabilities](#capabilities) and [pivot_root](#pivot_root). When these features are used in conjunction to create an isolated environment, it's called a container. When a new container is created using [Docker Engine](#docker-engine) a [Docker Image](#docker-image) need to be provided. The root filesystem in the image is added to [Docker Filesystem](#docker-filesystem). And the runtime configuration in the image is used by the [Docker Runtime](#docker-runtime) to create the process inside the container.  


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

> 4026531834 time       70   1 root /sbin/init
> 4026531835 cgroup     70   1 root /sbin/init
> 4026531837 user       70   1 root /sbin/init
> 4026531840 net        70   1 root /sbin/init
> 4026532266 ipc        70   1 root /sbin/init
> 4026532277 mnt        67   1 root /sbin/init
> 4026532278 uts        68   1 root /sbin/init
> 4026532279 pid        70   1 root /sbin/init

```
And all other processes will inherit namespaces from it's parent process. So if we do same thing for the current shell process, we get exactly same namespace id's as with `init`: 

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

Another way a process namespaces could be viewed is by exploring the `/proc` filesystem. Which are a pseudo filesystem provided by the kernel. For each process we have `/proc/<pid>/ns/<namespace>`, so we could list the namespaces of `init` (PID 1) using filesystem as well.

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
Control Group (also called resource controllers), is a way to manage resources like memory, disk, CPU, network etc. So that resource limits can be added to a container, and usage can be extracted. Cgroup is structured in multiple separate hierarchies under `/sys/fs/cgroup`. Which contains each of it's subsystems. And a `cgroup` is isolated from host using it's `cgroup` namespace together with `cgroups` mounted from the host. When a Docker container is started, Docker Runtime will create a new child group named `docker/<container id>` on the host under each subsystem. The host `cgroup` namespace will be copied, and if a limit is added it will be changed in the namespace. Following are some of the cgroup subsystem:

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
So for instance, it has `cap_sys_chroot` which is needed by `pivot_root` to change root filesystem. It also has `cap_mknod` which is needed by some images to create special files in `/dev`. `cap_setuid` and `cap_setgid` is needed to map user and groups. In fact, most of it's capabilities are actually needed in order to initialize the container.


## pivot_root 

Used by Docker to change root filesystem to image filesystem. This is done in the process which starts up the container (PID 1). Following is an example how this is done:

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
Docker Engine runs a daemon called containerd, which will provide a service that can be used for managing Docker containers. Such as starting, stopping or pulling images. This service is used by the Docker Client executable `docker`. Normally the client connects to containerd using the Docker UNIX socket file descriptor `/var/run/docker.sock`. When a container is started, an executable path within image is provided from client. And Docker uses a Docker Runtime called `runc` to isolate the process using namespaces, mount `/dev` & `/sys` filesystems, change root of filesystem using `pivot_root` and so forth. For example, `docker run -it nginx bash` will connect to containerd and send command to run `bash` in `nginx:latest` image filesystem. Containerd will use `runc` to execute bash. And because of `-it` flags, a shared `TTY` device will be created in host that has `STDIN`, `STDOUT` & `STDERR` from bash connected to it. And it's `TTY` will be redirected by `containerd` to `docker` client, basically as a reverse shell.

## Docker Runtime
Docker containers are created by the Docker Runtime `runc`. And a container are simply an isolated environment where processes can run. `runc` need a filesystem and a `runtime configuration` in order to create a container. So to use `runc` directly we could do following:

```bash
docker run --name ubuntu ubuntu
mkdir test; cd test
# export rootfs
docker export ubuntu > rootfs.tar
mkdir rootfs
tar -xf rootfs.tar -C ./rootfs

# create config.json
runc spec

# run container 
runc run containerid

```

The first process created inside a container is always `PID 1`. And in a Linux system this usually `systemd` or `SysV init`. So a container doesn't do any bootstrap or management of user processes. All this is handled by [Docker Engine](#docker-engine) instead. And when `PID 1` is terminated, so is the container.


## Docker Filesystem
Root filesystems are part of an image and contains the executables needed together with all it's dependencies (userland). When an executable on this root filesystem runs in the [Docker Runtime](#docker-runtime), it's called a container. The filesystem used by Docker is a union filesystem called `overlay`, and it's just like the image format based upon layers. The top layer is where the container can make changes, and the layers below belong to the image which is immutable. So, if same image are used by multiple containers, it's shared. 


## Docker Image
Docker images are basically a manifest file which contains a list of "layers" bundled together with it's runtime configuration. Each layer is built upon previous layers, but when using it it appears as flattened. Layers could be thought of as tarballs, if these tarballs are extracted in correct order to disk, you'll get the image root filesystem. The manifest is used by Docker to create the `overlay` filesystem. And the runtime configuration file hold information about what namespaces to use, which executable to run as default, capabilities etc. Which is later used by the [Docker Runtime](#docker-runtime).
