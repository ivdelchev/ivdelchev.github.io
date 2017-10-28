---
layout: post
title: Creating a snapshot of a dockerized Elasticsearch
tags: [Docker, Elasticsearch]
description: Elasticsearch has a great snapshot feature, which works for clusters and supports different output repos - file system, S3, etc. In this short post I will show how to enable it for a dockerized ES with a file system (fs) output.
---

Elasticsearch has a great snapshot feature, which works for clusters and supports different
output repos - file system, S3, etc. In this short post I will show how to enable it for a
dockerized ES with a file system (fs) output.

The first thing we need is to add the output folder in the Dockerfile for Elasticsearch (this is the folder inside the container, not on the host).

```bash
RUN echo 'path.repo: ["/opt/elasticsearch/backup"]' >> /usr/share/elasticsearch/config/elasticsearch.yml
```

We need a docker volume, which we will later mount to the ES container, so that the data can be persisted.

```python
docker volume create --name=leandot-backup-prod
```

Now we are ready to build our custom ES image.

```python
version: '2'
services:
  es:
    build: .
    image: leandot-es:1.0.0
    volumes:
      - leandot-data-prod:/usr/share/elasticsearch/data
      - leandot-backup-prod:/opt/elasticsearch/backup
volumes:
  leandot-data-prod:
    external: true
  leandot-backup-prod:
    external: true
```

After recreating the container we can trigger the snapshot repo initialization.

```bash
curl -XPUT http://leandot-git:9200/_snapshot/es_backup -d '{
    "type": "fs",
    "settings": {
        "location": "/opt/elasticsearch/backup"
    }
}'
```
However, we get a response saying:

```json
{"error":"RepositoryVerificationException[[es_backup] path  is not accessible on master node]; nested: FileNotFoundException[/opt/elasticsearch/backup/tests-Y1AbUndPQ_O1je_zyFGa1Q-master (Permission denied)]; ","status":500}
```

It seems we have a permission issue with the folder. Let's put those debugging skills to use. Login into the running container with:

```bash
docker exec -it CONTAINER_ID bash
```

and let's examine the folder and the process permissions. First of all let's check the user with which the ES process is running:

```bash
ps aux | grep elastic
```

gives us

```
elastic+     1  1.2 22.6 3134220 231304 ?      Ssl  22:59   0:10 /usr/lib/jvm/java-8-openjdk-amd64/jre/bin/java -Xms256m -Xmx1g -Djava.awt.headless=true -XX:+UseParNewGC -XX:+UseConcMarkSweepGC -XX:CMSInitiatingOccupancyFraction=75 -XX:+UseCMSInitiatingOccupancyOnly -XX:+HeapDumpOnOutOfMemoryError -XX:+DisableExplicitGC -Dfile.encoding=UTF-8 -Delasticsearch -Des.foreground=yes -Des.path.home=/usr/share/elasticsearch -cp :/usr/share/elasticsearch/lib/elasticsearch-1.7.4.jar:/usr/share/elasticsearch/lib/*:/usr/share/elasticsearch/lib/sigar/* org.elasticsearch.bootstrap.Elasticsearch
root        55  0.0  0.1  12816  1028 ?        S+   23:12   0:00 grep elastic
```

So we see that we are running ES with a user elasticsearch (truncated in that case, you can run something like ps axo user:20,pid,... to get the full name). Checking the folder, where we would like to snapshot our data reveals that it is owned by `root` and nobody else has write permissions on it.

```bash
root@d6e79a296527:/opt/elasticsearch# ls -al /opt/elasticsearch
drwxr-xr-x 2 root root 4096 Mar  4 22:56 backup
```

The problem is that the volume mounted from the outside into the folder `/opt/elasticsearch/backup` will have `root` as owner unless we do something about it when starting the container. Actually this is exactly what Elasticsearch is doing with its folders for data and logs. Since that can't happen at build time (remember, we are mounting the folder at container start) then we need to run a script on boot. This pattern has become very widespread nowadays - often there is a bash script named `docker-entrypoint.sh`, which is set as the image entrypoint.  I don't know a better way to modify the script than copying the original `docker-entrypoint.sh` file, changing it and making sure it is copied to the image with the Dockerfile. Keep in mind that in this case you will essentially overwrite the original script, so if it gets updated in the original image you will not get the changes.

Now we modify the original script to `chown` the backup folder to user `elasticsearch`

```bash
# Change the ownership of user-mutable directories to elasticsearch
for path in \
	/usr/share/elasticsearch/data \
	/usr/share/elasticsearch/logs \
	/opt/elasticsearch/backup \
; do
	chown -R elasticsearch:elasticsearch "$path"
done
```

and update the Dockerfile to include the script on build, otherwise we will retain the original one, already added to the image.

```bash
COPY docker-entrypoint.sh /
```

Submitting the PUT request to initialize a new snapshot repo now works and we get a much more friendly

```bash
curl -XPUT http://leandot-git:9200/_snapshot/es_backup -d '{
     "type": "fs",
     "settings": {
         "location": "/opt/elasticsearch/backup"
     }
}'
{"acknowledged":true}
```

The final code can be found in the [leandot-git repo](https://github.com/ivdelchev/leandot-git/tree/master/es-snapshot).

