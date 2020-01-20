---
layout:  post    # 使用的布局（不需要改）
catalog: true   # 是否归档
author: 陈国林 # 作者
tags:         #标签
    - TSDB
---

# 一. 常用命令
## ① database
1. 查询database: `$ SHOW database`
2. 创建database: `$ CREATE database {database_name}`
3. 删除database: `$ DROP database {database_name}`

## ② measurement
5. 删除所有的measurement: `$ DROP SERIES FROM /.*/`
6. 查看某个measurement的所有tags: `$ SHOW tag keys from {measurement_name}`
7. 查看某个measurement的所有fields: `$ SHOW field keys from {measurement_name}`

## ③ continuous query（CQ）
1. 查询CONTINUOUS QUERY: `$ SHOW CONTINUOUS QUERY`
2. 删除CONTINUOUS QUERY: `$ DROP CONTINUOUS QUERY {cq_name} ON {database_name}`
3. CQ创建
   ```
   CREATE CONTINUOUS QUERY xxxx_cq ON {database_name} 
   RESAMPLE FOR 20m 
   BEGIN SELECT 
   sum(play_num) AS play_num, 
   ... 
   INTO xxx.rp_10m.yyy 
   FROM xxx.autogen.yyy GROUP BY time(10m), * 
   END
   ```
   
## ④ retention policy(RP)   
1. 查询RETENTION POLICY: `$ SHOW RETENTION POLICIES`
2. 创建RETENTION POLICY: `$ CREATE RETENTION POLICY {retention_policy_name} ON {database_name} DURATION {duration} REPLICATION {n} [SHARD DURATION {duration}] [DEFAULT]`
3. 修改RETENTION POLICY: `$ ALTER RETENTION POLICY {rp_name} ON {database_name} DURATION {duration} REPLICATION {n} SHARD DURATION {duration} DEFAULT`
4. 删除RETENTION POLICY: `$ DROP RETENTION POLICY {rp_name} ON {database_name}`

# 二. API
InfluxDB API提供了较简单的方式用于数据库交互，该API使用了HTTP的方式，并以JSON格式进行返回。

## ① /ping
`/ping` 支持GET和HEAD，都可用于获取指定信息

1. 获取InfluxDB版本信息。
   ```
   $ curl -sl -I http://localhost:8086/ping
   HTTP/1.1 204 No Content
   Content-Type: application/json
   Request-Id: ebe357b8-ea19-11e7-8001-000000000000
   X-Influxdb-Version: v1.3.6
   Date: Tue, 26 Dec 2017 08:51:11 GM
   ```

## ② /query
`/query` 支持GET和POST的HTTP请求，可用于查询数据和管理数据库。

1. 使用SELECT查询数据
   ```
   $ curl -G "http://localhost:8086/query?db=mydb" --data-urlencode 'q=SELECT * FROM "mymeas"'  
   {"results":[{"series":[{"name":"mymeas","columns":["time","myfield","mytag1","mytag2"],"values":[["2016-05-20T21:30:00Z",12,"1",null],["2016-05-20T21:30:20Z",11,"2",null],["2016-05-20T21:30:40Z",18,null,"1"],["2016-05-20T21:31:00Z",19,null,"3"]]}]}]}
   ```

2. 使用INTO字段
   ```
   $ curl -XPOST "http://localhost:8086/query?db=mydb" --data-urlencode 'q=SELECT * INTO "newmeas" FROM "mymeas"'  
   {"results":[{"series":[{"name":"result","columns":["time","written"],"values":[["1970-01-01T00:00:00Z",4]]}]}]}
   ```
   
3. 创建数据库
   ```
   $ curl -XPOST "http://localhost:8086/query" --data-urlencode 'q=CREATE DATABASE "mydb"'  
   {"results":[{}]}
   ```

4. 使用http认证来创建数据库
   ```
   $ curl -XPOST "http://localhost:8086/query?u=myusername&p=mypassword" --data-urlencode 'q=CREATE DATABASE "mydb"'  
   {"results":[{}]}
   ```

5. 使用基础认证来创建数据库
   ```
   $ curl -XPOST -u myusername:mypassword "http://localhost:8086/query" --data-urlencode 'q=CREATE DATABASE "mydb"'  
   {"results":[{}]}
   ```

## ③ /write
`/wirte` 只支持POST的HTTP请求，使用该Endpoint可以写数据到已存在的数据库中。

1. 使用秒级的时间戳，将一个point写入数据库mydb
   ```
   $ curl -i -XPOST "http://localhost:8086/write?db=mydb&precision=s" --data-binary 'mymeas,mytag=1myfield=90 1463683075'
   ```

2. 将一个point写入数据库mydb，并指定RP为myrp
   ```
   $ curl -i -XPOST "http://localhost:8086/write?db=mydb&rp=myrp" --data-binary 'mymeas,mytag=1 myfield=90'
   ```

3. 使用HTTP认证的方式，将一个point写入数据库mydb
   ```
   $ curl -i -XPOST "http://localhost:8086/write?db=mydb&u=myusername&p=mypassword" --data-binary 'mymeas,mytag=1 myfield=91'
   ```

4. 使用基础认证的方式，将一个point写入数据库mydb
   ```
   $ curl -i -XPOST -u myusername:mypassword "http://localhost:8086/write?db=mydb" --data-binary 'mymeas,mytag=1 myfield=91'
   ```

5. 写多个points到数据库中，需要使用新的一行
   ```
   $ curl -i -XPOST "http://localhost:8086/write?db=mydb" --data-binary 'mymeas,mytag=3,myfield=89,mymeas,mytag=2,myfield=34 1463689152000000000'
   ```

6. 通过导入文件的形式，写入多个points，需要使用`@`来指定文件
   ```
   $ curl -i -XPOST "http://localhost:8086/write?db=mydb" --data-binary @data.txt
   @data.txt 文件内容如下
   mymeas,mytag1=1 value=21 1463689680000000000
   mymeas,mytag1=1 value=34 1463689690000000000
   mymeas,mytag2=8 value=78 1463689700000000000    
   mymeas,mytag3=9 value=89 1463689710000000000
   ```

