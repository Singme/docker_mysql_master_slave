### 基于docker的mysql主从复制搭建 
#### 一主一从
首先拉取docker mysql的镜像，这里使用的版本是latest  
然后使用此镜像启动容器，这里需要分别启动主从两个容器  
* Master_1:  
分别设置对外映射端口，容器的名称，用户密码参数  
> docker run -p 3309:3306 --name mysql_master_1 -e MYSQL_ROOT_PASSWORD=123456 -d mysql:latest  
* Slave_1:  
> docker run -p 3310:3306 --name mysql_slave_1 -e MYSQL_ROOT_PASSWORD=123456 -d mysql:latest  
Master_1对外映射的端口是3309，Slave_1对外映射的端口是3310。因为docker容器是相互独立的，每个容器有其独立的ip,所以不同容器使用相同的端口是不会有冲突的。  
对于容器的独立ip,可以通过：  
> docker inspect --format='{{.NetworkSettings.IPAddress}}' mysql_master_1  
*配置Master*  
通过```docker exec -it mysql_master_1 /bin/bash```命令进入到Master容器内部，切换到/etc/mysql目录下，然后对my.cnf进行编辑。因为docker容器内部没有安装vim，所以需要先通过apt-get update命令，之后使用apt-get install vim命令来安装vim。  
在my.cnf中添加如下配置：  
```  
# 同一局域网内注意唯一
server-id=100
# 开启二进制日志功能，可以随便取名（关键）
log-bin=mysql-bin
```  
配置完成之后，需要重启mysql服务使配置生效。可以通过重新启动容器 docker restart mysql_master_1来生效服务配置。  
下一步在Master数据库创建数据同步用户，先授权slave用户REPLICATION SLAVE权限和REPLICATION CLIENT权限，用于在主从库之间同步数据。  
> create user 'slave'@'%' identified by '123456';  
> grant replication slave,replication client on *.* to 'slave'@'%';  
*配置Slave*  
在Slave配置文件my.cnf中添加如下配置：  
```  
# 设置server_id,注意要唯一
server-id=101
# 开启二进制日志功能，以备Slave作为其他Slave的Master时使用
log-bin=mysql-slave-bin
# relay_log配置中继日志
relay_log=edu-mysql-relay-bin
```  
配置后的slave也需要重启mysql服务和docker容器。  
*链接Master和Slave*  
在Master容器进入mysql，执行 show master status;  
获取File和Position字段的值，在后面的操作完成之前，需要保证Master库不能做任何操作，否则将会引起状态变化，File和Position字段的值改变。  
接下来，在Slave容器中进入mysql，然后执行：  
> change master to master_host='172.17.0.2',master_port=3306,master_user='slave',master_password='123456',master_log_file='mysql-bin.000001',master_log_pos=907,master_connect_retry=30;  
命令说明：  
master_host:指Master容器的独立ip  
master_port:指Master容器的端口号  
master_user:指用于数据同步的用户  
master_password:指用于同步的用户密码  
master_log_file:指Slave从哪个日志文件开始复制数据，即刚刚提到的File字段的值  
master_log_pos:指从哪个Position开始读，即提到的Position字段的值  
master_connect_retry:如果连接失败，重试的时间间隔，单位是秒，默认是60秒  
正常情况下，在没有开启主从复制过程，SlaveORunning和SlaveSQLRunning都是为No，需要使用start slave来开启，然后再次通过show slave status \G; 来查询主从同步状态。  
最后，可通过在Master库中先建数据库，来检查Slave库是否存在此数据库，根据数据库是否存来测试配置的成功。  
#### 一主多从  
一主多从跟一主一从中，Slave库配置是一样的，就是多开些Slave容器，然后修改这些Slave库的服务器配置，配置从上设置，即可完成一主多从。  

