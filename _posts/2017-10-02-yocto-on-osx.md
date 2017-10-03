---
layout: post
title: "Building with Yocto on macOS"
date: 2017-10-02
---
There are a few investigations that I would like to perform which involve
building GNU/Linux systems. To this end, I'll need a way to build such
systems from source. One such tool for building full GNU/Linux distributions
is [Yocto](https://www.yoctoproject.org/). For my investigations I'll use
Yocto to build a simple system image. Following is an explanation of building
using Yocto on macOS.

### Yocto is not supported on macOS

My primary computer is a Mac laptop. Building with Yocto on GNU/Linux is straight
forward; however it
[does not support](https://lists.yoctoproject.org/pipermail/yocto/2017-January/033878.html)
macOS. To build with Yocto on macOS I'll need to either build in a VM (for
example creating a Linux VM using VirtualBox) or use Docker. Docker might
be a little easier to work with, so I'll take that route.

### Docker on macOS

Docker is software project that allows one to create containers, each of which
contain all the binaries necessary to execute a given program or duplicate
a development environment. Isolation between containers is accomplished by using
features in the Linux kernel such as cgroups and kernel namespaces. On macOS
these features are not available. Instead, [Docker on macOS](https://docs.docker.com/docker-for-mac/docker-toolbox/#the-docker-for-mac-environment
) uses a Linux VM
based on Alpine Linux which runs on HyperKit, a lightweight macOS virtualization
solution. The Docker application for macOS is available [here](https://docs.docker.com/docker-for-mac/install/).

After installing Docker, I'll need an image with the tools necessary build with
Yocto. Luckily, such an image already exists:
[gmacario/build-yocto](https://hub.docker.com/r/gmacario/build-yocto/).
This image is based on an Ubuntu 16.04 image but with additional tools
needed by Yocto.

Creating a new container and testing it is as simple as first downloading
the image:

{% highlight shell_session %}
$ docker pull gmacario/build-yocto
{% endhighlight %}

and running an instance of the container. The following will start a new
container, run ```echo```, then destroy the container:

{% highlight shell_session %}
$ docker run gmacario/build-yocto echo hi
hi
{% endhighlight %}

The image contains everything necessary to build with Yocto. However,
by default any changes made in the container do not survive once it is
destroyed. Something will need to persist data between build, else restarting
the container will lose all its data. The simplest way to do this is to allow
a path on macOS to appear in the container, then let Yocto build using that
path. For example, the following docker argument will let the container access
a local directory called "yocto" and access it from /home/build/yocto:

{% highlight shell_session %}
--volume=${PWD}/yocto:/home/build/yocto
{% endhighlight %}

Unfortunately, if Yocto were to build its files using that directory the
build would fail; during the build some hardlinks are made, and the macOS
file system does not support this in the same way that Linux expects. As a
result, the build fails.

Instead, a [Docker volume](https://docs.docker.com/engine/admin/volumes/volumes/)
can be used. A volume is the preferred way to persist data in a container.
Docker will manage the volumes itself and the file system will be consistent
with what the container expects.

To create a volume, for example called yoctovolume:

{% highlight shell_session %}
$ docker volume create --name yoctovolume
{% endhighlight %}

The volume can now be given to a container when it starts. When this happens,
the volume will by accessibly only by root. The
build-yocto image runs as user __build__ by default. To allow that user to access
the volume, the permissions need to be changed first:

{% highlight shell_session %}
$ docker run -it --rm -v yoctovolume:/workdir gmacario/build-yocto sudo chown -R build:build /workdir
{% endhighlight %}

When the volume is given to a container it will now be accessible by the build user.

### Building with Yocto on Docker

Now that an image is setup, building an example GNU/Linux distribution with Yocto
is straight forward. Yocto will be used to create a QEMU image of a minimalistic
distribution which simply boots and provides a shell.

From here on, `mac$` shows a terminal on the Mac and `docker$` shows a
terminal inside docker.

The first step is to clone the Yocto metadata and tools. Note that Yocto only
officially supports building on some GNU/Linux distributions. The docker image is
based on Ubuntu-16.04. Because of this, Yocto 2.2 (Morty) is used, as it
officially supports that version of Ubuntu.

To start the Docker container and clone the metadata:

{% highlight shell_session %}
mac$ docker run -it -v yoctovolume:/workdir gmacario/build-yocto
docker$ cd /workdir
docker$ git clone -b morty git://git.yoctoproject.org/poky.git
{% endhighlight %}

Once the metadata and tools are downloaded, a build needs to be configured. This is
accomplished by:

{% highlight shell_session %}
docker$ source /workdir/poky/oe-init-build-env /workdir/build
{% endhighlight %}

where /workdir/build is the output directory for the configuration and build
artifacts. Finally, the image is built using the bitbake tool. The simplest
image is attempted: core-image-minimal. The -k argument is given so that if a
build failure occurs bitbake will complete as much as it can before giving up.

{% highlight shell_session %}
docker$ bitbake -k core-image-minimal
...
NOTE: Tasks Summary: Attempted 2048 tasks of which 9 didn't need to be rerun and all succeeded.

Summary: There were 7 WARNING messages shown.
{% endhighlight %}

The build will take some time, as tarballs are fetched and everything is built from
scratch, including the kernel on up as well as tools used during the build. On
my 2015 MacBook Pro with a 2.9 GHz Intel Core I5-5257U
(docker was given access to two cores as well as 2GB of RAM) the build took ~3 hours.
When the build completed the QEMU image and kernel file were located
under tmp/deploy/images/${MACHINE}:

{% highlight shell_session %}
docker$ ls -l /workdir/build/tmp/deploy/images/qemux86/
total 20228
lrwxrwxrwx 2 build build      72 Oct  2 05:44 bzImage -> bzImage--4.8.24+git0+c84532b647_f6329fd287-r0-qemux86-20171002031647.bin
-rw-r--r-- 2 build build 7046656 Oct  2 05:44 bzImage--4.8.24+git0+c84532b647_f6329fd287-r0-qemux86-20171002031647.bin
lrwxrwxrwx 2 build build      72 Oct  2 05:44 bzImage-qemux86.bin -> bzImage--4.8.24+git0+c84532b647_f6329fd287-r0-qemux86-20171002031647.bin
-rw-r--r-- 1 build build    1170 Oct  2 05:45 core-image-minimal-qemux86-20171002031647.qemuboot.conf
-rw-r--r-- 2 build build 9717760 Oct  2 05:45 core-image-minimal-qemux86-20171002031647.rootfs.ext4
-rw-r--r-- 2 build build     788 Oct  2 05:45 core-image-minimal-qemux86-20171002031647.rootfs.manifest
-rw-r--r-- 2 build build 2604376 Oct  2 05:45 core-image-minimal-qemux86-20171002031647.rootfs.tar.bz2
lrwxrwxrwx 2 build build      53 Oct  2 05:45 core-image-minimal-qemux86.ext4 -> core-image-minimal-qemux86-20171002031647.rootfs.ext4
lrwxrwxrwx 2 build build      57 Oct  2 05:45 core-image-minimal-qemux86.manifest -> core-image-minimal-qemux86-20171002031647.rootfs.manifest
lrwxrwxrwx 1 build build      55 Oct  2 05:45 core-image-minimal-qemux86.qemuboot.conf -> core-image-minimal-qemux86-20171002031647.qemuboot.conf
lrwxrwxrwx 2 build build      56 Oct  2 05:45 core-image-minimal-qemux86.tar.bz2 -> core-image-minimal-qemux86-20171002031647.rootfs.tar.bz2
-rw-r--r-- 2 build build 4557948 Oct  2 05:44 modules--4.8.24+git0+c84532b647_f6329fd287-r0-qemux86-20171002031647.tgz
lrwxrwxrwx 2 build build      72 Oct  2 05:44 modules-qemux86.tgz -> modules--4.8.24+git0+c84532b647_f6329fd287-r0-qemux86-20171002031647.tgz
{% endhighlight %}

Now that the QEMU image is built it must be extracted from the docker
container. The scp tool could be used, or Docker can copy the files
from a running container. To do this, one needs the session ID of the
container. The Docker prompt in the container looks like the following:

{% highlight shell_session %}
build@0bb147c4a140:/workdir/build$
{% endhighlight %}

where 0bb147c4a140 is the session ID. Using this, copying the QEMU image
is accomplished using:

{% highlight shell_session %}
mac$ docker cp 0bb147c4a140:/workdir/build/tmp/deploy/images/qemux86 .
{% endhighlight %}


### Running the image with QEMU

Finally, the QEMU image is ready to be tested. If QEMU is not installed,
[Homebrew](https://brew.sh/) can install it:

{% highlight shell_session %}
mac$ brew install qemu
{% endhighlight %}

The following then launches QEMU with the image:

{% highlight shell_session %}
$ qemu-system-i386 -kernel bzImage -hda core-image-minimal-qemux86.ext4 -append "console=ttyS0 root=/dev/hda" -nographic
{% endhighlight %}

and a terminal is provided for the newly created GNU/Linux system:

{% highlight shell_session %}
Poky (Yocto Project Reference Distro) 2.2.2 qemux86 /dev/tty1

qemux86 login: root
root@qemux86:~# echo hello world
hello world
{% endhighlight %}

In future posts, I'll build several GNU/Linux images using Yocto.
