---
title: rainbond迁移数据库
date: 2022-06-10 17:23:31
tags:
categories: rainbond
---

气死人的一次
<!-- more -->

```
kubectl edit rainbondcluster -n rbd-system #进入修改数据库地址
kubectl delete pod -l name=rainbond-operator -n rbd-system  #重启控制节点
kubectl get pod -l name=rbd-api -n rbd-system -o yaml   #查看是否生效
最重要的一点数据备份
```

```
docker run --name=rainbond-allinone --add-host=rbd-api-api:10.43.74.134   --restart=always -d -p 7070:7070 -v ~/.ssh:/root/.ssh -v ~/rainbonddata:/app/data -v ~/license:/opt/license -e DISABLE_DEFAULT_APP_MARKET=true -e IS_ENTERPRISE_EDITION=true -e INSTALL_IMAGE_REPO=goodrain.me -e RAINBOND_VERSION=enterprise-2110 -e MYSQL_HOST=rm-1fyl65r2dah0kd9a2.mysql.rds.inner.jhszwy.net  -e MYSQL_PORT=3306 -e MYSQL_USER=ytrb -e MYSQL_PASS=ytRb@dblG2022 -e DB_TYPE=mysql -e MYSQL_DB=console -e LICENSE_PATH=/opt/license/hzyt-goodrain-CONSOLE-LICENSE.yb -e LICENSE_KEY=1234567890  goodrain.me/rainbond:enterprise-2110-allinone

create database console character set utf8 collate utf8_general_ci;
```

```
初始化集群出现数据库迁移但是没有备份的情况如下处理

docker run 或 rbd-api-ui 和 region库，连接数据库自动初始化

这时进入的是一个初始化没有集群的平台，我门采用 

grctl config

接入已安装的集群
```

