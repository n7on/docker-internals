# Docker Internals
Docker is a way to isolate and manage a process. This repo show the basic concepts of Docker without using Docker. Following main kernel features are used by Docker:

### namespaces

Namespaces are used to isolate the process. So that users, hostname, network, pid's etc only are visible from it's namespace. This is the main concept of Docker Containers. Namespaces have different types, like:

* user
* cgroup
* ipc
* mnt
* uts
* pid

### cgroups

Control Groups, manages resources like memory, disk, CPU and network. So that resource limits can be added to a container.

### capabilities

Used by Docker to set permissions.

### pivot_root 

Used by Docker to change root filesystem to image filesystem.


Filesystems are called images and they contains the executables needed together with all it's dependencies (userland). When an executable on this filesystem runs in a namespace, it's called a container. It's a perfectly normal process but it's isolated from the rest of the system using kernel features above.

## Images
Docker images are basically a list of "layers". And each layer is a tarball. So if these tarballs are extracted in correct order to disk, you'll get the image filesystem. 


### Download Image
Images are usually downloaded using "docker pull", but we could do this without Docker by calling Dockers registry API's directly.  

Ex. Download alpine:latest to ~/.docker-internals/ 

``` bash

# note: in the main Docker registry, all official images are part of the "library" repository.
./image-download library/alpine:latest

```
### Create Image
Docker Images are usually created using a Dockerfile and "docker build". But we could do this without Docker by copying what we need to a folder, create a tarball and upload it to Docker repository using Docker repository API's. Executables in Linux usually have dependencies though, to shared objects (dynamic libraries) for example. So we need to add them as well. So if we would like an Image with only "ls" and "bash", we could do like this:

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

# 5. upload to Docker repository using Docker repository API's
# Before following exports is needed:
# 
# export DOCKER_USERNAME=<docker-hub-username>
# export DOCKER_PASSWORD=<docker-hub-password>
#
./image-upload <docker-hub-username>/<repository>:<tag>

```


## Containers
Container are just namespaces, and the host has access to all namespaces. So the host is also a container!? - Yes! However, we could use this container concept without Docker as well, by doing following:

* create a namespace
* mount image filesystem
* change root using pivot_root
* run an executable

So to run alpine image we could do like this.
```bash

# TODO - create this script
./run library/alpine:latest sh
```

## Other uses of Docker Images
Docker Images can also be used in other ways as well. 

### chroot
We could download the image (without Docker) and extract it to a local folder and chroot into that. This would only isolate filesystem though, so it's not really a container. 

Ex. Use image with chroot.

``` bash

# note: in the main Docker registry, all official images are part of the "library" repository.
./run-chroot library/alpine:latest sh

```

### WSL2
We could use the image in WSL2, importing the filesystem as a tarball in WSL2 will create a new WSL2 "distribution". 

WSL2 actually has a lot common with Docker, each distribution runs under same kernel, and the kernel runs in a light-weight Hyper-V VM. So all distributions share host. Meaning that a WSL2 distribution and Docker Containers are conceptually same thing. WSL2 distributions are initialized differently though, while Docker Containers basically runs one main process, WSL2 runs a normal init (like System V or System D) starting up lot's of different processes, behaving more like a Linux distribution.

Ex. Use image with WSL2

First create some folders in Windows

``` powershell
# Windows
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
