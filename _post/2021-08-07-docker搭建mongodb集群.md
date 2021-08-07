---
title: docker搭建mongodb集群
date: 2021-08-07 +0800
categories: [mongodb, docker]
tags: [mongodb, docker]     # TAG names should always be lowercase
---



本文按照mongodb网站上的[指导教程](https://docs.mongodb.com/manual/tutorial/deploy-shard-cluster/)使用docker搭建了一个shard集群，供学习参考。

该配置适用于Linux系统，Mac系统上的docker网络实现与Linux不相同，宿主机与容器通信会存在问题，需要另行处理。

现有如下目录结构：
```
.
├── docker-compose.yaml
└── shard-setup
    └── setup.sh
```

其中，`docker-compose.yaml`内容如下，文章撰写时，mongodb版本最新为5.0.0：
```yaml
version: "3.8"

services:
  shard-setup:
    hostname: shard-setup
    container_name: shard-setup
    image: mongo:5.0.0
    entrypoint: /usr/bin/bash /shard-setup/setup.sh
    volumes: 
      - ./shard-setup:/shard-setup
    networks: 
      - mongo-shard-net
    depends_on: 
      - mongos-1
    restart: "no"

  mongos-1:
    hostname: mongos-1
    container_name: mongos-1
    image: mongo:5.0.0
    entrypoint: 
      - "mongos" 
      - "--configdb"
      - "cfgrs0/cfgrs0-1:27019,cfgrs0-2:27019,cfgrs0-3:27019"
      - "--bind_ip_all"
    networks: 
      mongo-shard-net:
        ipv4_address: 172.28.0.51
    expose: 
      - 27017
    restart: on-failure
    depends_on: 
      - cfgrs0-1
      - cfgrs0-2
      - cfgrs0-3
      - dbrs0-1
      - dbrs0-2
      - dbrs0-3
      - dbrs1-1
      - dbrs1-2
      - dbrs1-3

  cfgrs0-1:
    hostname: cfgrs0-1
    container_name: cfgrs0-1
    image: mongo:5.0.0
    entrypoint: /usr/bin/mongod --configsvr --replSet "cfgrs0" --bind_ip_all
    networks: 
      - mongo-shard-net
    restart: on-failure
    expose: 
      - "27019"
  cfgrs0-2:
    hostname: cfgrs0-2
    container_name: cfgrs0-2
    image: mongo:5.0.0
    entrypoint: /usr/bin/mongod --configsvr --replSet "cfgrs0" --bind_ip_all
    networks: 
      - mongo-shard-net
    restart: on-failure
    expose: 
      - "27019"
  cfgrs0-3:
    hostname: cfgrs0-3
    container_name: cfgrs0-3
    image: mongo:5.0.0
    entrypoint: /usr/bin/mongod --configsvr --replSet "cfgrs0" --bind_ip_all
    networks: 
      - mongo-shard-net
    restart: on-failure
    expose: 
      - "27019"

  # 配置dbrs0和dbrs1
  dbrs0-1:
    hostname: dbrs0-1
    container_name: dbrs0-1
    image: mongo:5.0.0
    entrypoint: /usr/bin/mongod --shardsvr --replSet "dbrs0" --bind_ip_all
    networks: 
      - mongo-shard-net
    restart: on-failure
    expose: 
      - "27018"
  dbrs0-2:
    hostname: dbrs0-2
    container_name: dbrs0-2
    image: mongo:5.0.0
    entrypoint: /usr/bin/mongod --shardsvr --replSet "dbrs0" --bind_ip_all
    networks: 
      - mongo-shard-net
    restart: on-failure
    expose: 
      - "27018"
  dbrs0-3:
    hostname: dbrs0-3
    container_name: dbrs0-3
    image: mongo:5.0.0
    entrypoint: /usr/bin/mongod --shardsvr --replSet "dbrs0" --bind_ip_all
    networks: 
      - mongo-shard-net
    restart: on-failure
    expose: 
      - "27018"

  dbrs1-1:
    hostname: dbrs1-1
    container_name: dbrs1-1
    image: mongo:5.0.0
    entrypoint: /usr/bin/mongod --shardsvr --replSet "dbrs1" --bind_ip_all
    networks: 
      - mongo-shard-net
    restart: on-failure
    expose: 
      - "27018"
  dbrs1-2:
    hostname: dbrs1-2
    container_name: dbrs1-2
    image: mongo:5.0.0
    entrypoint: /usr/bin/mongod --shardsvr --replSet "dbrs1" --bind_ip_all
    networks: 
      - mongo-shard-net
    restart: on-failure
    expose: 
      - "27018"
  dbrs1-3:
    hostname: dbrs1-3
    container_name: dbrs1-3
    image: mongo:5.0.0
    entrypoint: /usr/bin/mongod --shardsvr --replSet "dbrs1" --bind_ip_all
    networks: 
      - mongo-shard-net
    restart: on-failure
    expose: 
      - "27018"

networks: 
  mongo-shard-net:
    name: mongo-shard-net
    ipam: 
      config: 
        - subnet: 172.28.0.0/16
```
`setup.sh`用于设置config以及shard主从，内容如下：

```shell
#/bin/bash

CFGRS0_NAME=cfgrs0
CFGRS0_REPLICA_1=${CFGRS0_NAME}-1
CFGRS0_REPLICA_2=${CFGRS0_NAME}-2
CFGRS0_REPLICA_3=${CFGRS0_NAME}-3

CFGSVR_PORT=27019

DBRS0_NAME=dbrs0
DBRS0_REPLICA_1=${DBRS0_NAME}-1
DBRS0_REPLICA_2=${DBRS0_NAME}-2
DBRS0_REPLICA_3=${DBRS0_NAME}-3

DBRS1_NAME=dbrs1
DBRS1_REPLICA_1=${DBRS1_NAME}-1
DBRS1_REPLICA_2=${DBRS1_NAME}-2
DBRS1_REPLICA_3=${DBRS1_NAME}-3

DBSVR_PORT=27018

until mongosh --host ${CFGRS0_REPLICA_1} --port ${CFGSVR_PORT} --quiet <<EOF
exit
EOF
do
  sleep 5
done
mongosh --host ${CFGRS0_REPLICA_1} --port ${CFGSVR_PORT} --quiet <<EOF
rs.initiate(
  {
    _id: "${CFGRS0_NAME}",
    configsvr: true,
    members: [
      { _id : 0, host : "${CFGRS0_REPLICA_1}:${CFGSVR_PORT}", priority: 2 },
      { _id : 1, host : "${CFGRS0_REPLICA_2}:${CFGSVR_PORT}", priority: 1 },
      { _id : 2, host : "${CFGRS0_REPLICA_3}:${CFGSVR_PORT}", priority: 1 }
    ]
  }
)
EOF

for rs in 0 1
do
  until mongosh --host `eval echo '$'{DBRS"${rs}"_REPLICA_1}` --port ${DBSVR_PORT} --quiet <<EOF
exit
EOF
  do
    sleep 5
  done
mongosh --host `eval echo '$'{DBRS"${rs}"_REPLICA_1}` --port ${DBSVR_PORT} --quiet <<EOF
rs.initiate(
  {
    _id: "`eval echo '$'{DBRS"${rs}"_NAME}`",
    members: [
      { _id : 0, host : "`eval echo '$'{DBRS"${rs}"_REPLICA_1}`:${DBSVR_PORT}", priority: 2 },
      { _id : 1, host : "`eval echo '$'{DBRS"${rs}"_REPLICA_2}`:${DBSVR_PORT}", priority: 1 },
      { _id : 2, host : "`eval echo '$'{DBRS"${rs}"_REPLICA_3}`:${DBSVR_PORT}", priority: 1 }
    ]
  }
)
EOF
done

# mongos --configdb ${CFGRS0_NAME}/${CFGRS0_REPLICA_1}:${CFGSVR_PORT},\
# ${CFGRS0_REPLICA_2}:${CFGSVR_PORT},${CFGRS0_REPLICA_3}:${CFGSVR_PORT} \
# --bind_ip_all &

MONGOS=mongos-1

# 配置shard
until mongosh --host ${MONGOS} --quiet <<EOF
exit
EOF
do
  sleep 5
done

mongosh --host ${MONGOS} --quiet <<EOF
sh.addShard("${DBRS0_NAME}/${DBRS0_REPLICA_1}:${DBSVR_PORT},${DBRS0_REPLICA_2}:${DBSVR_PORT},${DBRS0_REPLICA_3}:${DBSVR_PORT}")
sh.addShard("${DBRS1_NAME}/${DBRS1_REPLICA_1}:${DBSVR_PORT},${DBRS1_REPLICA_2}:${DBSVR_PORT},${DBRS1_REPLICA_3}:${DBSVR_PORT}")
EOF

```
使用`docker-compose up`即可启动集群，使用`docker-compose down --volumes`即可停止并删除集群。

使用`docker-compose stop`可以保存此次启动的集群容器，下次使用`docker-compose start`启动既可以恢复，可以看到上次启动时mongodb保存的数据。

PS：仅使用`docker-compose down`可能会致使磁盘空间不断减小，每次挂载的volume在`/var/lib/docker`目录下会越积越多。