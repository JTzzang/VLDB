# How to Run LinkBench@MySQL on DockerContainer

## Ref
https://hub.docker.com/r/meeeejin/lb-mysql

## 1. Pull Docker Images
``` bash
$ sudo docker pull meeeejin/lb-mysql:latest
```

## 2. Permission 
```bash
$ sudo chown -R 999:docker [data_dir]
$ sudo chown -R 999:docker [log_dir]
```

## 3. Create Container 
###mapping

```bash
$ sudo docker run -it \
--name [container name] \
-v [data_dir]:/var/lib/mysql \
-v [log_dir]:/var/log/mysql \
-v [cnf_dir]:/etc/mysql/mysql.conf.d \
-v [experiment_log_dir]:/local_log \
meeeejin/lb-mysql:latest

# mysql root pw : vldb1234

```
## 4. load data
```bash 
$ sudo docker exec -it [container_name ] /bin/bash

#in container

$ mysql -uroot -pvldb1234

mysql> create database linkdb;

mysql> use linkdb;

mysql> CREATE TABLE `linktable` (
  `id1` bigint(20) unsigned NOT NULL DEFAULT '0',
  `id2` bigint(20) unsigned NOT NULL DEFAULT '0',
  `link_type` bigint(20) unsigned NOT NULL DEFAULT '0',
  `visibility` tinyint(3) NOT NULL DEFAULT '0',
  `data` varchar(255) NOT NULL DEFAULT '',
  `time` bigint(20) unsigned NOT NULL DEFAULT '0',
  `version` int(11) unsigned NOT NULL DEFAULT '0',
  PRIMARY KEY (link_type, `id1`,`id2`),
  KEY `id1_type` (`id1`,`link_type`,`visibility`,`time`,`id2`,`version`,`data`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1 PARTITION BY key(id1) PARTITIONS 16;

CREATE TABLE `counttable` (
  `id` bigint(20) unsigned NOT NULL DEFAULT '0',
  `link_type` bigint(20) unsigned NOT NULL DEFAULT '0',
  `count` int(10) unsigned NOT NULL DEFAULT '0',
  `time` bigint(20) unsigned NOT NULL DEFAULT '0',
  `version` bigint(20) unsigned NOT NULL DEFAULT '0',
  PRIMARY KEY (`id`,`link_type`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1;

CREATE TABLE `nodetable` (
  `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT,
  `type` int(10) unsigned NOT NULL,
  `version` bigint(20) unsigned NOT NULL,
  `time` int(10) unsigned NOT NULL,
  `data` mediumtext NOT NULL,
  PRIMARY KEY(`id`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1;


mysql> quit

load 전에 config/FBWorkload 에서 18000000001500000000으로 바꿔주기
# load
##  in root/linkbench
$ ./bin/linkbench -c config/MyConfig.properties -l

```


## 5. Run

1. modify run.sh

```bash 
$ sudo docker exec -it [container_name ] /bin/bash
```

```bash

cd root/linkbench
vi run.sh
# 아래와 같이 수정

./bin/linkbench -c config/MyConfig.properties -csvstats /local_log/[log_name]_final-stats.csv -csvstream /local_log/[log_name]_streaming-stats.csv -L /local_log/[log_name]_lb.lb -r
```


