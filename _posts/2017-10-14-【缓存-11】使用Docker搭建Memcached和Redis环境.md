---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - 缓存
---

# 一. Memcached部署
1. 下载 1.5版本镜像: `$ docker pull memcached:1.5`
2. 启动 memcached: `$ docker run -d -name memcached -m 256m -p 11211:11211 --privileged=true memcached:1.5`
3. 安装 memcached-cli: `$ npm install -g memcached-cli`
4. 使用 memcached
   ```
   $ npx memcached-cli 127.0.0.1:11211
   127.0.0.1:11211> help

   Commands:

     help [command...]                   Provides help for a given command.
     exit                                Exits application.
     get <key>                           Get the value of a key
     set <key> <value> [expires]         Set the value of a key
     add <key> <value> [expires]         Set the value of a key, fail if key exists
     replace <key> <value> [expires]     Overwrite existing key, fail if key not exists
     delete <key>                        Delete a key
     increment <key> <amount> [expires]  increment
     decrement <key> <amount> [expires]  decrement
     flush                               Flush all data
     stats                               Show statistics
   ```

# 二. Redis部署
1. 下载 5.0版本镜像: `$ docker pull redis:5.0`
2. 下载 redis.conf文件: `$ curl "http://download.redis.io/redis-stable/redis.conf" > /www/redis/conf/redis.conf`
3. 启动 redis: `$ docker run -d --name redis -p 6379:6379 -v /www/redis/conf/redis.conf:/etc/redis/redis.conf --privileged=true redis:5.0 redis-server /etc/redis/redis.conf`
4. 安装 redis-cli: `$ npm install -g redis-cli`
5. 使用 redis
   ```
   $ redis-cli -h 127.0.0.1 -p 6379
   127.0.0.1:6379> help
   redis-cli 4.0.1
   To get help about Redis commands type:
          "help @<group>" to get a list of commands in <group>
          "help <command>" for help on <command>
          "help <tab>" to get a list of possible help topics
          "quit" to exit

   To set redis-cli preferences:
          ":set hints" enable online hints
          ":set nohints" disable online hints
   Set your preferences in ~/.redisclirc
   ```
