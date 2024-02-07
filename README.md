# Docker Internals
Docker is a way to isolate processes, using virtually every possible way in Linux kernel, but mainly `namespaces`. This is a basic introduction to the concepts of Docker without using Docker. 

Following main kernel features are used by Docker:
### namespaces

Namespaces are used to isolate processes. So that users, hostname, network, pid's etc only are visible from it's namespaces. This is the main concept of Docker Containers. Namespaces have 8 different types: 
* `net`. Network interfaces namespace.
* `mnt`. Mount namespace.
* `uts`. Hostname namespace.
* `pid`. Process namespace.
* `user`. User namespace (doesn't require privileged account).
* `time`. System time namespace.
* `ipc`. Inter-Process Communication namespace. Like shared memory, message queues and semaphores.
* `cgroup`. Cgroup namespace.  

Each namespace is held by at least 1 process. And a process can only belong to one namespace for each type at a given time. And all processes per default actually belong to one of each types in the `default namespaces`. So in that sense, `the host is also a container`. This can be visualized using `lsns`:

```bash
# list the init process namespaces
sudo lsns -p 1

4026531834 time       70   1 root /sbin/init
4026531835 cgroup     70   1 root /sbin/init
4026531837 user       70   1 root /sbin/init
4026531840 net        70   1 root /sbin/init
4026532266 ipc        70   1 root /sbin/init
4026532277 mnt        67   1 root /sbin/init
4026532278 uts        68   1 root /sbin/init
4026532279 pid        70   1 root /sbin/init

```
And all other processes will inherit namespaces from it's parent process. So if we do same thing for the current shell process, we get exactly same namespace id's as with `init`: 

```bash
lsns -p $$ | awk '{print $1,$2}'

4026531834 time
4026531835 cgroup
4026531837 user
4026531840 net
4026532266 ipc
4026532277 mnt
4026532278 uts
4026532279 pid
```

We could start a new shell with new `uts` namespace using `unshare` command, which is a command that is a wrapper of the `syscall` with same name. And it's used in order to un-share a process from `default namespaces`. So we could start bash using `unshare` (to un-share from `uts` namespace) and list it's `uts` namespace id:  

> Note that `unshare` need root to create all types of namespaces, except `user`. And this is also why Docker need root.  

```bash
sudo unshare --uts bash
# now running bash in new uts namespace

lsns -p $$

# and these namespaces is same as for init
4026531834 time       71     1 root /sbin/init
4026531835 cgroup     71     1 root /sbin/init
4026531837 user       71     1 root /sbin/init
4026531840 net        71     1 root /sbin/init
4026532266 ipc        71     1 root /sbin/init
4026532277 mnt        68     1 root /sbin/init
4026532279 pid        71     1 root /sbin/init

# but uts namespace have a new id
4026536218 uts         2 74238 root bash
```

Another way a process namespaces could be viewed is by exploring the `/dev` filesystem (which are a dynamically mounted filesystem of type `proc`). For each process we have `/dev/<pid>/ns/<namespace>`, so we could show the namespaces of `init` (PID 1) using filesystem as well.

```bash
sudo ls -l /proc/1/ns

lrwxrwxrwx 1 root root 0 Feb  4 16:26 cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx 1 root root 0 Feb  4 16:26 ipc -> 'ipc:[4026532266]'
lrwxrwxrwx 1 root root 0 Feb  4 16:26 mnt -> 'mnt:[4026532277]'
lrwxrwxrwx 1 root root 0 Feb  4 16:26 net -> 'net:[4026531840]'
lrwxrwxrwx 1 root root 0 Feb  4 16:26 pid -> 'pid:[4026532279]'
lrwxrwxrwx 1 root root 0 Feb  7 18:56 pid_for_children -> 'pid:[4026532279]'
lrwxrwxrwx 1 root root 0 Feb  4 16:26 time -> 'time:[4026531834]'
lrwxrwxrwx 1 root root 0 Feb  7 18:56 time_for_children -> 'time:[4026531834]'
lrwxrwxrwx 1 root root 0 Feb  4 16:26 user -> 'user:[4026531837]'
lrwxrwxrwx 1 root root 0 Feb  4 16:26 uts -> 'uts:[4026532278]'

# or for specific namespace, like "mnt"
sudo readlink /proc/1/ns/mnt

```


### cgroup

Control Group (also called resource controllers), is a way to manage resources like memory, disk, CPU, network etc. So that resource limits can be added to a container, and usage can be extracted. Cgroup is structured like multiple separate hierarchies under `/sys/fs/cgroup`. Which contains each of it's subsystems. And a cgroup could isolated from host using it's cgroup namespace. Following are some of these subsystem:

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


We could list all the cgroups that can be managed using `lscgroup`, which corresponds to the directories inside `/sys/fs/cgroup`. When a Docker container is started, Docker Runtime will create a new child group named `docker/<container id>` under each subsystem in it's cgroup namespace. And we could run a Docker container with some limit, to explore:

```bash
# run sh in docker container named alpine using alpine image 
docker run --name alpine -it --rm --memory="512mb" alpine sh

# run docker stats to see it's limit
docker stats

CONTAINER ID   NAME      CPU %     MEM USAGE / LIMIT     MEM %     NET I/O       BLOCK I/O   PIDS
c35d8b3ed3aa   alpine    0.00%     496KiB / 512MiB       0.09%     586B / 0B     0B / 0B     1

```

### capabilities

Used by Docker to set permissions.

### pivot_root 

Used by Docker to change root filesystem to image filesystem.


## Docker Runtime
Docker Engine runs a daemon called containerd, which will provide a service that can be used for managing Docker containers. Such as starting, stopping or pulling images. This service is used by the Docker Client executable `docker`. Normally the client connects to containerd using the Docker UNIX socket file descriptor `/var/run/docker.sock`. When a container is started, an executable path within image is provided from client. And Docker uses a Docker Runtime called `runc` to isolate the process using namespaces, mount `/dev` & `/sys` filesystems, change root of filesystem using `pivot_root` and so forth. For example, `docker run -it nginx bash` will connect to containerd and send command to run `bash` in `nginx:latest` image filesystem. Containerd will use `runc` to execute bash. And because of `-it` flags, a shared `TTY` device will be created in host that has `STDIN`, `STDOUT` & `STDERR` from bash connected to it. And it's `TTY` will be redirected by `containerd` to `docker` client, basically as a reverse shell.

## Filesystem
Root filesystems are part of an image and contains the executables needed together with all it's dependencies (userland). When an executable on this root filesystem runs in the Docker Runtime, it's called a container. It's a perfectly normal process but it's isolated from the rest of the system using kernel features listed above. The filesystem used by Docker is a union filesystem called `overlay`, and it's just like the image format based upon layers. The top layer is where the container can make changes, and the layers below belong to the image which is immutable. So, if same image are used by multiple containers, it's shared. 

This introduction will not dig any further into the filesystem though. Images will be flattened and written to current filesystem to simplify in order to make other concepts more transparent. 

## Images
Docker images are basically a manifest file which contains a list of "layers" bundled together with it's runtime configuration. Each layer is built upon previous layers, but when using it it appears as flattened. Layers could be thought of as tarballs, if these tarballs are extracted in correct order to disk, you'll get the image root filesystem. The manifest is used by Docker to create the `overlay` filesystem. And the runtime configuration file hold information about what namespaces to use, which executable to run as default, capabilities etc. Which is later used by the Docker Runtime.


### Download Image
Images are usually downloaded using `docker pull`, but we could do this without Docker by calling Docker registry API's directly.  

Following script requests Docker registry API's to fetch the manifest, downloads the layers and extracts it to the fileystem in correct order under `~/.docker-internals/images/<docker-hub-username>/<repo>/<tag>`. Making usable in order to explore it further.  

Ex. Download alpine:latest to `~/.docker-internals/images` 

``` bash

# note: in the main Docker Registry, all official images are part of the "library" "user".
./image-download library/alpine:latest

```

### Upload Image
Images are usually uploaded using `docker push`, but we could do this without Docker by calling Docker Registry API's directly using [./image-upload](./image-upload). You probably would want to first use [./image-download](./image-download) to download some other image to work on, and move that to `~/.docker-internals/images/<docker-hub-username>/`

[./image-upload](./image-upload) expects following environment variables to be exported before running.

```bash
export DOCKER_USERNAME=<your username>
export DOCKER_PASSWORD=<your password>

```

Ex. Move Nginx image/filesystem so it can be altered and uploaded to your own Docker Repo.
```bash
mkdir -p ~/.docker-internals/images/<docker-hub-username>
mv ~/.docker-internals/images/library/nginx ~/.docker-internals/images/<docker-hub-username>/nginx

```

Ex. Upload `~/.docker-internals/images/<docker-hub-username>/alpine/latest` to `<docker-hub-username>/alpine:latest`  
``` bash

# note: in the main Docker registry, all official images are part of the "library" repository.
./image-upload <docker-hub-username>/alpine:latest

```


### Create Image
Docker Images are usually created using a Dockerfile and `docker build`. But we could do this without Docker by copying what we need to `/.docker-internals/images/<docker-hub-username>/<repository>/<tag>/` path and run [./image-upload](./image-upload). Which upload the manifest, configuration file and a single layer (as a tarball) to Docker repository using it's API's. Executables in Linux usually have dependencies to shared objects (dynamic libraries), so we need to add them as well. With that in mind, we would create an Image with only `ls` and `bash`, and upload it to our own Docker Repo, as a base-image:

```bash
# 1. create folders
path=~/.docker-internals/images/<docker-hub-username>/<repository>/<tag>/
mkdir -p $path/{bin,lib}

cd $path

# 2. copy "ls" and "bash"
cp $(which bash) $(which ls) ./bin

# print shared object dependencies
ldd ./bin/ls

# 3. copy shared objects to our "lib"

# selinux
cp /lib/x86_64-linux-gnu/libselinux.so.1 ./lib
# glibc
cp /lib/x86_64-linux-gnu/libc.so.6 ./lib
# pcre
cp /lib/x86_64-linux-gnu/libpcre2-8.so.0 ./lib
cp /lib64/ld-linux-x86-64.so.2 ./lib
# above would need to be in lib64, so we just create that as a link.
ln -s lib lib64 

# What about linux-vdso.so.1? That is actually a kernel module loaded from memory.

# Now do same thing for "bash". Only 1 extra object to be added
ldd ./bin/ls
cp /lib/x86_64-linux-gnu/libtinfo.so.6 ./lib

# 4. Optional: run it in chroot, to test that it works.
# sudo chroot ./test bash

# 5. upload to Docker Registry using it's API's
# Before following exports is needed:
# 
# export DOCKER_USERNAME=<docker-hub-username>
# export DOCKER_PASSWORD=<docker-hub-password>
#
./image-upload <docker-hub-username>/<repository>:<tag>

```


## Containers
Docker containers are created by the [Docker Runtime](#docker-runtime). Where `containerd` is used for managing the container lifecycle (start, stop etc). And `runc` is used as it's container runtime. A Container Runtime is basically how a process is isolated. And containers are just normal processes that holds namespaces. We could run an executable inside an image filesystem that holds a couple of namespaces without using Docker. And instead use [./container-namespace](./container-namespace), which does following:

* `unshare` creates the namespaces uts, mount, pid, net and ipc. And runs `init` __inside the namespaces created__. The `--fork` flag is also given. Otherwise no new other processes could be created in the namespace.
* Directory above the root filesystem is mounted to itself. 
* oldroot directory is created, which is needed by `pivot_root`.
* `pivot_root` is used to change root to new root filesystem.
* `proc` filesystem is mounted to `/proc`.
* oldroot is unmounted and removed.
* hostname is changed to image name.
* The bash process is __replaced__ with the intended process, which is part of the new root filesystem. This is needed because the process started need to be PID 1. 

When a namespace is created, normally the current namespace context is copied. So for example `uts` still has it's hosts hostname. But if it's changed within the namespace, it's isolated from host. Same thing with `mnt` namespace, but it will be reset because `/proc` is mounted to new root filesystem.  

The `net` namespace will not have it's host network information copied though, which means it will only have a loopback device per default. So a virtual network device need to be created outside container, and shared with the container `net` namespace.

To run sh inside alpine image, as a container, we could do like this.
```bash

./container-namespaces library/alpine:latest sh
```

## Other uses of Docker Images
Docker Image root filesystems can also be used in other ways. Because when we [Download Image](#download-image) it's separated from Docker and it's image concept, and it could be used for whatever purpose. 

### chroot
We could download the image without Docker (using [./image-download](./image-download)) and chroot into the extracted root filesystem. This would only isolate filesystem though, so it doesn't qualify as a container. It's convenient though, for testing and exploring an image.

Ex. Use image root filesystem with chroot.

``` bash

# note: in the main Docker Registry, all official images are part of the "library" "user".
./container-chroot library/alpine:latest sh

```

### WSL2
We could use the image root filesystem in WSL2, importing the filesystem as a tarball in WSL2 will create a new WSL2 distribution. 

> WSL2 actually has a lot common with Docker, each distribution runs under same kernel, and the kernel runs in a light-weight Hyper-V VM, so all distributions share host (kernel). Meaning that a WSL2 distribution and Docker Containers are conceptually same thing. But WSL2 distributions use ext4 filesystem and are initialized differently. And while Docker Containers basically runs one main process, WSL2 runs a normal init like System V or SystemD. And thus behave more like a normal Linux distribution.

Docker Desktop for Windows can be used with WSL2, which will create two distributions called docker-desktop & docker-desktop-data. The one where Docker Runtime runs is docker-desktop.

We could use any Docker image in WSL2 by doing following:

Ex. Use image with WSL2

``` powershell
# First create some folders in Windows
$Path = c:\WSLDistros\alpine
New-Item -ItemType Directory -Path $Path

```
And copy tarball to that folder  
```bash
# Linux

# download
./image-download library/alpine:latest

# pack
tar -czvf alpine.tar.gz -C ~/.docker-internals/library/alpine/latest .

# copy
cp alpine.tar.gz /mnt/c/WSLDistros/
```
And import it into WSL2.
``` powershell
# Windows again

wsl.exe --import "alpine" C:\WSLDistros\alpine\ C:\WSLDistros\alpine.tar.gz
```
