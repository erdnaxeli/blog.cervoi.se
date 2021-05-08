---
title: "Build Crystal Docker images for ARM"
date: 2021-05-08T13:28:29+02:00
draft: false
---

For a [recent project](https://github.com/erdnaxeli/castblock) I wanted to build Docker images for Raspberry Pi.
The goal was to ease distribution and adoption.
As the project needs a specific dependency, providing a Docker image is convenient.

# Building a Crystal Docker image

The task is pretty simple.
The Crystal team provides [Docker images](https://hub.docker.com/r/crystallang/crystal) ready to use, so it is very easy:

```Dockerfile
FROM crystallang/crystal:1.0.0-alpine
COPY lib /src/lib
COPY src /src/src
COPY shard.yml shard.lock /src/
WORKDIR /src
RUN shards build --release --static

FROM alpine
RUN apk add tini
COPY --from=0 /src/bin/castblock /usr/bin/castblock
ENTRYPOINT ["/sbin/tini", "--", "/usr/bin/castblock"]
```

Note that this Dockerfile is just an example and you should adapt it for your project, especially the executable name.

First, we are using a multistage Dockerfile.
The reason is that we don't want the final image to contains all the dependencies needed only for the compilation step.
So in a first step we use the official Crystal image to build the project, then we copy the result in another image where compilations dependencies (the Crystal compiler and dev versions of some libraries) are not installed.

Secondly, we are using Alpine based images.
The only reason here is to reduce the size of the final image.

And thirdly, we are using [Tini](https://github.com/krallin/tini).
This is related to how processes are run inside a container, you should read [the author explanation](https://github.com/krallin/tini/issues/8) if you want to know more about why it is useful.

So far so good, we have a Docker image.
But for which architecture?
I never though about it before now, because I was only running Docker containers on x86-64 servers, but now that I want to build Docker images for Raspberry Pi I need to care about it.
And the answer is quite obvious: as I am building this Docker image on a x86-64 computer, the resulting image is for x86-64 CPUs.
In fact if we run `docker inspect` on the produced image we can see `"Architecture": "amd64"` (amd64 is another name for x86-64).

# Building a Crystal Docker image for ARM

## Using cross-compilation

Raspberry Pi use ARM CPUs.
The exact version varies accross Raspberry Pi models.

Knowing that, we can try to use cross compilation.
Cross compilation is a way to produce a binary for a different platform than the one we are using to build the binary.
Crystal (through LLVM) supports that in an easy way.

So let's try it:

```Dockerfile
FROM crystallang/crystal:1.0.0-alpine
COPY lib /src/lib
COPY src /src/src
COPY shard.yml shard.lock /src/
WORKDIR /src
RUN crystal build --release --cross-compile --target armv6k-unknown-linux-gnueabihf src/castblock.cr
```

By running this on a x86-64 computer, we are still using the x86-64 Crystal image.
But by specifying `--cross-compile --target armv6k-unknown-linux-gnueabihf` we tell Crystal to build a binary for an ARM CPU instead of the x86-64 one.

The target `armv6k-unknown-linux-gnueabihf` is called a triple[^triple].
It is read like this:
* armv6k: the targeted CPU architecture.
  Here armv6k is the one Raspberry Pi OS is built for, and is compatible with all existing Raspberry Pi CPUs.
* unknown: the targeted vendor (pc, apple, nvidia, …).
  Here we don't specify anything.
* linux: the targeted operating system.
* gnueabihf: the targeted EABI (Embedded Application Binary Interface), which specify conventions for the binary.
  The same OS (here Linux) can use different EABIs.
  Here we are using the GNU EABI, with the hard float (hf) specification[^hard float].

A little note here.
We are using a GNU EABI, which basically means that we want to be compatible with the GNU LibC.
But doesn't Alpine use the musl libc?
Yes it is, our binary will not be able to run on an Alpine distribution.
The reason we use a GNU EABI is that [the musleabihf target is not supported yet](https://github.com/crystal-lang/crystal/issues/5467) by Crystal.

## The linking step

We are not done yet, that would be too easy.
Actually the build command with `--cross-compile` does not produce an executable that we can run.
Instead it builds an object file that need to be linked to shared libraries.
As we are building our binary in an x86-64 environment, only libraries built for x86-64 platform are present.

We need to link our binary in an ARM environment. But how to get it? Docker helps use here: we can provides a `--platform` parameter to the `FROM` instruction to specify the platform for which we want the image:

```Dockerfile
FROM crystallang/crystal:1.0.0-alpine
COPY lib /src/lib
COPY src /src/src
COPY shard.yml shard.lock /src/
WORKDIR /src
RUN crystal build --release --cross-compile --target armv6k-unknown-linux-gnueabihf src/castblock.cr

FROM --platform=linux/arm/v7 debian:buster-slim
WORKDIR /src
RUN apt-get update
RUN apt-get install -y \
  g++ \
  git \
  libpcre3-dev \
  libevent-dev \
  libgc-dev \
  libssl-dev \
  libxml2-dev \
  llvm \
  make \
  zlib1g-dev
RUN git clone --depth 1 --branch 1.0.0 https://github.com/crystal-lang/crystal.git .
RUN make libcrystal
COPY --from=0 /src/castblock.o .
RUN cc castblock.o -o castblock  -rdynamic -L/usr/bin/../lib/crystal/lib -lz `command -v pkg-config > /dev/null && pkg-config --libs --silence-errors libssl || printf %s '-lssl -lcrypto'` `command -v pkg-config > /dev/null && pkg-config --libs --silence-errors libcrypto || printf %s '-lcrypto'` -lpcre -lm -lgc -lpthread src/ext/libcrystal.a -levent -lrt -ldl
```

We added a new stage in our multistage Dockerfile.
As we compiled our project for a GNU ABI we use Debian as a base.
Then we need to install some dependencies we will link against.
We also need to build a specific shared library for Crystal[^libcrystal].
Then we can run the linker on our object file to produce an executable binary.

## QEMU

But if we try to build this, we will got an error like this one:
```
standard_init_linux.go:219: exec user process caused: exec format error
```

What it is trying to tell us is that we are running an executable (`cc`) built for an ARM platform on a x86-64 platform.
This can't work.

Are we doomed?
Not yet.
We can use emulation to run a program built for a different CPU.

The project that will help us is [QEMU](https://www.qemu.org/).
QEMU is an emulator and virtualization project that supports a wide range of CPUs.
The second thing that we need is called bindfmt\_misc.
Bindfmt\_misc is a component of the Linux kernel that allows us to register an interpreter for executables.
So what we have to do is to use bindfmt\_misc to tell Linux to run executables built for ARM using QEMU.

To do that in a quick and simple way, we will use the awesome project [qus](https://github.com/dbhi/qus).

```console
$ docker run --rm --privileged aptman/qus -s -- -p arm
cat ./qemu-binfmt-conf.sh | sh -s -- --path=/qus/bin -p --suffix -static
Setting /qus/bin/qemu-arm-static as binfmt interpreter for arm
```

Now we can built our image, and `cc` will actually be run by `qemu-arm-static`.

## The final Dockerfile

Just as we did for the very first Docker image we built, we need a last stage where we copy the final binary.

```Dockerfile
FROM crystallang/crystal:1.0.0-alpine
COPY lib /src/lib
COPY src /src/src
COPY shard.yml shard.lock /src/
WORKDIR /src
RUN crystal build --release --cross-compile --target armv6k-unknown-linux-gnueabihf src/castblock.cr

FROM --platform=linux/arm/v7 debian:buster-slim
WORKDIR /src
RUN apt-get update
RUN apt-get install -y \
  g++ \
  git \
  libpcre3-dev \
  libevent-dev \
  libgc-dev \
  libssl-dev \
  libxml2-dev \
  llvm \
  make \
  zlib1g-dev
RUN git clone --depth 1 --branch 1.0.0 https://github.com/crystal-lang/crystal.git .
RUN make libcrystal
COPY --from=0 /src/castblock.o .
RUN cc castblock.o -o castblock  -rdynamic -L/usr/bin/../lib/crystal/lib -lz `command -v pkg-config > /dev/null && pkg-config --libs --silence-errors libssl || printf %s '-lssl -lcrypto'` `command -v pkg-config > /dev/null && pkg-config --libs --silence-errors libcrypto || printf %s '-lcrypto'` -lpcre -lm -lgc -lpthread src/ext/libcrystal.a -levent -lrt -ldl

FROM --platform=linux/arm/v7 debian:buster-slim
RUN apt-get update && apt-get install -y \
        ca-certificates \
        libgc1c2 libevent-2.1-6 \
        libssl1.1 \
        tini \
    && c_rehash  # see https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=923479
COPY --from=1 /src/castblock /usr/bin/castblock
ENTRYPOINT ["/usr/bin/tini", "--", "/usr/bin/castblock"
```

In this last stage we only install the runtimes dependencies needed, not the compile time ones.
If you want to read about an awful bug that almost makes me crazy, you can check the Debian bugtracker link ;)

We now have a Docker image that can run on any Rapsberry Pi, and we built it all on our x86-64 computer!
We can verify that using `docker inspect`: ` "Architecture": "arm"`.

# Building a Crystal Docker image for AArch64

But does it actually run on all Rapsberry Pi?

Our image actually run on armv6 compatible platforms, and armv6 is a 32 bits architecture.
But recent Raspberry Pi models have actually a 64 bits CPU.
This ARM 64 bits architecture is called AArch64.

The stable Raspberry Pi OS image is built for a 32 bits architecture and can perfectly run on all Raspberry Pi, but a beta version is availble targeting 64 bits architecture, and others 64 bits distributions are available too.
We want to support AArch64 platform!

We could use the same steps as previously, cross-compilation, linking, and final image, changing all platform references to specify the AArch64 platform.
But this force us to use a Debian base, which is almost 20Mo heavier that Alpine.
Fortunately, a [Crystal package](https://pkgs.alpinelinux.org/package/edge/community/aarch64/crystal) is available for Alpine AArch64!

So actually our Dockerfile will look more like the one for x86-64:

```Dockerfile
FROM --platform=linux/arm64/v8 alpine:edge AS crystal
RUN echo '@edge http://dl-cdn.alpinelinux.org/alpine/edge/community' >>/etc/apk/repositories
RUN apk add --update --no-cache --force-overwrite \
  crystal@edge \
  g++ \
  gc-dev \
  libxml2-dev \
  llvm10-dev \
  llvm10-static \
  make \
  musl-dev \
  openssl-dev \
  openssl-libs-static \
  pcre-dev \
  shards@edge \
  yaml-dev \
  yaml-static \
  zlib-dev \
  zlib-static
COPY lib /src/lib
COPY src /src/src
COPY shard.yml shard.lock /src/
WORKDIR /src
RUN shards build --static --release

FROM --platform=linux/arm64/v8 alpine:latest
RUN apk add tini
COPY --from=crystal /src/bin/castblock /usr/bin/castblock
ENTRYPOINT ["/sbin/tini", "--", "/usr/bin/castblock"]
```

There is not (yet?) an official Crystal image for AArch64, that's why we use Alpine directly.
Moreover the package is only available in Alpine Edge, in the community repository.

Before building this image we need to setup qus to be able to run AArch64 executables:

```console
$ docker run --rm --privileged aptman/qus -s -- -p aarch64
cat ./qemu-binfmt-conf.sh | sh -s -- --path=/qus/bin -p aarch64 --suffix -static
Setting /qus/bin/qemu-aarch64-static as binfmt interpreter for aarch64
```

The built is a bit slower than with cross-compilation, as now the Crystal compiler runs under QEMU emulation, but we trade that for a smaller final Docker image.

`docker inspect` confirms the platform: `"Architecture": "arm64"`.

# Conclusion

We saw how we can leverage cross-compilation and QEMU to build Docker images for x84-64, ARM and AArch64.
They can run on any Raspberry Pi, either with a 32 or 64 bits OS.
The images should also run on other ARM platforms.

With those methods you can support many platforms, without needing to actually having them to build the images.
The images can be built on any x86-64 platform, using for example your favorite CI/CD tool.

[^triple]: Although actually it contains 4 values `¯\_(ツ)_/¯`

[^hard float]: Hard float means that float computation is done by the CPU itself, and it is opposed to soft float wich means that you have to write code to do the float computation by manipulating only integer, which is way slower.

[^libcrystal]: This [was removed](https://github.com/crystal-lang/distribution-scripts/pull/87) and will not be needed in the next Crystal version.
