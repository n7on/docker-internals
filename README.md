# Docker Internals

# Images
Docker images are basically a list of "layers". And a layer are a tarball. So if these tarballs are extracted in correct order to disk, you'll get the Image.

Ex. Download alpine:latest to ~/.docker-internals/ 

``` bash

# note: in the main Docker registry, all official images are part of the "library" repository.
./download-image library/alpine latest

```

# Running
An image can obviously be used by a container in Docker. But it can also be used in other ways, like with chroot or WSL2 for example. 

Ex. Use image with chroot.

``` bash

# note: in the main Docker registry, all official images are part of the "library" repository.
./run library/alpine latest sh

```