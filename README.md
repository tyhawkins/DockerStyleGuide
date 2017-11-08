# Docker Style Guide

This is meant as a non-exhaustive set of tips to writing clean, usable Dockerfiles.

### Before you start

It's assumed you're at least a little familiar with Docker's [best practices](https://docs.docker.com/engine/userguide/eng-image/dockerfile_best-practices/) document, **especially** using [multi-stage builds](https://docs.docker.com/engine/userguide/eng-image/multistage-build/#use-multi-stage-builds).  This is meant not to replace, but augment.  I will reiterate some of these points to point out particularly useful or powerful conventions.

## Minimize your layers as much as possible

Instead of having multiple RUN commands, combine RUN commands into a single layer like so:

    ENV variable="value" \
        othervariable="other value"

    RUN yum install -y \
        wget \
    &&  wget https://example.com/source-package.tar.gz

This is effectively the same as writing `yum install -y wget && wget https://example.com/source-package.tar.gz`, but is much easier to read, especially with an increasing number of commands.  This also applies to layers other than `RUN`, such as `ENV`

## Justify commands and packages, use && convention

These two combined massively increase readability and maintainability of Dockerfiles.  Commands and packages should be justified to the same level of indentation.

    WORKDIR /opt/

    RUN yum install -y \
        build-essentials \
        curl \
        wget \
    &&  wget https://example.com/source-package.tar.gz \
    &&  tar xzf source-package.tar.gz \
    &&  ./configure \
    &&  make \
    &&  make install

While scanning through, it's obvious where new commands start and where continuations of commands (such as listings of packages) are.  Also as part of this, every line (except the last) needs to have a backslash to tell Docker to ignore the incoming newline.  This should also be applied to `ENV` variables.  Also, using `&&` instead of `;` or other conventions kills the `docker build` if any of the commands fail.

## Sort packages alphabetically

Pretty simple, and the benefits should be obvious: you can quickly add/remove dependencies and easily check to see if they're included or not.

    RUN yum install -y \
    autoconf \
    automake \
    binutils \
    bison \
    epel-release \
    flex \
    gcc \
    gcc-c++ \
    gettext \
    git \
    libtool \
    make \
    openssl-devel \
    patch \
    pkgconfig \
    postgresql-devel \
    redhat-rpm-config \
    rpm-build \
    rpm-sign \
    wget \
    zlib-devel \

## Use virtual packages (Alpine only)

This is less Docker-centric but since Docker very often uses Alpine base images, this should apply just about as often.  Prepending the virtual package with a dot (.) also prevents collision with any actual packages, and frees you to use any name you want for the virtual package.

    FROM alpine:latest

    WORKDIR /myapp
    COPY . /myapp

    RUN apk --no-cache add \
        ca-certificates \
        openssl \
        python \
        py-pip \
    &&  apk --no-cache add --virtual .build-dependencies \
        build-base \
        python-dev \
        wget \
    &&  pip install -r requirements.txt \
    &&  python setup.py install \
    &&  apk del .build-dependencies

    CMD ["myapp", "start"]

Of course, many times you can use multi-stage builds so you don't have to remove the packages to begin with...

## Multi-Stage builds

Use them.  For reference, here's what a multi-stage build looks like:

    FROM golang:1.7.3 as builder
    WORKDIR /go/src/github.com/alexellis/href-counter/
    RUN go get -d -v golang.org/x/net/html  
    COPY app.go .
    RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o app .

    FROM alpine:latest
    RUN apk --no-cache add ca-certificates
    WORKDIR /root/
    COPY --from=builder /go/src/github.com/alexellis/href-counter/app .
    CMD ["./app"]

This demonstrates a Go app being built with a specific docker image, then the image is discarded and only the resulting binary is copied into the final image.  This means that the final image size will only be `alpine` plus the `WORKDIR` and the `app`, instead of necessitating basing the image off of `golang:1.7.3`.

## ADD versus COPY

Generally, you should use `COPY` when all you need to do is add files into the container, but it's also important to note `ADD` can extract archives **and** add from remote URLs.  "The best use for ADD is local tar file auto-extraction into the image, as in `ADD rootfs.tar.xz /`" (directly from Docker's best practices).
