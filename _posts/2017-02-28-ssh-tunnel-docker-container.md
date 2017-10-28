---
layout: post
title: Docker SSH Tunnel
tags: [Docker]
description: Sometimes when you are developing locally you might need access to a remote system running a docker container, which has closed all of its ports for security reasons.
---

Sometimes when you are developing locally you might need access to a remote system running a docker container, which has closed all of its ports for security reasons (which is something I recommend strongly as it really reduces the surface area for security breaches and attacks). So how can we get access to the production data in our own local Docker stack?

Often the configuration of an app, consisting of several dockerized services, will be described with a `docker-compose.yml` file and the internal addressing will be with the service names on the docker virtual network. Something like that:

```python

version: '2'
services:
  es:
    image: docker.elastic.co/elasticsearch/elasticsearch:5.2.1
  redis:
    image: redis:3.2
networks:
  default:
    external:
      name: leandot-net-local

```

So any docker container attached to the `leandot-net-local` will be able to talk to the `redis` and `es` services even if they do not expose their ports to the outside world.

```bash
docker run --net=leandot-net-local -it --rm redis:3.2 redis-cli -h redis
```

In order to be able to talk to a production service running on a remote Docker engine, we need to somehow treat it as if it is running in our `leandot-net-local` network. A good way to do that is to run a container, whose only task is to tunnel all traffic sent to itself over to the production system. Let's create such a container, which will tunnel to a production Redis system.

* I assume that you can SSH to the remote system with your SSH key passwordlessly, if not please set it up first.
*  Make sure that we can tunnel traffic through the instance running the remote docker engine. Check that `/etc/ssh/sshd_config` does not contain something like `AllowTcpForwarding no`.
* Find out the internal IP of the redis container on production. Run the following on **production**

```bash
docker inspect YOUR_REDIS_INSTANCE_ID | grep IPAddress
```

and you will get the address, which we will refer to as CONTAINER_IP later on.

```bash
"SecondaryIPAddresses": null,
    "IPAddress": "",
	    "IPAddress": "172.18.0.6",
```

* Now comes a tricky part. If you simply mount your private SSH key in a container, it won't have the correct permissions and SSH will complain. So we need to mount the file and change its permissions in the container. This is something where we can use [bindfs](http://bindfs.org/). The clever folks at [cardcorp](https://github.com/cardcorp/card-rocker/blob/master/r-ssh/Dockerfile) have already solved the issue so let's borrow what we need and quickly explain what is going on. Here is the Dockerfile we are going to use:

```python
FROM debian:stable

## Create a user docker, who is in the staff group
RUN useradd docker \
	&& mkdir /home/docker \
	&& chown docker:docker /home/docker \
	&& addgroup docker staff

## Install openssh, bindfs and kmod
RUN apt-get update \
  && apt-get install -y --no-install-recommends openssh-client bindfs kmod\
  && apt-get clean \
  && rm -rf /var/lib/apt/lists/

## Install bindfs to be able to mount SSH keys from the host with the right permissions
RUN mkdir /home/docker/.ssh \
  && mkdir /home/docker/.ssh-external \
  && echo "bindfs#/home/docker/.ssh-external /home/docker/.ssh fuse force-user=docker,force-group=docker,perms=0700 0 0" >> /etc/fstab
RUN chown -R docker:docker /home/docker/.ssh-external
```


* What is left now is to establish a tunnel to the instance. Let's use a simple `docker-compose.yml` file that uses the above Dockerfile to build an image and start it with the right parameters.

```python

version: '2'
services:
  redis_ssh_tunnel:
    build: .
    entrypoint: ssh
    stdin_open: true
    tty: true
    volumes:
      - ~/.ssh/id_rsa:/home/docker/.ssh-external/id_rsa:ro
    command: "-oStrictHostKeyChecking=no -t -i /home/docker/.ssh-external/id_rsa -L *:6379:CONTAINER_IP:6379 YOUR_USER@YOUR_DOMAIN"
networks:
  default:
    external:
      name: leandot-net-local


```

If all is configured correctly and you start the container **locally** with

```python
docker-compose up
```

you should be able to establish a tunnel to the new system.

* You should be able to connect to the production redis docker container, even though it is completely shielded from the outside world and your connection will be securely encrypted, thanks to the ssh connection. Not bad! If you have an application that normally connects to your local `redis`, you can change the name to `redis_ssh_tunnel` and you will be reading data from production. To avoid actually writing to a production system unintentionally it might make sense to split the connections that read from those that write and only swap the name for the *reading* one.

```bash
docker run --net=leandot-net-local -it --rm redis:3.2 redis-cli -h redis_ssh_tunnel
```

The full source code is available on [leandot-git](https://github.com/ivdelchev/leandot-git/tree/master/redis-ssh-forward).

