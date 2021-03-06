---
title: Hadoop2.x 工作台构建(Hadoop圈应用独立部署)
date: 2018-05-03 
tags:
- Hadoop
categories:
- BigData
---
之前hadoop圈的应用（如hive、sqoop、azkaban等）都在master节点部署，一次性多个应用同时启动，耗尽内存后Linux随机删掉进程，影响集群和应用的稳定性。
<!--more-->

# 整体说明
解决思路是在联通的网络环境下，取一台单独的服务器，不作为集群一部分，仅部署各种应用，应用使用jvm控制内存，更进一步使用独立用户搭配cgroup控制资源的使用，维持整体稳定。这台单独的服务器下称为“工作台”。
# 试验环境
workshop原版: 10.41.236.56
## 快速拷贝
### shell脚本
这是从A3的209拷贝任务脚本到"母体"(10.41.236.56), 在母体上执行命令
```bash
export SRC_HOST=10.41.236.209
scp -r hadoop@$SRC_HOST:/home/hadoop/soft/file /home/hadoop/soft
```
### 自定义jar包
这是从A3的209拷贝任务脚本到"母体"(10.41.236.56), 在母体上执行命令
```bash
export SRC_HOST=10.41.236.209
scp hadoop@$SRC_HOST:/home/hadoop/soft/*.jar /home/hadoop/soft/
```
### 环境变量与软件
这命令是从"母体"(10.41.236.56)拷贝到"执行者"executor上, 在执行者的机器上执行命令
```bash
export SRC_HOST=10.41.236.56
sudo scp hadoop@$SRC_HOST:/etc/profile.d/xcube.sh /etc/profile.d/xcube.sh
source /etc/profile.d/xcube.sh
scp -r hadoop@$SRC_HOST:/home/hadoop/soft /home/hadoop/soft

```



# 整体规划
## 环境变量
```bash
sudo vim /etc/profile.d/xcube.sh  # 添加以下内容
# JAVA
export JAVA_HOME=/home/hadoop/soft/jdk 
export JRE_HOME=$JAVA_HOME/jre 
export CLASSPATH=$JAVA_HOME/lib:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar:$JRE_HOME/lib:$CLASSPATH
export PATH=$PATH:$JAVA_HOME/bin:$JRE_HOME/bin 
# Hadoop
export HADOOP_HOME=/home/hadoop/soft/hadoop 
export HADOOP_DEV_HOME=${HADOOP_HOME}
export HADOOP_MAPARED_HOME=${HADOOP_HOME}  
export HADOOP_COMMON_HOME=${HADOOP_HOME}  
export HADOOP_HDFS_HOME=${HADOOP_HOME}  
export YARN_HOME=${HADOOP_HOME}  
export HADOOP_CONF_DIR=${HADOOP_HOME}/etc/hadoop
export YARN_CONF_DIR=${HADOOP_HOME}/etc/hadoop
export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
# Hive
export HIVE_HOME=/home/hadoop/soft/hive
alias bee='beeline -n hadoop -u jdbc:hive2://10.41.236.209:10000'
export PATH=$PATH:$HIVE_HOME/bin
#Sqoop
export SQOOP_HOME=/home/hadoop/soft/sqoop
export PATH=$PATH:$SQOOP_HOME/bin
#Hbase
#export HBASE_HOME=/home/hadoop/soft/hbase
#Scala
export SCALA_HOME=/home/hadoop/soft/scala
export PATH=$PATH:$SCALA_HOME/bin
#Spark
export SPARK_HOME=/home/hadoop/soft/spark
export PATH=$PATH:$SPARK_HOME/bin


# 账号密码jdbc
export azkaban_username=azkaban
export azkaban_password=azkaban%941

export kylin_username=ADMIN
export kylin_password=KYLIN%258

export tr_url=jdbc:oracle:thin:@//10.41.37.47:1521/dgpffin.huawei.com
export tr_username=uniccs
export tr_password=uniccs123

export xcube_url=jdbc:oracle:thin:@//a3xcubeoracle01.beta.hic.cloud:1521/a3xcube_srv
export xcube_username=xcube
export xcube_password=huawei123

export xcube_url_kaifa=jdbc:oracle:thin:@//szftdscan02.huawei.com:1521/xcubedt_srv
export xcube_username_kaifa=xcube
export xcube_password_kaifa=huawei123

export cpi_url=jdbc:oracle:thin:@//10.98.65.205:1521/nkuatr.huawei.com
export cpi_username=uniconfigbase_query
export cpi_password=huawei123

export ccs_url=jdbc:oracle:thin:@//nkdb370371-cls.huawei.com:1521/nka3fin.huawei.com
export ccs_username=uniccs
export ccs_password=xhrg#3ddcr

export rcm_url=jdbc:oracle:thin:@//nkgtsp16566-cls.huawei.com:1521/cepd
export rcm_username=fcquery
export rcm_password=huawei123

export ccm_url=jdbc:oracle:thin:@//nkgtsp16566-cls.huawei.com:1521/cepd
export ccm_username=ccm
export ccm_password=d12aafcddac156c

export tr_url=jdbc:oracle:thin:@//nkdb370371-cls.huawei.com:1521/nka3fin.huawei.com
export tr_username=starplate
export tr_password=test
```

## JDK1.8
```bash
mkdir -p /home/hadoop/soft
cd /home/hadoop/soft
# mv jdk.tar.gz
tar -zvxf jdk-8u151-linux-x64.tar.gz
mv jdk1.8.0_151 jdk
sudo vim /etc/profile
export JAVA_HOME=/home/hadoop/soft/jdk
export JRE_HOME=$JAVA_HOME/jre
export PATH=$JAVA_HOME/bin:$JRE_HOME/bin:$PATH
export CLASSPATH=$JAVA_HOME/lib:$CLASSPATH
export PATH

source /etc/profile
```
## hadoop
hadoop圈的应用大都需要引用hdfs、yarn、mapreduce相关jar包才能启动，所以首先要在工作台上配置hadoop，使其能够访问集群的hdfs文件系统，能提交mapreduce任务。


### 解压、环境变量
```bash
tar -zxvf hadoop-2.7.2.tar.gz
mv hadoop-2.7.2 hadoop
sudo vim /etc/profile
#Hadoop&YARN
export HADOOP_DEV_HOME=/home/hadoop/soft/hadoop
export PATH=$PATH:$HADOOP_DEV_HOME/bin
export PATH=$PATH:$HADOOP_DEV_HOME/sbin
export HADOOP_HOME=/home/hadoop/soft/hadoop
export HADOOP_MAPARED_HOME=${HADOOP_DEV_HOME}
export HADOOP_COMMON_HOME=${HADOOP_DEV_HOME}
export HADOOP_HDFS_HOME=${HADOOP_DEV_HOME}
export YARN_HOME=${HADOOP_DEV_HOME}
export HADOOP_CONF_DIR=${HADOOP_DEV_HOME}/etc/hadoop
source /etc/profile
```
### 目录
```bash
mkdir -p /data01/huawei/BigData/jntmp/journal
mkdir -p /data01/huawei/BigData/hadoop/name
mkdir -p /data01/huawei/BigData/hadoop/data
sudo chown -R hadoop:root /data01

```
### core-site.xml

```xml
<configuration>
<property>
    <name>fs.defaultFS</name>
    <value>hdfs://ns1</value>
</property>
<property>
    <name>hadoop.tmp.dir</name>
    <value>/home/hadoop/soft/hadoop/tmp</value>
</property>
</configuration>
```
### hdfs-site.xml
dfs.client.failover.proxy.provider.ns1这一项一定要配置，ns1要跟core-site.xml和dfs.nameservices对应
```xml
<configuration>
    <property>
        <name>dfs.nameservices</name>
        <value>ns1</value>
    </property>
    <property>
        <name>dfs.ha.namenodes.ns1</name>
        <value>nn1,nn2</value>
    </property>
    <property>
        <name>dfs.namenode.rpc-address.ns1.nn1</name>
        <value>10.41.236.209:9000</value>
    </property>
    <property>
        <name>dfs.namenode.rpc-address.ns1.nn2</name>
        <value>10.41.236.115:9000</value>
    </property>
    <property>
        <name>dfs.client.failover.proxy.provider.ns1</name>
        <value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value>
    </property>
</configuration>
```
### yarn-site.xml
```xml
<configuration>
    <property>
        <name>yarn.resourcemanager.ha.enabled</name>
        <value>true</value>
    </property>
    <property>
        <name>yarn.resourcemanager.cluster-id</name>
        <value>ns1</value>
    </property>
    <property>
        <name>yarn.resourcemanager.ha.rm-ids</name>
        <value>rm1,rm2</value>
    </property>
    <property>
        <name>yarn.resourcemanager.hostname.rm1</name>
        <value>10.41.236.209</value>
    </property>
    <property>
        <name>yarn.resourcemanager.hostname.rm2</name>
        <value>10.41.236.115</value>
    </property>
</configuration>
```
### mapred-site.xml
```xml
<configuration>
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>
    <property>
        <name>mapreduce.jobhistory.address</name>
        <value>10.41.236.209:10020</value>
    </property>
    <property>
        <name>yarn.app.mapreduce.am.staging-dir</name>
        <value>/user</value>
    </property>
</configuration>
```
### 验证
>hadoop   jar ~/soft/hadoop/share/hadoop/mapreduce/hadoop-mapreduce-examples-2.7.2.jar pi 10 100000000

## Hive

### 解压、环境变量
```bash
tar -zvxf apache-hive-2.3.2-bin.tar.gz
mv apache-hive-2.3.2-bin hive
sudo vim /etc/profile
#HIVE
export HIVE_HOME=/home/hadoop/soft/hive
source /etc/profile
```
### 目录
```bash
mkdir -p /data01/huawei/BigData/hive/tmp
sudo chown -R hadoop:root /data01

```
### hive-site.xml
最小化修改.
1. 长sql有中文名, 导致job name过长, 在此限制一下, 避免在结束时`Job status not available`的问题
2. 开发sql有时候不是非常严格规范, 需要放宽:hive.strict.checks检查
```xml
<configuration>
<property>
    <name>fs.defaultFS</name>
    <value>hdfs://ns1</value>
</property>
<property>
    <name>hadoop.tmp.dir</name>
    <value>/home/hadoop/soft/hadoop/tmp</value>
</property>
<property>
    <name>hive.jobname.length</name>
    <value>10</value>
    <description>max jobname length</description>
  </property>
  <property>
    <name>hive.strict.checks.cartesian.product</name>
    <value>false</value>
    <description>
      Enabling strict Cartesian join checks disallows the following:
        Cartesian product (cross join).
    </description>
</configuration>
```
### beeline
命令行工具测试是否可用
>beeline -n hadoop -u jdbc:hive2://10.41.236.209:10000

## sqoop
1. 解压二进制包到/home/hadoop/sqoop, 同时确认为SQOOP_HOME
2. 拷贝jdbc依赖的ojdbc6.jar到$SQOOP_HOME/lib/下
3. 修改metastore的地址, 预防在~/.sqoop下占用太多空间, 在xml文件<configuration>节内添加(或解注释)以下内容
> vim $SQOOP_HOME/conf/sqoop-site.xml
```xml
<property>
    <name>sqoop.metastore.server.location</name>
    <value>/data01/soft/sqoop-metastore/shared.db</value>
    <description>Path to the shared metastore database files. If this is not set, it will be placed in ~/.sqoop/.
    </description>
</property>
<property>
    <name>sqoop.metastore.server.port</name>
    <value>16000</value>
    <description>Port that this metastore should listen on.
    </description>
</property>
```
4. 因为没有配置`$HBASE_HOME/$HCAT_HOME/$ACCUMULO_HOME/$ZOOKEEPER_HOME`这些路径, 所以会`Warining`提示不能导入Hcatalog/Accumulo的任务. 忽略之, 直到需要了再配置

## spark
### 配置
`spark-env.sh`如果环境变量已经有, 改配置文件基本不需要改
```bash
vim ~/soft/spark/conf/spark-env.sh
# 当系统的环境变量配置好后,不需要下方的配置, SPARK_DRIVER_MEMORY属于优化配置, 可省略
#export SPARK_MASTER_WEBUI_PORT=8081 # default:8080
SPARK_DRIVER_MEMORY=4G
```
`spark-defaults.conf`以下配置按照NodeManager是12C12G计算
```bash
vim ~/soft/spark/conf/spark-defaults.conf
spark.master                        yarn
spark.submit.deployMode				cluster
spark.home                          /home/hadoop/soft/spark
spark.eventLog.enabled              true
spark.eventLog.dir                  hdfs://ns1/spark/spark-log
spark.serializer                    org.apache.spark.serializer.KryoSerializer
# hive on spark 建议5/6/7
spark.executor.cores                6
# 按6*12G/12vcores计算
spark.executor.memory               5222m
# 28个计算节点*每个节点2个executor
spark.executor.instances            56
# 所有core的两三倍：56*6*3=1008
spark.default.parallelism           1000
# 15% * spark.executor.memory
spark.yarn.executor.memoryOverhead  921m
spark.driver.memory                 4g
spark.yarn.driver.memoryOverhead    400m
spark.yarn.jars                     hdfs://ns1/spark/jars/*.jar
```
### 验证
```
./bin/spark-submit --class org.apache.spark.examples.SparkPi ./examples/jars/spark-examples_2.11-2.2.0.jar 10
```

## Azkaban-3.50.2
github编译的3.50.2的二进制包:
`https://szxsvn02-ex:3690/svn/CP_CCM_SVN/UniSTAR Common/10.Project Team/15.xCube/14 Hadoop环境搭建/package&config/package/azkaban-3.50.2`
### 说明
web工程是UI界面, 用于查看任务等; exec-server工程是执行者的工程, 用于执行任务. Azkaban从3.0.0开始, 支持多个执行器, 本次采用**multiple executor mode**多执行器的部署形式.
也即Azkaban-web工程只需要部署一个, Azkaban-executor工程可能需要部署多台.
### 备份数据库
有备无患的步骤, 其中参数`hex-blob`不可省略
>mysqldump -h10.41.236.209 -uhive -phuawei123 azkaban --hex-blob > a3.sql
### 升级数据库
生产和测试环境使用的2.5.0版本, 需要执行`azkaban-sql-3.0.0.zip`中的的4个脚本, 升级到3.0时代, 然后再执行`azkaban-db-3.50.2.zip`中的2个upgrade脚本.
```
create.executors.sql
update.active_executing_flows.3.0.sql
update.execution_flows.3.0.sql
create.executor_events.sql
```
偷懒可以粘贴这一段
```sql
CREATE TABLE executors (
  id INT NOT NULL PRIMARY KEY AUTO_INCREMENT,
  host VARCHAR(64) NOT NULL,
  port INT NOT NULL,
  active BOOLEAN DEFAULT true,
  UNIQUE (host, port),
  UNIQUE INDEX executor_id (id)
);
CREATE INDEX executor_connection ON executors(host, port);

ALTER TABLE active_executing_flows DROP COLUMN host;
ALTER TABLE active_executing_flows DROP COLUMN port;

ALTER TABLE execution_flows ADD COLUMN executor_id INT DEFAULT NULL;
CREATE INDEX executor_id ON execution_flows(executor_id);

CREATE TABLE executor_events (
  executor_id INT NOT NULL,
  event_type TINYINT NOT NULL,
  event_time DATETIME NOT NULL,
  username VARCHAR(64),
  message VARCHAR(512)
);
CREATE INDEX executor_log ON executor_events(executor_id, event_time);
---3.20.0-->3.22.0
ALTER TABLE project_versions ADD resource_id VARCHAR(512);
ALTER DATABASE azkaban CHARACTER SET utf8 COLLATE utf8_general_ci;
ALTER TABLE projects MODIFY name VARCHAR(64) CHARACTER SET utf8 COLLATE utf8_general_ci;
```
如果出错, 请看: [参考文档](https://azkaban.github.io/azkaban/docs/latest/#upgrade-27)
### Azkaban-web-server工程
#### 下载解压
```bash
mkdir -p /home/hadoop/soft/azkaban
cd /home/hadoop/soft/azkaban
unzip azkaban-web-server.zip
rm -f azkaban-web-server.zip
```
#### 修改配置
10.41.236.56上已经改好适配A3环境, 如果是改环境, **mysql**/邮箱地址/keystore这三部分
```bash
# 改conf配置, 主要是mysql/邮箱地址/keystore
vim azkaban-web-server/conf/azkaban.properties
```
#### keystore
只有一开始keystore password需要输入两次`azkaban`, 后面一路回车, 注意到correct?确认的时候, 需要输入一个`y`, 后面也是回车, 直到最后的warning.
```bash
cd /home/hadoop/soft/azkaban/azkaban-web-server
keytool -keystore keystore -alias jetty -genkey -keyalg RSA
Enter keystore password:azkaban
Re-enter new password:azkaban
What is your first and last name?
  [Unknown]:
What is the name of your organizational unit?
  [Unknown]:
What is the name of your organization?
  [Unknown]:
What is the name of your City or Locality?
  [Unknown]:
What is the name of your State or Province?
  [Unknown]:
What is the two-letter country code for this unit?
  [Unknown]:
Is CN=Unknown, OU=Unknown, O=Unknown, L=Unknown, ST=Unknown, C=Unknown correct?
  [no]:  y

Enter key password for <jetty>
        (RETURN if same as keystore password):

Warning:
The JKS keystore uses a proprietary format. It is recommended to migrate to PKCS12 which is an industry standard format using "keytool -importkeystore -srckeystore keystore -destkeystore keystore -deststoretype pkcs12".
```
#### 启动与访问验证
启动
```bash
cd /home/hadoop/soft/azkaban/azkaban-web-server
#sh bin/azkaban-web-start.sh
sh bin/start-web.sh
```
访问
>https://10.41.236.56:8443

停止
>sh /home/hadoop/soft/azkaban/azkaban-web-server/bin/azkaban-web-shutdown.sh
### Azkaban-exec-server工程
由于使用的是多个执行器的部署形式, **每个**执行器也应该部署hadoop等软件, 进行配置步骤. executors之间可以考虑拷贝环境变量文件, 拷贝软件目录. 
#### 下载解压
```bash
mkdir -p /home/hadoop/soft/azkaban
cd /home/hadoop/soft/azkaban
unzip azkaban-exec-server.zip
rm -f azkaban-exec-server.zip
```
#### 修改配置
10.41.236.56和10.41.236.44已经改好适配A3环境, 如果是改环境, **mysql**/邮箱地址/keystore这三部分
```bash
# 改conf配置, 主要是mysql/邮箱地址/keystore
vim azkaban-exec-server/conf/azkaban.properties
```

#### 启动与停止
启动
```bash
cd /home/hadoop/soft/azkaban/azkaban-exec-server
#sh bin/azkaban-executor-start.sh
sh bin/start-exec.sh
```
停止
>/home/hadoop/soft/azkaban/azkaban-exec-server/bin/azkaban-executor-shutdown.sh



