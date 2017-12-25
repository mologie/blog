---
layout: post
title:  "Cross Compiling C++ with CMake/LLVM"
date:   2017-12-25 16:25:01 +0100
categories: programming
---

# Forword

The purpose of this guide is to give an overview of setting up a cross compilation environment at
the example of a Raspberry Pi 1 (armv6hf) or 2/3 (armv7) running Alpine Linux[^1]. The procedures
shown here apply to a range of devices and are not limited to the Pi or ARM devices.

Going through this guide will take you between 30 minutes and a few hours.

Motivation for this guide are recent developments in the LLVM project, specifically the addition of
`lld` in the stable distribution of LLVM 5.0. With `lld`, we can skip setting up GCC and GNU
binutils entirely, which is widely known as horrible pain and causing lots of suffering.[^2]

The following tools are used in this guide; please check if they work for your purpose first:

* Alpine Linux 3.7 as distro on the Raspberry Pi, though you can adapt this guide to any other
  distribution with a bit of effort.
* Linux or macOS on x86-64 as host system where the compiling happens
* Docker for building an Alpine Linux sysroot
* C/C++ as programming language, with C++17 being the most recent, supported version
* Clang and LLVM 5.0 and as compiler frontend and backend
* CMake as meta build system; it'll generate makesfiles capable of cross-compilation

The target audience for this guide are people familiar with the command line and CMake, who have
developed C/C++ software before and wish to extend their knowledge by learning how to use the former
for cross-compiling with LLVM.

# Prerequisites

**A Raspberry Pi running Alpine Linux 3.7**

[Installation instructions.](https://wiki.alpinelinux.org/wiki/Raspberry_Pi)
Note that on embedded systems, Alpine Linux mounts a tmpfs overlay over `/` and extracts all of its
packages there. Thus, you must not have more packages installed than what fits in the available
RAM minus what is required by running applications.[^3]

You may also use any other Linux distribution that uses libgcc[^4] (not to be confused with glibc,)
including Raspbian and Debian. The only things that change when using a different distro are the
sysroot's contents and thus also the libstdc++/GCC version, which are reflected as `GCC_BASEVER` in
the CMake toolchain file presented later in thie guide.

For clarity, a sysroot is a subset of the files from the root directory of the target machine, which
are required for compiling applications for it. Usually, it's a thinned out version of `/lib` and
`/usr`, though a full copy of both from a readily set-up destination system works fine, too.

**For macOS: A package manager**

You're using [Homebrew](https://brew.sh/) and have it installed in the default location
`/usr/local`, right?

**LLVM 5.0**

On Linux, get LLVM 5.0 from your package manager. Debian/Ubuntu users can use the first-party LLVM
distribution from [apt.llvm.org](https://apt.llvm.org/).

On macOS, install LLVM 5.0 via Homebrew: `brew install llvm@5`

LLVM 5.0 was the latest stable version at the time of writing this guide and supports C++17. More
recent versions probably work fine, but require you to adjust paths including version numbers in
the scripts that are published in later sections.

**Docker**

We'll use Docker for building a sysroot. You may skip installing Docker and use a real system for
collecting your sysroot instead, but for the Raspberry Pi running Alpine, using Docker appears to be
the most trivial solution. You may also use Qemu or other more complex software products, but these
are out this guide's scope.

[Download Docker from docker.com](https://www.docker.com/community-edition#/download). The official
website also contains instructions for installing Docker on macOS and various Linux distributions.

**CMake**

This guide was tested with CMake 3.10. Install CMake from your package manager. MacOS users may run
`brew install cmake` to get the latest version.

# Toolchain Setup

You're now sitting in front of a Linux or macOS machine with LLVM 5.0 and CMake 3.10+ installed,
have Docker available and have your Raspberry Pi 1/2/3 running Alpine Linux in your local network.
Great! Let's begin the incantation:

**Building an Alpine Linux Sysroot**

Follow this log from a shell on your computer, where Alpine x86-64 will be started in Docker and
used for installing a second Alpine Linux system inside of the container. The nested installation
becomes the sysroot for our cross-compilation toolchain.

{% highlight shell %}
# Pick a location to store your sysroot (and later CMake toolchain files) at.
$ SYSROOT=$HOME/Toolchains/sysroots/alpine-3.7-armhf
$ mkdir -p "$SYSROOT"

# Create a persistent Docker conatiner for running Alpine 3.7. This allows you
# to return to it later and pull more packages from the Alpine repos. We also
# grant the container write access to $SYSROOT.
$ sudo docker run -it -v $SYSROOT:/sysroot --name alpine_stager alpine:3.7 sh
{% endhighlight %}

Docker downloads Alpine 3.7 and drops you into a new shell. In that shell, the directory `/sysroot`
is mapped to the directory at `$SYSROOT` in the previous shell.

Note that Docker automatically persists this container under the name we gave it (`alpine_stager`.)
When you don't specify a name, Docker makes one up (see `docker ps -a`.) Use `--rm` to start a
non-persistent container, or purge the container later with `docker rm <name>`.

{% highlight shell %}
# These variables are later passed to apk, Alpine's package manager
# Pick a mirror near you: http://rsync.alpinelinux.org/alpine/MIRRORS.txt
/ $ BRANCH=v3.7/main
/ $ MIRROR="http://ftp.halifax.rwth-aachen.de/alpine/$BRANCH"
/ $ SYSROOT=/sysroot
/ $ ARCH=armhf

# Download the initial root file system for your new sysroot.
/ $ apk -X $MIRROR -U --allow-untrusted --root $SYSROOT --initdb --arch $ARCH \
      add alpine-base
# Docker on macOS prevents writing suid binaries to volumes, resulting in:
#   .../busybox-1.27.2-r7.trigger: line 20: /bin/bbsuid: Permission denied
# The warning may be safely ignored, because we won't ever boot the sysroot.

# Extend this list by the packages you need. Many programs need 'linux-headers'
# and all of them need 'alpine-sdk' when compiling. Search for packages with:
# apk -X $MIRROR -U --allow-untrusted --root $SYSROOT search terms here
/ $ apk -X $MIRROR --root $SYSROOT --arch $ARCH \
      add alpine-sdk linux-headers YOUR ADDITIONAL PACKAGES HERE

# Your sysroot is now set up. Quit Docker:
/ $ exit
{% endhighlight %}

**Installing Additional Packages**

If you need to install additional packages from Alpine's repositories later, reenter the container
and run `apk` again like so:

{% highlight shell %}
# Attach to the container's shell:
$ docker start -ia alpine_stager

# Repeat the environment setup from above:
/ $ BRANCH=...
/ $ MIRROR=...
/ $ SYSROOT=/sysroot
/ $ ARCH=...

# Install packages:
/ $ apk -X $MIRROR --root $SYSROOT --arch $ARCH add name-of-new-package a-b-c
/ $ exit
{% endhighlight %}

**About Permissions**

You'll find that files in `$SYSROOT` are owned by root, because you were root in the container. I
recommend to adjust the owner and permissions using `chown -R you:yourgroup $SYSROOT` and `chmod -R
u+rw $SYSROOT`, so that cross-compiled libraries in later sections can be installed into `$SYSROOT`
without elevating to the real root on your system.

Rinse and repeat whenever more packages are added through `apk` from within the container.

# CMake Toolchain File

We've got all ingredients for cross-compilation ready: LLVM, which does the compiling, and a
sysroot, which tells LLVM how your target system looks like. This section is about telling LLVM to
actually do the cross-compiling with your sysroot with the least amount of effort possible.

For this guide, I've opted to use CMake as meta build system. Despite of all its kinks and quirks,
its broad feature set and platform support make it the smallest evil in practice. **No changes to
your existing CMake files are required for adding cross-compilation support.**

CMake handles cross-compilation through *toolchain files*, which are just plain CMake scripts that
are run during CMake's internal setup and validation phase.

Save the following file to `~/Toolchains/armv6-alpine-linux-musleabihf-llvm.cmake`:  
<script src="https://gist.github.com/mologie/4425c16477e2218af83078c4f8708867.js"></script>

([Link for browsers without JS enabled.](https://gist.github.com/mologie/4425c16477e2218af83078c4f8708867))
([Raw file link.](https://gist.githubusercontent.com/mologie/4425c16477e2218af83078c4f8708867/raw/armv6-alpine-linux-musleabihf-llvm.cmake))

**The above toolchain file configures CMake for compiling for the Raspberry Pi Zero and 1.**
For the Raspberry Pi 2/3, change the relevant lines of the target platform configuration to:

{% highlight cmake %}
SET(CMAKE_SYSTEM_PROCESSOR arm) # uname -p
SET(CROSS_TARGET arm-alpine-linux-musleabihf)
SET(CROSS_MACHINE_FLAGS "-marm -march=armv7 -mfloat-abi=hard -mfpu=vfp")
{% endhighlight %}

Rename the file appropriately, for example to `armv7-alpine-linux-musleabihf-llvm.cmake`.

# Installing Additional Libraries

Just like software can be compiled from source and installed to `/usr/local`, you may cross-compile
software and install it into your sysroot. The following shows an example, where yaml-cpp is
compiled as static library:

{% highlight shell %}
$ git clone https://github.com/jbeder/yaml-cpp.git
$ cd yaml-cpp
$ mkdir -p build/alpine-3.7-armv6-hf; cd build/alpine-3.7-armv6-hf
$ cmake \
    -D CMAKE_TOOLCHAIN_FILE=$HOME/Toolchains/armv6-linux-musleabihf-llvm.cmake \
    -D YAML_CPP_BUILD_TESTS=OFF \
    -D YAML_CPP_BUILD_TOOLS=OFF \
    -D YAML_CPP_BUILD_CONTRIB=OFF \
    ../../
$ make
$ SYSROOT=$HOME/Toolchains/sysroots/alpine-3.7-armhf
$ make DESTDIR=$SYSROOT install
{% endhighlight %}

Note: The DESTDIR argument is not supported by all Makefiles. CMake happens to support it.

There's no magic formula for cross-compiling things. If a project was not designed for
cross-compiling and does Complex Stuff™, the build might fail. If it indeed does fail:

* Check, whether the build process wants to run tests automatically. This won't work, because your
  machine can't run code that's not compiled for it. (You may set up qemu-binfmt on Linux to
  automagically make these run through, but that's out of this guide's scope.) Solution: Patch
  the build files to not depend on generated executables.
* Header files and libraries might be missing from your sysroot.
  Install them manually or with `apk`.

Not all projects use CMake. That would be horrible, because CMake is horrible (although it works
somewhat, just like most governments.) You'll find many Automake-based projects, which are not
covered in this guide.

Configuring Automake for cross-compilation boils down to setting a whole lot of environment
variables to point Automake to your cross-compilers; also, set `--target` to the target system's
[triplet](http://wiki.osdev.org/Target_Triplet).

# Toolchain Usage Example

Did compiling yaml-cpp during the previous section work? 🎉

Let's check the rest of the toolchain for proper functionality by writing a small demo program and
linking it against the static archive of yaml-cpp that is availble in your sysroot. Finally, the
program is copied to the Raspberry Pi via SSH and started.

It's also a good exercise to include some floating point calculations to check if LLVM correctly
compiles for the target device's FPU.

**example.cpp**
{% highlight cpp %}
#include <yaml-cpp/yaml.h>
#include <iostream>
static double __attribute__((noinline)) two() {
  return 2.0;
}
int main() {
  // via https://github.com/jbeder/yaml-cpp/wiki/How-To-Emit-YAML
  YAML::Emitter out;
  out << "Hello, World!";
  std::cout << "Here's the output YAML:\n" << out.c_str() << "\n";
  std::cout << "Some floating point math: " << 3.0/two() << "\n";
  return 0;
}
{% endhighlight %}

**CMakeLists.txt**
{% highlight cmake %}
CMAKE_MINIMUM_REQUIRED(VERSION 3.10.0 FATAL_ERROR)
SET(CMAKE_CXX_STANDARD 17) # yaml-cpp requires at least C++11; C++17 is current
SET(CMAKE_CXX_STANDARD_REQUIRED ON)
SET(CMAKE_CXX_EXTENSIONS OFF)
PROJECT(example-project CXX)
FIND_PACKAGE(yaml-cpp)
INCLUDE_DIRECTORIES(${YAML_CPP_INCLUDE_DIR})
ADD_EXECUTABLE(example example.cpp)
TARGET_LINK_LIBRARIES(example ${YAML_CPP_LIBRARIES})
{% endhighlight %}

On your machine with the cross-toolchain installed in `~/Toolchains`:

{% highlight shell %}
$ mkdir build; cd build
$ TOOLCHAINF_FILE=$HOME/Toolchains/armv6-alpine-linux-musleabihf-llvm.cmake
$ cmake -D CMAKE_TOOLCHAIN_FILE=$TOOLCHAINF_FILE ..
$ make
-- Configuring done
-- Generating done
-- Build files have been written to: ...snip.../build
Scanning dependencies of target example
[ 50%] Building CXX object CMakeFiles/example.dir/example.cpp.o
[100%] Linking CXX executable example
[100%] Built target example
$ scp example root@pi:/tmp
{% endhighlight %}

Run the example program on the Raspberry Pi. Note that Alpine does not ship with `libstdc++` in its
base image. Thus, if your Pi contains a blank-ish Alpine system, run `apk add libstdc++` and commit
the changes to persistent storage via `lbu ci -d` first.

```
pi ~ # /tmp/example
Here's the output YAML:
Hello, World!
Some floating point math: 1.5
```

And that concludes this guide on cross-compiling. Fair winds!

# Meta

This guide was created as part of my home automation system's documentation, where I use the
Raspberry Pi 1 as development board.

If you've found an error, please send me an email at the contact address in the footer. For
questions create a Stack Overflow post, link to this guide, and provide *exact steps on how to
reproduce the error*, which can be followed even if one did not have this guide. Then, send me a
link to the post via email.

# Footnotes

[^1]: Alpine Linux is my distribution of choice for embedded systems, because it can run from RAM and is thus not affected by power failures. Raspbian/Debian are designed for a read-write root filesystem, which is OK for desktop systems and servers, but a questionable choice for systems, which are supposed to work reliably in an uncontrolled environment. The disadvantage of Alpine is that data must be explicitly committed to persistent storage.

[^2]: LLVM, in contrast to GCC, has cross-compilation capabilties built in and thus avoids the headache of compiling a compiler that then compiles an incomplete cross-compiler for compiling support files, with which one can finally build the real cross compiler. Eek. Granted, LLVM from scratch is not for the faint of heart either. However, for the purpose of this guide, we assume that LLVM is easily installable on your system. After all, one LLVM installation covers all target devices.

[^3]: This especially implies that you'll have trouble installing `alpine-sdk` on the Raspberry Pi itself. We'll use Docker for building the sysroot instead because of this intentional design limitation, which is reasonable for embedded systems.

[^4]: <https://gcc.gnu.org/onlinedocs/gccint/Libgcc.html>
