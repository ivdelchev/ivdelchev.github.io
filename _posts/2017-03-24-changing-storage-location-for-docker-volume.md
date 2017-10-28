---
layout: post
title: Changing the Docker volume storage location
tags: [Docker]
description: How to change the default Docker volume storage location with existing data, step-by-step.
---

At least on my Centos 7 distribution, the default location for docker volumes is `/var/lib/docker/volumes` and in order to change that you need to point the whole docker base path to another location. Sometimes this is not what you want, you might just want to have a particular volume write on an external mounted drive. This is often the case for data - e.g. for Postgres, Elasticsearch etc.

What worked for me in that case was to first symlink to the external folder and then create the volume.

```
cd /var/lib/docker/volumes/
ln -s /your/data/path/ volume-name
docker volume create --name volume-name
```

