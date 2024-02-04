# Docker Internals
Docker is a way to isolate and manage a process. This is a basic introduction to the concepts of Docker without using Docker. 

Following main kernel features are used by Docker:
### namespaces

Namespaces are used to isolate processes. So that users, hostname, network, pid's etc only are visible from it's namespaces. This is the main concept of Docker Containers. Namespaces have 8 different types like net, mnt, uts, pid etc. And each namespace is held by at least 1 process. And a process can only belong to one namespace for each type at a given time. So, if we for example start Nginx in Docker, we could list it's namespaces: 

```bash
# start nginx
docker run -it -d -p 8080:80 nginx 

# list all namespaces used by that process
ps aux | grep nginx | awk '$5 == "master" {print $1}' | xargs lsns -n -p | awk '{print $1, $2}'
4026531834 time
4026531835 cgroup
4026531837 user
4026532718 mnt
4026532719 uts
4026532720 ipc
4026532721 pid
4026532722 net
```

Namespaces can be created using `unshare` command, and process started in namespaces using `nsenter`.

### cgroups

Control Groups, manages resources like memory, disk, CPU and network. So that resource limits can be added to a container.

### capabilities

Used by Docker to set permissions.

### pivot_root 

Used by Docker to change root filesystem to image filesystem.


## Docker Runtime
Docker Engine runs a daemon called containerd, which will provide a service that can be used for managing Docker containers. Such as starting, stopping or pulling images. This service is used by the Docker Client executable `docker`. Normally the client connects to containerd using the Docker UNIX socket file descriptor `/var/run/docker.sock`. When a container is started, an executable path within image is provided from client. And Docker uses a Docker Runtime called `runc` to isolate the process using namespaces, mount `/dev` & `/sys` filesystems, change root of filesystem using `pivot_root` and so forth. For example, `docker run -it nginx bash` will connect to containerd and send command to run `bash` in `nginx:latest` image filesystem. Containerd will use `runc` to execute bash. And because of `-it` flags, it will provide the `docker` client with a reverse shell to the `bash` process.

## Filesystem
Root filesystems are part of an image and contains the executables needed together with all it's dependencies (userland). When an executable on this root filesystem runs in the Docker Runtime, it's called a container. It's a perfectly normal process but it's isolated from the rest of the system using kernel features listed above. The filesystem used by Docker is called `overlay`, and it's just like the image format based upon layers. The top layer is where the container can make changes, and the layers below belong to the image which is immutable. So same image can be shared by multiple containers.  

This introduction will not dig any further into the filesystem though. Images will be flattened and written to current filesystem to simplify in order to make other concepts more transparent. 

## Images
Docker images are basically a list of "layers" bundled together with it's runtime configuration. Each layer is built upon previous layers, but when using it it appears as flattened. Layers could be thought of as tarballs, if these tarballs are extracted in correct order to disk, you'll get the image root filesystem. Images format is based on a manifest, which is a json file that holds all layer information and the image runtime configuration. This information is used by Docker to create the `overlay` filesystem. And the runtime configuration file hold information about what namespaces to use, which executable to run as default, capabilities etc. Which is later used by the Docker Runtime.


### Download Image
Images are usually downloaded using "docker pull", but we could do this without Docker by calling Docker registry API's directly.  

Following script requests the manifest, downloads the layers and extracts it to the fileystem in correct order under `~/.docker-internals/<docker-hub-username>/<repo>/<tag>`. Making it visible and usable in order to explore it further.  

Ex. Download alpine:latest to `~/.docker-internals/` 

``` bash

# note: in the main Docker Registry, all official images are part of the "library" "user".
./image-download library/alpine:latest

```

### Upload Image
Images are usually uploaded using `docker push`, but we could do this without Docker by calling Docker Registry API's directly using [./image-upload](./image-upload). You probably would want to first use [./image-download](./image-download) to download some other image to work on, and move that to `~/.docker-internals/<docker-hub-username>/`

[./image-upload](./image-upload) expects following environment variables to be exported before running.

```bash
export DOCKER_USERNAME=<your username>
export DOCKER_PASSWORD=<your password>

```

Ex. Move Nginx image/filesystem so it can be altered and uploaded to your own Docker Repo.
```bash
mkdir -p ~/.docker-internals/<docker-hub-username>
mv ~/.docker-internals/library/nginx ~/.docker-internals/<docker-hub-username>/nginx

```

Ex. Upload `~/.docker-internals/<docker-hub-username>/alpine/latest` to `<docker-hub-username>/alpine:latest`  
``` bash

# note: in the main Docker registry, all official images are part of the "library" repository.
./image-upload <docker-hub-username>/alpine:latest

```


### Create Image
Docker Images are usually created using a Dockerfile and `docker build`. But we could do this without Docker by copying what we need to `/.docker-internals/<docker-hub-username>/<repository>/<tag>/` path and run [./image-upload](./image-upload). Which upload the manifest, configuration file and a single layer (as a tarball) to Docker repository using it's API's. Executables in Linux usually have dependencies though, to shared objects (dynamic libraries) for example, so we need to add them as well. Given that we would like an Image with only `ls` and `bash`, we could do like following to upload it to our own Docker Repo, as a base-image:

```bash
# 1. create folders
path=~/.docker-internals/<docker-hub-username>/<repository>/<tag>/
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
Docker containers are created by the [Docker Runtime](#docker-runtime) where `containerd` is used for managing the container lifecycle (start, stop etc). And `runc` is used as it's container runtime. And a Container Runtime is basically how a process is isolated. So containers are just normal processes that holds namespaces. We could run an executable inside an image filesystem that holds a couple of namespaces without using Docker. And instead use [./container-namespace](./container-namespace), which does following:

* Start up process with it's own namespaces (uts, mount, pid, net and ipc)
* Mount image root filesystem
* Mount /proc

When a namespace is created, normally the current namespace is copied. So for example `uts` still has it's hosts hostname. But if it's changed within the namespace, it's isolated from host. Same thing with `mounts`. But because `--mount-proc` and `--root` are given, `/proc` is mounted to new root filesystem and therefore resetting both `pid` and `mounts`. 

The `net` namespace will not have it's host network information copied though, which means it will only have a loopback device per default, making this container not able to connect to Internet. This could be achived by creating a virtual network device at the host, and attach that to the `net` namespace.

This example is not using `pivot_root` but instead uses `unshare` with `--root` switch. The reason for that is because `pivot_root` basically need to be done for each process, i.e. it's not maintained in any namespace. Making it hard to do this automatically in bash. `unshare` & `pivot_root` commands are just thin layers of it's `syscalls`.

So to run sh inside alpine image, as a container, we could do like this.
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
