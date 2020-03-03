---
layout: post
title:  "Most used docker commands"
author: prabhu
categories: [ Docker ]
image: assets/images/docker.jpg
featured: true
---

The use case of docker is expanding every day and commands used to achieve the task differes with each scenario. Here we will list out the basic and most used docker commands used in our day to day operations.

## Docker repository

* Login to a remote docker repository

```
docker login -u <username> -p <password> <repository url>
```

* Logout of docker repository

```
docker logout <repository url>
```

## Docker images

* List images available in local repository

```
docker images
```

* Remove an image from local repository

```
docker rmi <image name>
```

* Pull an image from docker repository

```
docker pull <image name>
```

* Push a local image to remote repository
```
docker push <image name>
```

* Build an image using Dockerfile

```
docker build -t <image name> --rm .
```

* Load an image from tar archive

```
docker load < busybox.tar.gz
```

* Create a new docker images from a running container

```
docker commit <container name> <image name>
```

* View docker build history of an image

```
docker history --no-trunc <image name>
```

## Docker containers

* List all running containers

```
docker ps
```

* List all containers including stopped ones

```
docker ps -a
```

* Remove a container. -f option will force stop a running container 

```
docker rm -f <container name>
```

* Remove all containers that are not running

```
docker rm $(docker ps -a -q)
```

* Start a container from an image

```
docker run --name <container name> -i -t -d --rm <image name>
```

* Login to a running container in bash prompt

```
docker exec -it <container name> bash
```

* Stop a running container

```
docker stop <container name>
```

* Start a container that is stopped

```
docker start <container name>
```


#### There are many more frequently used commands but again depending on the scenarios. This is just a start to scratch the surface !!! Cheers !!!