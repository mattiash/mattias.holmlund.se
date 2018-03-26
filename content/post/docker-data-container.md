---
title: "Docker Data Container"
date: 2015-08-23T21:17:40+02:00
categories:
- category
- subcategory
tags:
- docker
keywords:
- tech
#thumbnailImage: //example.com/image.jpg
---

<!--more-->

# Building the perfect data container

{{<alert info>}}
2018-03-25: This article was originally published on [Medium](https://medium.com/@mattias.holmlund/building-the-perfect-data-container-2df6bb3528df). I now regard this method obsolete and recommend using [Docker named volumes](https://docs.docker.com/storage/volumes/) for storing data.
{{</alert>}}

The [recommended](https://docs.docker.com/userguide/dockervolumes/) solution for storing persistent data in docker today seems to be data containers. [The idea](https://medium.com/@ramangupta/why-docker-data-containers-are-good-589b3c6c749e) is that you have one container where you run your application (the application container), and a second container where you store the data for the application (the data container). You start by creating the data container and make sure that it exposes volumes where the data shall be stored, and then you start the application container with `--volumes-from <data-container>` to allow the application container to store all its data in the data container.

The question for me was then: How can we build a good data container? Since the data container shall only store the data for the application, we should try to make the data container as small and simple as possible, both to decrease its size on disk, but more importantly, to minimize the amount of stuff that can go wrong and requires maintenance. Reading up on data containers, I found two different recommended methods for building data containers.

The first method, recommended in the [docker documentation](https://docs.docker.com/userguide/dockervolumes/), was to build the data container from the same container that the application uses:

> Let’s create a new named container with a volume to share. While this container doesn’t run an application, it reuses the training/postgres image so that all containers are using layers in common, saving disk space.

This minimizes the size of the data container on disk when it is created, but what happens after a while? The purpose of the data container is to store the data for a long time. After a while, we might want to replace the application with another container, containing a new version of the application, while still using the same data-container to preserve our precious data. Then suddenly, the data container and the application container are not derived from the same base container any longer. After a few years, your data container might be based on an old container, with known security problems. It is not a big problem since it doesn’t actually run anything, but it doesn’t feel right.

Another popular method is to build the data container from the smallest possible image. I’ve seen [data containers built from busybox](https://medium.com/@ramangupta/why-docker-data-containers-are-good-589b3c6c749e), as well as data containers built with [a single statically linked go-binary](http://blog.xebia.com/2014/07/04/create-the-smallest-possible-docker-container/). This produces fairly small containers, but we are still including binaries we have no intention of using.

## Building an empty container
Why not build an empty container instead? The following Dockerfile builds an empty container

```dockerfile
FROM scratch
MAINTAINER Mattias Holmlund <mattias@holmlund.se>

VOLUME /data
ENTRYPOINT ["/no/such/file"]
```

The container is created from the special scratch container, which is an absolutely empty container.

It seems that the VOLUME command automatically creates a /data directory. The ENTRYPOINT command must be present, otherwise docker refuses to build the container, but it can point to anything.

This container contains nothing but an empty directory. No technical debt or wasted disk-space in here.

Notice that this container cannot be run since the ENTRYPOINT is invalid, but docker allows us to create a container without running it.

To test the container, use the following commands:

```shell
$ docker create --name my-data mattiash/data
76c0452a09f201da31a0bc56d71c367faf1dd14ed1a0809a85442383d6cf5152
$ docker run --volumes-from my-data -i -t --rm ubuntu /bin/bash
root@78694d60717d:/# echo hello > /data/test.txt
root@78694d60717d:/# exit
exit
$ docker run --volumes-from my-data -i -t --rm ubuntu /bin/bash
root@6ca4b9e9edea:/# cat /data/test.txt 
hello
root@6ca4b9e9edea:/# exit
exit

```

First, we create the data container and give it the name my-data. Then, we run an application container from ubuntu and link it to my-data with --volumes-from. Inside this container (line 4), we write a file under /data and then exit. When we exit, the application container is removed since we specified --rm when we created it. We then start a new application container linked to the same data-container and print out the data from the data container.

This minimalistic data container is available on docker hub as mattiash/data and the source is on github.