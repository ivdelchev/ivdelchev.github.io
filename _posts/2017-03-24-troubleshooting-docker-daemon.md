---
layout: post
title: Troubleshooting the Docker Daemon
tags: [Docker]
description: How to start the Docker Daemon in debug mode and check if your distribution supports Docker fully.
---

I recently had an issue with my docker engine and had to debug in more detail. In order to start the daemon in debug mode edit (or create if missing) `/etc/docker/daemon.json` and set `{"debug":true}`. More information here - [Daemon Configuration File](https://docs.docker.com/engine/reference/commandline/dockerd/#daemon-configuration-file).

This script [Check Config](https://github.com/docker/docker/blob/master/contrib/check-config.sh) is also quite useful, it will tell you whether your distribution meets the requirements for docker.
