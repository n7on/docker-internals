# Docker Internals
Docker is a way to isolate and manage a process. Two main kernel features are used in order to do that:

* namespaces
Namespaces are used to isolate the process. So that users, groups, hostname, network, pid's etc only are visible from correct namespace. This is basically the main concept of a Docker Container.

* cgroups
Manages resources like memory, disk, CPU and network. So that resource limits can be added to a Container.

The filesystem is isolated in a similar way as with chroot (change root), but Docker uses namespaces instead. Filesystems are called images and they contains the executables needed together with all it's dependencies (this is user-land). When an executable on this filesystem runs in a namespace, it's called a Container. It's a normal process but it's isolated from the rest of the system (using namespaces), but it still uses the same Kernel (kernel-land). A container usually runs some kind of service, like a webserver. And when attaching to a Container, basically what happens is that a new process (usually a shell) are executed in the same namespace, and the terminal is attached to this shell.

# Images
Docker images are basically a list of "layers". And each layer is a tarball. So if these tarballs are extracted in correct order to disk, you'll get the image filesystem. Docker does this automatically, but we could do this without Docker by calling Dockers registry API's directly.  

Ex. Download alpine:latest to ~/.docker-internals/ 

``` bash

# note: in the main Docker registry, all official images are part of the "library" repository.
./download-image library/alpine:latest

```


# Use Image
An image (filesystem) can obviously be "associated" with a container (namespace) in Docker, but we could do this outside Docker as well using "unshare" command to create a namespace. And cgexec to make it use cgroups.

```bash

./namespace library/alpine:latest sh
```


It can also be used in other ways, like with chroot. So we could download the image (without Docker) and extract it to some local folder and chroot into that, simulating "docker run".

Ex. Use image with chroot.

``` bash

# note: in the main Docker registry, all official images are part of the "library" repository.
./chroot library/alpine:latest sh

```

We could also download and extract an image without Docker, and create a single tarball. This tarball could then be imported into Docker, as an image. This is somehow ridiculous, but gives us some insight.

Ex. Use image with docker
```bash
# download
./download-image library/alpine:latest

# pack
tar -czvf alpine.tar.gz -C ~/.docker-internals/library/alpine/latest .

# import
cat alpine.tar.gz | docker import --message "import test" - alpineimport:latest

# run
docker run -it --rm alpineimport:latest sh
```

Or we could use the image in WSL2, importing the tarball will create a new WSL2 "distribution". 

WSL2 actually has a lot common with Docker, each distribution runs under same kernel, and the kernel runs in a light-weight Hyper-v VM. So all distributions share host, meaning that a WSL2 distribution and Docker Containers are conceptually same thing. WSL2 distributions is initialized differently though, while Docker Containers basically runs 1 main process, WSL2 runs a normal init (like System V or SystemD) starting up lot's of different processes, behaving more like an distribution.

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
./download-image library/alpine:latest

# pack
tar -czvf alpine.tar.gz -C ~/.docker-internals/library/alpine/latest .

# copy
cp alpine.tar.gz /mnt/c/WSLDistros/
```

``` powershell
# Windows again

wsl.exe --import "alpine" C:\WSLDistros\alpine\ C:\WSLDistros\alpine.tar.gz
```



# Create Image
Docker Images are usually created with Docker using a Dockerfile. But we could actually do this without Docker by copying what we need to a folder, create a tarball and import it into Docker. Executables in Linux usually have dependencies though, to shared objects (dynamic libraries) for example. So we need to copy them as well. So if we would like an Image with only "ls" and "bash", we could do like this:

```bash
# 1. create folders
mkdir -p test/{bin,lib}

# 2. copy "ls" and "bash"
cp $(which bash) $(which ls) test/bin

# print shared object dependencies
ldd test/bin/ls

# 3. copy shared objects to our "lib"

# selinux
cp /lib/x86_64-linux-gnu/libselinux.so.1 test/lib
# glibc
cp /lib/x86_64-linux-gnu/libc.so.6 test/lib
# pcre
cp /lib/x86_64-linux-gnu/libpcre2-8.so.0 test/lib
cp /lib64/ld-linux-x86-64.so.2 test/lib
# above would need to be in lib64, so we just create that as a link.
cd test # because paths need to be correct
ln -s lib lib64 
cd ..

# What about linux-vdso.so.1? That is actually a kernel module loaded from memory.

# Now do same thing for "bash". Only 1 extra object to be added
ldd test/bin/ls
cp /lib/x86_64-linux-gnu/libtinfo.so.6 test/lib

# 4. chroot into it, to test it
sudo chroot ./test bash

# or just run ls in chroot environment
sudo chroot ./test ls /

# 5. create tarball

tar -czvf ourimage.tar.gz -C ./test .

# import
cat ourimage.tar.gz | docker import --message "import ourimage" - ourimage:latest

# run
docker run -it --rm ourimage:latest bash

# See the resemlance between "docker run" and chroot. It's same concept. 
```

This method of creating images is basically how base images are born. 

