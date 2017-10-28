---
layout: post
title: Indie Developer Guide to Production, Part 1 - Configure Remote Python Interpreter to use Docker in PyCharm IDE
tags: [Docker]
description: Introducing Docker and PyCharm IDE and laying the grounds for a reasonable IDE setup for Python development.
---

This post will introduce [Docker](https://www.docker.com/) and [PyCharm IDE](https://www.jetbrains.com/pycharm/) and lay the grounds for a reasonable IDE setup for Python development, that keeps your development environment sane and allows you to focus on the actual business logic in the future.

## Docker

All our services will be built and deployed with Docker. If you have not heard about Docker, then it is a very good time you did, because it's taking over the development and sysadmin worlds and it has a good reason to. In the dark pre-container pre-Docker times you had to setup your environment locally, polluting your machine with numerous languages, packages, libraries, often conflicting and breaking older projects if upgraded. Provisioning and deployment was a hassle, you had to write your own scripts in Chef, Puppet, Ansible, which is tangetial to what we want to achieve here - the ability to iterate and release fast as an indie developer. Even if you had more manpower you wouldn't want to waste it in writing deployment scripts, would you?

You can mix tech stacks in a clean and defined way in different containers, exposing only their interfaces and essentially build a service-oriented architecture without much hassle. For example, imagine you normally write your code in Python and you need a language-detection library for your project. However, the best state-of-the-art library is only available in Java...what do you do?

1. rewrite it completely in Python spending sleepless weeks
2. take the crappy existing Python version and hope nobody notices that you detect Korean as Chinese
3. happily pack the Java version in a Docker image, exposing only what you need via a simple HTTP API and move on with building your real product.

But enough theory, let's get our hands dirty now. I am going to show you my Docker setup on Mac, if you are using a different OS the process should be similar. Recently Docker released a native solution for Mac OS and Windows, which is finally supported in PyCharm \(our IDE of choice\). So we will install the native Docker for Mac. Jump to the [Install Docker](https://docs.docker.com/engine/installation/#supported-platforms) page and follow the instructions, until you are able to do the following \(versions can and will change\):

```
$ docker --version
Docker version 17.03.0-ce, build 60ccb22

$ docker-compose --version
docker-compose version 1.11.2, build dfed245

$ docker-machine --version
docker-machine version 0.10.0, build 76ed2a6
```

Before we proceed, please spend a couple of minutes to check the official [Docker overview](https://docs.docker.com/engine/docker-overview/#docker-engine). It will introduce you to important terminology and concepts.

Back to the three tools from above. Those are vital to how we are going to work with Docker in the future. Let's spend a bit of time on each one.

### Docker CLI

```
➜  pycharm git:(initial_setup) ✗ docker --help

Usage:    docker COMMAND

A self-sufficient runtime for containers

Options:
      --config string      Location of client config files (default "/Users/ivdelchev/.docker")
  -D, --debug              Enable debug mode
      --help               Print usage
  -H, --host list          Daemon socket(s) to connect to (default [])
  -l, --log-level string   Set the logging level ("debug", "info", "warn", "error", "fatal") (default "info")
      --tls                Use TLS; implied by --tlsverify
      --tlscacert string   Trust certs signed only by this CA (default "/Users/ivdelchev/.docker/ca.pem")
      --tlscert string     Path to TLS certificate file (default "/Users/ivdelchev/.docker/cert.pem")
      --tlskey string      Path to TLS key file (default "/Users/ivdelchev/.docker/key.pem")
      --tlsverify          Use TLS and verify the remote
  -v, --version            Print version information and quit

Management Commands:
  checkpoint  Manage checkpoints
  container   Manage containers
  image       Manage images
  network     Manage networks
  node        Manage Swarm nodes
  plugin      Manage plugins
  secret      Manage Docker secrets
  service     Manage services
  stack       Manage Docker stacks
  swarm       Manage Swarm
  system      Manage Docker
  volume      Manage volumes

Commands:
  attach      Attach to a running container
  build       Build an image from a Dockerfile
  commit      Create a new image from a container's changes
  cp          Copy files/folders between a container and the local filesystem
```

The `docker` command can manage containers - build, run, stop, start, delete and much more. Normally it communicates with your local Docker Engine via a command line interface \(CLI\) and all commands are executed locally. We will see how this can be changed with `docker-machine` but for now this is what you need to remember.

Docker gives you the possibility to organize and self-document your projects. It can build images automatically by reading the instructions from a Dockerfile. So if in a month time you forgot how you did something and didn't document it, relax, just check out the Dockerfile sitting in your project folder and it will all be there. The instructions are very readable and [well documented](https://docs.docker.com/engine/reference/builder/).

### Docker Compose

```
➜  pycharm git:(initial_setup) ✗ docker-compose --help
Define and run multi-container applications with Docker.

Usage:
  docker-compose [-f <arg>...] [options] [COMMAND] [ARGS...]
  docker-compose -h|--help

Options:
  -f, --file FILE             Specify an alternate compose file (default: docker-compose.yml)
  -p, --project-name NAME     Specify an alternate project name (default: directory name)
  --verbose                   Show more output
  -v, --version               Print version and exit
  -H, --host HOST             Daemon socket to connect to

  --tls                       Use TLS; implied by --tlsverify
  --tlscacert CA_PATH         Trust certs signed only by this CA
  --tlscert CLIENT_CERT_PATH  Path to TLS certificate file
  --tlskey TLS_KEY_PATH       Path to TLS key file
  --tlsverify                 Use TLS and verify the remote
  --skip-hostname-check       Don't check the daemon's hostname against the name specified
                              in the client certificate (for example if your docker host
                              is an IP address)

Commands:
  build              Build or rebuild services
  bundle             Generate a Docker bundle from the Compose file
  config             Validate and view the compose file
  create             Create services
  down               Stop and remove containers, networks, images, and volumes
  events             Receive real time events from containers
```

Docker acquired a project called Fig, later renamed to Docker Compose, which finally got merged in the native Docker environment, which describes how your applications should be run. It is very important to have this mental model - the Dockerfile describes how to **build** your Docker Image and Docker Compose files describe how your Docker image will be **run**. Typical configuration is that Docker Compose is also used to build your images by pointing to a Dockerfile. Docker Compose files \(docker-compose.yml\) also describe how your services are named \(so that they can be discovered\), what environment variables are set \(e.g. to distinguish local and production\), what ports are open, what networks they attach to and so on.

### Docker Machine

```
➜  pycharm git:(initial_setup) ✗ docker-machine --help
Usage: docker-machine [OPTIONS] COMMAND [arg...]

Create and manage machines running Docker.

Version: 0.10.0, build 76ed2a6

Author:
  Docker Machine Contributors - <https://github.com/docker/machine>

Options:
  --debug, -D                        Enable debug mode
  --storage-path, -s "/Users/ivdelchev/.docker/machine"    Configures storage path [$MACHINE_STORAGE_PATH]
  --tls-ca-cert                     CA to verify remotes against [$MACHINE_TLS_CA_CERT]
  --tls-ca-key                         Private key to generate certificates [$MACHINE_TLS_CA_KEY]
  --tls-client-cert                     Client cert to use for TLS [$MACHINE_TLS_CLIENT_CERT]
  --tls-client-key                     Private key used in client TLS auth [$MACHINE_TLS_CLIENT_KEY]
  --github-api-token                     Token to use for requests to the Github API [$MACHINE_GITHUB_API_TOKEN]
  --native-ssh                        Use the native (Go-based) SSH implementation. [$MACHINE_NATIVE_SSH]
  --bugsnag-api-token                     BugSnag API token for crash reporting [$MACHINE_BUGSNAG_API_TOKEN]
  --help, -h                        show help
  --version, -v                        print the version

Commands:
  active        Print which machine is active
  config        Print the connection config for machine
  create        Create a machine
```

Docker Machine is somewhat abstract at first but is actually a very simple concept. Every time you use the `docker` command, the commands are executed on a Docker Engine host. Usually it is your local one, but it does not have to be. With the help of `docker-machine` you can run Docker commands on remotely managed hosts, as if they were local to you. Works like a charm and you will see it in action later in this guide.

## Docker Swarm

We will be using Docker version &gt; 17.03 with native Docker Swarm support, which has a host of exciting features that were not available previously such as multi-node deployments, overlay networks, service stacks, rolling deployments and so on. With the help of Docker Swarm commands we will be managing a cluster of Docker Engines called a `swarm` . The Docker Swarm mode \(previously a separate project, now natively supported in Docker\) saves us the trouble to deal with additional tools for scaling out and service management such as Kubernetes, or Apache Mesos, which while certainly useful might be an overkill for a small to medium sized project.

Let's tell Docker that we want to run in Swarm mode and initialize a cluster by issuing the following command:

```
➜ docker swarm init
Swarm initialized: current node (ojffj5dip3wue08ipnhyfbho8) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join \
    --token SWMTKN-1-3jwigkf9bqx39x09bcx134ikprxs4npd1z6z2k6t17pd1yjhe2-e9dde2im7nkw96lox81z37o85 \
    192.168.65.2:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

As you can see the initialization was successful and we can now run our services in a local single-node Swarm cluster. Later on our deployments will run on a remote multi-node Swarm cluster for redundancy and reliability. The next step we want to do is to create our own private network, in which we will be able to address our services by name without unnecessarily opening any ports.

`docker network create --driver=overlay --attachable leandot-net`

The important parts here are `--driver=overlay` and `--attachable` . `--driver=overlay` tells Docker Swarm to create a new overlay network, which extends to any nodes in the swarm that require it for a service, so that they can communicate as if on the same physical network. The second part`--attachable` is needed so that we can also attach standalone containers \(not only services\) to it for local development purposes.

You can list your current Docker networks and among them you should see the latest one we created:

```
➜  es docker network ls
NETWORK ID          NAME                    DRIVER              SCOPE
l5zrkdfuse32        leandot-net             overlay             swarm
c051ae03f9ed        host                    host                local
qjzhgyjm3rxh        ingress                 overlay             swarm
69806797eab3        pycharm_default         bridge              local
```

## PyCharm IDE

Python is a dynamic language with an immense set of libraries for diverse types of problems. It is mature, fast-enough and easy to learn. It also has all the building blocks we need and great IDE \(Integrated Development Environment\) support. Since we do not need critical speed or critical correctness, it is more than suitable for our tasks.

I am a big fan of using best-of-the-best when developing, much the same way you would happily spend a good amount of money for comfortable bed, robust shoes or a good monitor. PyCharm has everything you would need to work with the setup that is used in the book, great refactoring support and is the de-facto IDE for Python development. If you are a fan of other IDEs, or the more hacker-style Vim and Emacs I guess they would also work but I have not tested them.

PyCharm supports so-called remote interpreters for Python. What this means is that it can run the Python interpreter or read the installed packages from a remote machine via ssh and this is how we are going to use it. Python is a great language and has some reasonable options to install an isolated python environment per project, most notably with `virtualenv`. However, I still recommend setting it up remotely, because in that way you will eliminate the possibility for "but it worked on my machine" moments later on when deploying to production. Also the ability to use your `docker-compose.yml` file will enable your code to run in the docker environment of all services \(such as Elasticsearch or your webserver\), to use the same environment variables, hostnames, expose the needed ports and so on. It also is great for self-documenting all that is needed to run your code for a later stage, if you've taken a break from coding for a bit. This frictionless development setup is something that is easy to underestimate, but critical for the success of an indie-developed product. The remote-interpreter feature and require the Professional license \($19.99/month\) for PyCharm. It is an investment, but I'd really recommend it. If you prefer not to go down this road, install Python locally with virtualenv.

Head to the [PyCharm website](https://www.jetbrains.com/pycharm/), download and install the latest version for your OS. I am using PyCharm 2017.1.3 Professional. What is left is to tell PyCharm that we have a running Docker environment on our system and we want to make use of it. Open _PyCharm -&gt; Preferences_,  expand the Build, Execution, Deployment section and select _Docker_. You probably do not have this configured yet so it should be empty. Click on the + symbol and configure it as follows:

![](/assets/docker_for_mac.png)

On Mac OS, Docker clients communicate with the Docker environment via docker.sock. On other systems it might be a tcp URL.

That is the end of our introduction to the Docker ecosystem and the PyCharm IDE. In the next post from the series we will look at how to configure a project to use a Python interpreter from a Docker container.

