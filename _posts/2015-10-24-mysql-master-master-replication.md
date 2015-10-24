---
layout: post
title: "MySQL双主master-master配置实践"
categories:
  - MySQL
---


一直想着要自己亲手配置MySQL双主（master-master）相互同步的配置，趁着今天没有导出乱跑，吃完午饭就开始鼓捣，一下午终于鼓捣完了，现在把过程记录一下。

MySQL的环境直接通过[docker](https://www.docker.com/ docker)做部署的，image使用的是[mysql:5.7](https://hub.docker.com/_/mysql/)，因为需要修改MySQL的配置，所以就先启动了一个名为test-mysql的容器，然后通过`docker cp`命令把MySQL的配置文件和数据文件复制到宿主机上：
	
	docker cp test-mysql:/etc/mysql/ /data/mysql_1/conf/
	docker cp test-mysql:/var/lib/mysql /data/mysql_1/data/
	docker cp test-mysql:/etc/mysql/ /data/mysql_2/conf/
	docker cp test-mysql:/var/lib/mysql /data/mysql_2/data/

修改MySQL的配置文件/data/mysql\_1/conf/my.cnf，增加以下两列配置：

	server-id       = 1
	log-bin         = mysql-bin

/data/mysql\_1/conf/my.cnf文件增加下列配置：

	server-id       = 1
	log-bin         = mysql-bin

同时需要把/data/mysql\_1/data/auto.conf文件删掉，这个文件是由MySQL生成的，文件里存有server-uuid，第一次配置的时候没有做这一步，slave在启动时报错：*Last_IO_Error: Fatal error: The slave I/O thread stops because master and slave have equal MySQL server UUIDs; these UUIDs must be different for replication to work.*

然后是启动MySQL，mysql\_1和mysql\_2两个容器：

	docker run --name mysql_1 -p 3331:3306 -v /data/mysql_1/conf:/etc/mysql -v /data/mysql_1/data:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=root -d mysql:5.7
	docker run --name mysql_2 -p 3332:3306 -v /data/mysql_2/conf:/etc/mysql -v /data/mysql_2/data:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=root -d mysql:5.7

连接到mysql\_1，配置相应的MySQL replication配置：

	mysql -h0.0.0.0 -P3331 -uroot -proot
	create user 'replicator'@'%' identified by 'replicator';
	grant replication slave on *.* to 'replicator'@'%'; 
	CHANGE MASTER TO MASTER_HOST='172.17.0.13',MASTER_USER='replicator',MASTER_PASSWORD='replicator',MASTER_LOG_FILE='mysql-bin.000001',MASTER_LOG_POS=1305;
	start slave;

其中，MASTER\_HOST指向的ip为mysql\_2的容器ip，可以通过`docker inspect --format '{{.NetworkSettings.IPAddress}}'`命令查看。MASTER\_LOG\_FILE、MASTER\_LOG\_POS可以在mysql\_2下通过`show master status`查看。mysql\_2的主从配置也做相应的配置即可。
![show-master-status.jpg](/images/show-master-status.jpg)

到此所有的配置都完成，在mysql\_1下创建一个数据库，在mysql\_2下就可以看到了。

配置过程中参考了此篇[文章](https://www.digitalocean.com/community/tutorials/how-to-set-up-mysql-master-master-replication)，在此表示感谢。