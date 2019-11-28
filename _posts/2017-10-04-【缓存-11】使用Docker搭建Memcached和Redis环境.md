---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - 缓存
---

# 一. Memcached部署
1. 下载1.5版本镜像: `$ docker pull memcached:1.5`
2. 启动Memcached: `$ docker run -name memcached -m 256m -d -p 11211:11211 memcached:1.5`
3. 按照Memcached-cli: `$ npm install -g memcached-cli`
4. 使用Memcached
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
