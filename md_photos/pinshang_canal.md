pinshang_canal



## 部署ZooKeeper



~~~
  cd /opt/zookeeper-3.5.5/conf
  vim zoo.cfg
  cd /opt/zookeeper-3.5.5/data
  cat ../conf/zoo.cfg
  cat myid
  pwd
  rm -rf /opt/zookeeper-3.5.5/data/*
  echo 3 > myid
  rm -rf /zookeeper-3.5.5/logs/*
  echo 'export ZOOKEEPER_HOME=/opt/zookeeper-3.5.5' >> /etc/profile
  echo 'export PATH=$PATH:$ZOOKEEPER_HOME/bin:$ZOOKEEPER_HOME/conf' >> /etc/profile
  source /etc/profile
  zkServer.sh start
  ps -ef | grep zookeeper
  zkCli.sh -server 127.0.0.1:2181
  history



~~~



## DB 端 授权及配置



~~~javascript
# mysql 5.7:
		GRANT SELECT, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'canalBinlog'@'192.168%' identified by 'CanalBinlog@888';
		FLUSH PRIVILEGES;
创建账号：	
		# 创建用户 用户名：canal 密码：Canal@123456
		create user 'canal'@'%' identified by 'Canal@123456';

		# mysql8需要执行这句，将加密规则还原成mysql_native_password
		ALTER USER 'canal'@'%%' IDENTIFIED WITH mysql_native_password BY 'canal';

权限授予：
	# 授权 *.*表示所有库
		grant SELECT, REPLICATION SLAVE, REPLICATION CLIENT on *.* to 'canal'@'%' identified by 'Canal@123456';
		FLUSH PRIVILEGES; #刷新权限表（必须）
		
		
~~~



~~~
my.cnf
	[mysqld]
		# 打开binlog
		log-bin=mysql-bin

		# 选择ROW(行)模式
		binlog-format=ROW

		# 配置MySQL replaction需要定义，不要和canal的slaveId重复
		server_id=1
~~~





## Canal Admin 安装



~~~javascript

tar -zxf canal.admin-1.1.4.tar.gz -C /usr/local

# 创建 后台管理库
mysql -uroot -p'qm1qaz@WSX' < canal_manager.sql

#canal管理账号
create user canal_user@'192.168%' identified by 'CanalManager@888';
grant select,insert,delete,update on canal_manager.* to canal_user@'192.168%';
flush privileges;


# 配置 admin

vim /usr/local/canal-admin/conf/application.yml

server:
  port: 8089
spring:
  jackson:
    date-format: yyyy-MM-dd HH:mm:ss
    time-zone: GMT+8

spring.datasource:
  address: 192.168.101.32:3306
  database: pinshang_canal_manager
  username: canal_user
  password: CanalManager@888
  driver-class-name: com.mysql.jdbc.Driver
  url: jdbc:mysql://${spring.datasource.address}/${spring.datasource.database}?useUnicode=true&characterEncoding=UTF-8&useSSL=false
  hikari:
    maximum-pool-size: 30
    minimum-idle: 1

canal:
  adminUser: admin
  adminPasswd: admin



启动：
sh /usr/local/canal-admin/bin/startup.sh
ps -ef | grep canal

http://192.168.0.93:8089
默认账号：admin/123456
~~~



## 安装 server



###  admin 配置 集群

![image-20210909182047188](https://gitee.com/zzjk_01/cloudimages/raw/master/img/image-20210909182047188.png)





![image-20210909182706902](https://gitee.com/zzjk_01/cloudimages/raw/master/img/image-20210909182706902.png)



~~~
canal.properties 配置如下：


#################################################
######### 		common argument		#############
#################################################
# tcp bind ip
canal.ip =
# register ip to zookeeper
canal.register.ip =
canal.port = 11121
canal.metrics.pull.port = 11122
# canal instance user/passwd
#canal.user = canal
#canal.passwd = E3619321C1A937C46A0D8BD1DAC39F93B27D4458
#canal.user = 
#canal.passwd = 

# canal admin config
canal.admin.manager = http://192.168.103.55:8089
canal.admin.port = 11120
canal.admin.user = admin
canal.admin.passwd = 4ACFE3202A5FF5CF467898FC58AAB1D615029441

canal.zkServers = 192.168.103.97:2181,192.168.103.99:2181,192.168.103.105:2181
# flush data to zk
canal.zookeeper.flush.period = 1000
canal.withoutNetty = false
# tcp, kafka, RocketMQ
canal.serverMode = tcp
# flush meta cursor/parse position to file
canal.file.data.dir = ${canal.conf.dir}
canal.file.flush.period = 1000
## memory store RingBuffer size, should be Math.pow(2,n)
canal.instance.memory.buffer.size = 16384
## memory store RingBuffer used memory unit size , default 1kb
canal.instance.memory.buffer.memunit = 1024 
## meory store gets mode used MEMSIZE or ITEMSIZE
canal.instance.memory.batch.mode = MEMSIZE
canal.instance.memory.rawEntry = true

## detecing config
canal.instance.detecting.enable = false
#canal.instance.detecting.sql = insert into retl.xdual values(1,now()) on duplicate key update x=now()
canal.instance.detecting.sql = select 1
canal.instance.detecting.interval.time = 3
canal.instance.detecting.retry.threshold = 3
canal.instance.detecting.heartbeatHaEnable = false

# support maximum transaction size, more than the size of the transaction will be cut into multiple transactions delivery
canal.instance.transaction.size =  1024
# mysql fallback connected to new master should fallback times
canal.instance.fallbackIntervalInSeconds = 60

# network config
canal.instance.network.receiveBufferSize = 16384
canal.instance.network.sendBufferSize = 16384
canal.instance.network.soTimeout = 30

# binlog filter config
canal.instance.filter.druid.ddl = true
canal.instance.filter.query.dcl = false
canal.instance.filter.query.dml = false
canal.instance.filter.query.ddl = false
canal.instance.filter.table.error = false
canal.instance.filter.rows = false
canal.instance.filter.transaction.entry = false

# binlog format/image check
canal.instance.binlog.format = ROW,STATEMENT,MIXED 
canal.instance.binlog.image = FULL,MINIMAL,NOBLOB

# binlog ddl isolation
canal.instance.get.ddl.isolation = false

# parallel parser config
canal.instance.parser.parallel = true
## concurrent thread number, default 60% available processors, suggest not to exceed Runtime.getRuntime().availableProcessors()
#canal.instance.parser.parallelThreadSize = 16
## disruptor ringbuffer size, must be power of 2
canal.instance.parser.parallelBufferSize = 256

# table meta tsdb info
canal.instance.tsdb.enable = true
#canal.instance.tsdb.dir = ${canal.file.data.dir:../conf}/${canal.instance.destination:}
#canal.instance.tsdb.url = jdbc:h2:${canal.instance.tsdb.dir}/h2;CACHE_SIZE=1000;MODE=MYSQL;
canal.instance.tsdb.url = jdbc:mysql://192.168.101.32:3306:3306/pinshang_canal_manager?useUnicode=true&characterEncoding=UTF-8&useSSL=false
canal.instance.tsdb.dbUsername = canal_user
canal.instance.tsdb.dbPassword = CanalManager@888

# dump snapshot interval, default 24 hour
canal.instance.tsdb.snapshot.interval = 24
# purge snapshot expire , default 360 hour(15 days)
canal.instance.tsdb.snapshot.expire = 360

# aliyun ak/sk , support rds/mq
canal.aliyun.accessKey =
canal.aliyun.secretKey =

#################################################
######### 		destinations		#############
#################################################
canal.destinations =
# conf root dir
canal.conf.dir = ../conf
# auto scan instance dir add/remove and start/stop instance
canal.auto.scan = true
canal.auto.scan.interval = 5

#canal.instance.tsdb.spring.xml = classpath:spring/tsdb/h2-tsdb.xml
canal.instance.tsdb.spring.xml = classpath:spring/tsdb/mysql-tsdb.xml

canal.instance.global.mode = manager
canal.instance.global.lazy = false
canal.instance.global.manager.address = ${canal.admin.manager}
#canal.instance.global.spring.xml = classpath:spring/memory-instance.xml
#canal.instance.global.spring.xml = classpath:spring/file-instance.xml

canal.instance.global.spring.xml = classpath:spring/default-instance.xml

##################################################
######### 		     MQ 		     #############
##################################################
canal.mq.servers = 127.0.0.1:6667
canal.mq.retries = 0
canal.mq.batchSize = 16384
canal.mq.maxRequestSize = 1048576
canal.mq.lingerMs = 100
canal.mq.bufferMemory = 33554432
canal.mq.canalBatchSize = 50
canal.mq.canalGetTimeout = 100
canal.mq.flatMessage = true
canal.mq.compressionType = none
canal.mq.acks = all
#canal.mq.properties. =
canal.mq.producerGroup = test
# Set this value to "cloud", if you want open message trace feature in aliyun.
canal.mq.accessChannel = local
# aliyun mq namespace
#canal.mq.namespace =

##################################################
#########     Kafka Kerberos Info    #############
##################################################
canal.mq.kafka.kerberos.enable = false
canal.mq.kafka.kerberos.krb5FilePath = "../conf/kerberos/krb5.conf"
canal.mq.kafka.kerberos.jaasFilePath = "../conf/kerberos/jaas.conf"







~~~



### 安装 server 端



~~~javascript

tar -zxf canal.deployer-1.1.4.tar.gz -C /usr/local

cd /usr/local/canal-server-1.1.4/conf/

vim canal_local.properties 
# register ip
canal.register.ip = 

# canal admin config
canal.admin.manager = 192.168.103.55:8089
canal.admin.port = 11110
canal.admin.user = admin
canal.admin.passwd = 4ACFE3202A5FF5CF467898FC58AAB1D615029441
# admin auto register
canal.admin.register.auto = true
canal.admin.register.cluster = ps_canal



启动: 
	/usr/local/canal-server-1.1.4/bin/startup.sh  local
# 表示用 $base/conf/canal_local.properties 配置文件启动,  而不是/usr/local/canal-deployer/conf/canal.properties
# canal_local_conf=$base/conf/canal_local.properties

此时在ZK中内容如下：
[root@localhost ~]# zkCli.sh -server 192.168.0.135:2181
[zk: 192.168.0.135:2181(CONNECTED) 59] ls /[otter]

~~~

![image-20210909190029694](https://gitee.com/zzjk_01/cloudimages/raw/master/img/image-20210909190029694.png)



新建 instance



~~~
vim instance.propertios



#################################################
## mysql serverId , v1.0.26+ will autoGen
# canal.instance.mysql.slaveId=0

# enable gtid use true/false
canal.instance.gtidon=true

# position info
canal.instance.master.address=192.168.101.122:3306
canal.instance.master.journal.name=
canal.instance.master.position=
canal.instance.master.timestamp=
canal.instance.master.gtid=

# rds oss binlog
canal.instance.rds.accesskey=
canal.instance.rds.secretkey=
canal.instance.rds.instanceId=

# table meta tsdb info
canal.instance.tsdb.enable=true
#canal.instance.tsdb.url=jdbc:mysql://127.0.0.1:3306/canal_tsdb
#canal.instance.tsdb.dbUsername=canal
#canal.instance.tsdb.dbPassword=canal
canal.instance.tsdb.url = jdbc:mysql://192.168.101.32:3306/pinshang_canal_manager?useUnicode=true&characterEncoding=UTF-8&useSSL=false
canal.instance.tsdb.dbUsername = canal_user
canal.instance.tsdb.dbPassword = CanalManager@888

#canal.instance.standby.address =
#canal.instance.standby.journal.name =
#canal.instance.standby.position =
#canal.instance.standby.timestamp =
#canal.instance.standby.gtid=

# username/password
canal.instance.defaultDatabaseName=tj
canal.instance.dbUsername=canalBinlog
canal.instance.dbPassword=CanalBinlog@888
canal.instance.connectionCharset = UTF-8
# enable druid Decrypt database password
canal.instance.enableDruid=false
#canal.instance.pwdPublicKey=MFwwDQYJKoZIhvcNAQEBBQADSwAwSAJBALK4BUxdDltRRE5/zXpVEVPUgunvscYFtEip3pmLlhrWpacX7y7GCMo2/JM6LeHmiiNdH1FWgGCpUfircSwlWKUCAwEAAQ==

# table regex
canal.instance.filter.regex=.*\\..*
# table black regex
canal.instance.filter.black.regex=
# table field filter(format: schema1.tableName1:field1/field2,schema2.tableName2:field1/field2)
#canal.instance.filter.field=test1.t_product:id/subject/keywords,test2.t_company:id/name/contact/ch
# table field black filter(format: schema1.tableName1:field1/field2,schema2.tableName2:field1/field2)
#canal.instance.filter.black.field=test1.t_product:subject/product_image,test2.t_company:id/name/contact/ch

# mq config
canal.mq.topic=example
# dynamic topic route by schema or table regex
#canal.mq.dynamicTopic=mytest1.user,mytest2\\..*,.*\\..*
canal.mq.partition=0
# hash partition config
#canal.mq.partitionsNum=3
#canal.mq.partitionHash=test.table:id^name,.*\\..*
#################################################

~~~





~~~
useradd canal
passwd canal


mkdir canal
chown -R canal. canal


GRANT delete ON *.* TO 'canalBinlog'@'192.168%';


flush privileges;


GRANT insert,update ON *.* TO 'canalBinlog'@'192.168%';


flush privileges;

~~~



![image-20210909194734302](https://gitee.com/zzjk_01/cloudimages/raw/master/img/image-20210909194734302.png)







![image-20210909193922490](https://gitee.com/zzjk_01/cloudimages/raw/master/img/image-20210909193922490.png)





 adapter 绑定 instance 

<img src="https://gitee.com/zzjk_01/cloudimages/raw/master/img/image-20210909201414498.png" alt="image-20210909201414498" style="zoom:200%;" />







![image-20210909203058255](https://gitee.com/zzjk_01/cloudimages/raw/master/img/image-20210909203058255.png)





![image-20210909204008676](https://gitee.com/zzjk_01/cloudimages/raw/master/img/image-20210909204008676.png)



![image-20210909204117352](https://gitee.com/zzjk_01/cloudimages/raw/master/img/image-20210909204117352.png)





![image-20210909204130521](https://gitee.com/zzjk_01/cloudimages/raw/master/img/image-20210909204130521.png)



###################

![image-20210909204645622](https://gitee.com/zzjk_01/cloudimages/raw/master/img/image-20210909204645622.png)



![image-20210909204657728](https://gitee.com/zzjk_01/cloudimages/raw/master/img/image-20210909204657728.png)





![image-20210909204827086](https://gitee.com/zzjk_01/cloudimages/raw/master/img/image-20210909204827086.png)





![image-20210909204921718](https://gitee.com/zzjk_01/cloudimages/raw/master/img/image-20210909204921718.png)



![image-20210909204957847](https://gitee.com/zzjk_01/cloudimages/raw/master/img/image-20210909204957847.png)

![image-20210909205014876](https://gitee.com/zzjk_01/cloudimages/raw/master/img/image-20210909205014876.png)



![image-20210909205151818](https://gitee.com/zzjk_01/cloudimages/raw/master/img/image-20210909205151818.png)



















