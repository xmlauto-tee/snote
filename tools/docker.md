## 安装

```shell
# 安装工具
apt-get -y install apt-transport-https ca-certificates curl software-properties-common

# 安装证书，成功会返回ok
curl -fsSL http://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -

# 修改镜像源
add-apt-repository "deb [arch=amd64] http://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"

# 更新并安装
apt-get update
apt-get -y install docker-ce

# 验证
docker version

# 镜像源修改
# 创建或修改 /etc/docker/daemon.json 文件，修改为如下形式
{
    "registry-mirrors": ["http://hub-mirror.c.163.com"]
}
# 然后通过命令systemctl restart docker.service或service docker restart重启docker
# 其中registry-mirrors表示设置镜像地址，我们可以配置insecure-registries设置私有仓库地址
# 注意多个参数，需要有逗号分隔
```

## 基本使用

```shell
# 查看本地images
docker images

# 查看开启的容器
docker ps

# 查看所有容器，包括退出的
docker ps -a

# 交互式容器
docker run -i -t ubuntu /bin/bash

# 后台运行
docker run -itd ubuntu /bin/bash

# 指定运行的名字，后续涉及到容器的ID的都可以使用这个name来替换
docker run -itd --name linlei ubuntu /bin/bash

# 随系统启动
docker run --restart always

# 指定容器hostname
docker exec -itd --hostname linlei ubuntu /bin/bash

# 以root权限进入docker
docker exec -it -u root ubuntu /bin/bash

# 创建或修改环境变量，使用-e参数
docker run -itd --name mysql-test -p 3306:3306 -e MYSQL_ROOT_PASSWORD=123456 mysql

# 查看log
docker logs [ID]

# 停止容器
docker stop [ID]

# stop后的容器，如果没有被删除，可以通过start启动
docker start [ID]/[NAME]

# 进入容器
docker exec -it [ID] /bin/bash

# 搜索镜像
docker search ubuntu

# 拉取镜像
docker pull ubuntu

# 删除镜像
docker rmi ubuntu

# 删除容器
docker rm [container]

# 制作镜像
docker commit -m "commit msg" -a "author" [ID] main:tag

# 其中ID是容器ID，main是容器名，tag可以指定版本等信息，比如
docker commit -m="update" -a="test" e218edb10161 runoob/ubuntu:v2

# 还可以通过下面的方式对镜像新增
docker tag [ID] main:tag
# 注意这里的ID是镜像ID，docker commit是容器ID
```

## 主机与宿主机关联

### 挂载宿主机目录

docker run -it -v /usr/codeFile:/opt/code test /bin/bash  其中-v指定挂载目录，冒号前面是宿主机的目录，冒号后面是docker里面的目录

### 映射网络端口

docker run -it -p 5000:5000 test /bin/bash  其中-p 前者是宿主机的端口号，后者是容器的端口号

## 创建私有仓库

```shell
docker pull registry
docker run -dti --name=registry -v x:x -p 5000:5000 registry
vim /etc/docker/daemon.json
# 修改为内容如下
{
    "registry-mirrors": ["http://hub-mirror.c.163.com"],
    "insecure-registries": ["192.168.46.50:5000"]
}
service docker restart
docker start registry

# 这里先拉取一个小的镜像来测试
docker pull busybox

# 标记一个需要提交的镜像
root@docker:/etc/docker# docker tag busybox 192.168.46.50:5000/busybox

# 查看一下
root@docker:/etc/docker# docker images
REPOSITORY                   TAG                 IMAGE ID            CREATED             SIZE
192.168.46.50:5000/busybox   latest              83aa35aa1c79        4 weeks ago         1.22MB

# 上传
root@docker:/etc/docker# docker push 192.168.46.50:5000/busybox
The push refers to repository [192.168.46.50:5000/busybox]
a6d503001157: Pushed 
latest: digest: sha256:afe605d272837ce1732f390966166c2afff5391208ddd57de10942748694049d size: 527

# 拉取镜像
docker pull 192.168.46.50:5000/busybox

# 注意私有仓库，必须给定IP地址，否则会默认push到官方的地址上去
# 注意如果拉取的时候报错Error response from daemon: Get https://192.168.46.123:5000/v1/_ping: http: server gave HTTP response to HTTPS client可以在客户端加入如下配置 "insecure-registries": ["192.168.46.123:5000"]
# registry的仓库目录为/var/lib/registry，可以挂载到本地
```

### 查看私有仓库

```shell
root@docker:~# curl -XGET http://192.168.46.123:5000/v2/_catalog
{"repositories":["gcov","ubuntu"]}
root@docker:~# curl -XGET http://192.168.46.123:5000/v2/gcov/tags/list
{"name":"gcov","tags":["latest"]}
root@docker:~# curl -XGET http://192.168.46.123:5000/v2/ubuntu/tags/list
{"name":"ubuntu","tags":["1404"]}
root@docker:~#
```

### 进入私有仓库

```shell
docker exec -it [ID] sh 
# 配置文件在 /etc/docker/registry/config.yml下
# 如果要配置允许删除私有镜像，需要修改配置

storage:
  delete:
    enabled: true

# 然后通过docker restart [ID] 重启容器
```

### 删除私有仓库镜像

```shell
curl --header "Accept:application/vnd.docker.distribution.manifest.v2+json" -I -XGET 192.168.46.123:5000/v2/gcov/manifests/latest   
HTTP/1.1 200 OK
Content-Length: 2631
Content-Type: application/vnd.docker.distribution.manifest.v2+json
Docker-Content-Digest: sha256:280b9f0e210fab23a31a98729119b51aa512040a2d0adc04acc06feea64e2b27
Docker-Distribution-Api-Version: registry/2.0
Etag: "sha256:280b9f0e210fab23a31a98729119b51aa512040a2d0adc04acc06feea64e2b27"
X-Content-Type-Options: nosniff
Date: Wed, 15 Apr 2020 06:28:08 GMT

curl -I -X DELETE http://192.168.46.123:5000/v2/gcov/manifests/sha256:280b9f0e210fab23a31a98729119b51aa512040a2d0adc04acc06feea64e2b27
# 这里的sha是Docker-Content-Digest对应的

# 进入仓库目录，如果挂载到宿主机，直接进入宿主机
docker/registry/v2/repositories

# 删除需要删除的镜像的目录即可
# 删除完成后，执行下面的命令清理
docker exec -it registry sh -c 'registry garbage-collect /etc/docker/registry/config.yml'
```

### 图形化管理

```shell
docker search portainer
docker pull portainer/portainer
docker run -d -p 9000:9000 --restart=always -v /var/run/docker.sock:/var/run/docker.sock --name portainer portainer/portainer
# 然后输入192.168.46.123:9000访问
```

### 举例

```shell
# registry运行
docker run -dti --name=registry --restart always -v /root/docker_mount/registry:/var/lib/registry -v/root/docker_mount/config/config.yml:/etc/docker/registry/config.yml -p 5000:5000 registry

# flask运行
docker run -itd --hostname=flask --name=flask --restart always -v /root/docker_mount/flask:/app -v /root/docker_mount/files/flask_startup.sh:/root/startup.sh -p 5001:5000 192.168.46.123:5000/flask:latest bash -c "/root/startup.sh"

# flask内部启动脚本
root@docker:~/docker_mount/files# cat flask_startup.sh 
#!/bin/bash
service cron start
set -e
#source /root/anaconda3/etc/profile.d/conda.sh
source /etc/profile.d/conda.sh
conda activate
export FLASK_ENV=development
cd /app
python tools/check_env_owner.py &
flask run --host=0.0.0.0

# registry config配置
root@docker:~/docker_mount# cat config/config.yml 
version: 0.1
log:
  fields:
    service: registry
storage:
  delete:
    enabled: true
  cache:
    blobdescriptor: inmemory
  filesystem:
    rootdirectory: /var/lib/registry
http:
  addr: :5000
  headers:
    X-Content-Type-Options: [nosniff]
health:
  storagedriver:
    enabled: true
    interval: 10s
    threshold: 3
```

## 其它命令

### image重命名

```shell
root@docker:~# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
gcov                v1                  ae3e147b4d8b        3 hours ago         589MB
ubuntu              1802                ace29c6c05da        3 weeks ago         153MB
ubuntu              latest              72300a873c2c        6 weeks ago         64.2MB
nodesource/trusty   latest              28c48c5c54bd        3 years ago         552MB
root@docker:~# docker tag 28c48c5c54bd trusty:v1
root@docker:~# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
gcov                v1                  ae3e147b4d8b        3 hours ago         589MB
ubuntu              1802                ace29c6c05da        3 weeks ago         153MB
ubuntu              latest              72300a873c2c        6 weeks ago         64.2MB
trusty              v1                  28c48c5c54bd        3 years ago         552MB
nodesource/trusty   latest              28c48c5c54bd        3 years ago         552MB

# 注意新建的和原始的image的image id一样，此时不能通过id来删除了，只能通过名字来删除
root@docker:~# docker rmi nodesource/trusty
```

### 查看容器启动命令

docker inspect [容器ID或名字] 
容器内部可以通过ps -ef 的ID为1的就是

### paused

docker pause [ID] 
docker unpause [ID]

### 批量删除容器

批量删除退出状态的容器:  
docker rm `docker ps -a|grep -E "Exited|Created"|awk '{print $1}'`

### push镜像到docker hub

docker login

### 通过SSH访问docker容器

```shell
# docker启动的时候，挂载-p xxx:22，
# xxx表示外部端口，22是ssh默认登录端口，内部映射
apt-get install openssh-server

vim /etc/ssh/sshd_config

# 修改PermitRootLogin 为yes
service ssh restart

# 修改或创建root用户密码
sudo passwd root
```

## 好用的docker容器

### mysql

```shell
# 搜索和下载
# docker hub 官方地址: https://hub.docker.com/_/mysql
docker search mysql
# docker pull mysql
docker pull mysql:latest

# 启动
docker run -itd --name mysql-test -p 3306:3306 -e MYSQL_ROOT_PASSWORD=123456 mysql

# 进入
root@docker:~# docker exec -it mysql-test /bin/bash
root@bd7a22d25773:/data# mysql -h localhost -u root -p
mysql> show tables;
ERROR 1046 (3D000): No database selected
mysql> 

# 配置
docker run -itd --name mysql-test -v /my/custom:/etc/mysql/conf.d -e MYSQL_ROOT_PASSWORD=123456 mysql

# 使用
# 数据库位置在 /var/lib/mysql，所以可以通过-v的方式进行挂载，让数据持久化
root@docker:~# docker exec -it mysql-test /bin/bash
root@bd7a22d25773:/data# mysql -h localhost -u root -p
mysql> show tables;
ERROR 1046 (3D000): No database selected
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
4 rows in set (0.00 sec)

mysql> create database test;
Query OK, 1 row affected (0.00 sec)

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| test               |
+--------------------+
5 rows in set (0.01 sec)
mysql> use test;
Database changed
mysql> status;
--------------
mysql  Ver 8.0.23 for Linux on x86_64 (MySQL Community Server - GPL)

Connection id:          13
Current database:       test
Current user:           root@localhost
SSL:                    Not in use
Current pager:          stdout
Using outfile:          ''
Using delimiter:        ;
Server version:         8.0.23 MySQL Community Server - GPL
Protocol version:       10
Connection:             Localhost via UNIX socket
Server characterset:    utf8mb4
Db     characterset:    utf8mb4
Client characterset:    latin1
Conn.  characterset:    latin1
UNIX socket:            /var/run/mysqld/mysqld.sock
Binary data as:         Hexadecimal
Uptime:                 22 min 36 sec

Threads: 2  Questions: 24  Slow queries: 0  Opens: 157  Flush tables: 3  Open tables: 76  Queries per second avg: 0.017
--------------

mysql> create table test (id int, name varchar(20));
Query OK, 0 rows affected (0.01 sec)

mysql> show tables;
+----------------+
| Tables_in_test |
+----------------+
| test           |
+----------------+
1 row in set (0.00 sec)

mysql> select * from test;
Empty set (0.01 sec)
mysql> insert into test values (1, 'a');
Query OK, 1 row affected (0.00 sec)

mysql> select * from test;
+------+------+
| id   | name |
+------+------+
|    1 | a    |
+------+------+
1 row in set (0.00 sec)

mysql>
```

#### navicate连接报错

错误信息: Authentication plugin caching_sha2_password  
错误解决: 

```shell
# 进入容器
docker exec -it mysql-test /bin/bash

# 进入mysql
mysql -h127.0.0.1 -uroot -p

# 输入命令，修改root密码
# 注意这里的IP地址可能先前没有配置，参考下面【navicate权限问题】章节
ALTER USER 'root'@'192.168.46.20' IDENTIFIED WITH mysql_native_password BY '123456';
flush privileges;
```

#### navicate权限问题

错误信息: Access denied for user 'root'  
错误解决: 

```shell
# 进入容器
docker exec -it mysql-test /bin/bash

# 进入mysql
mysql -h127.0.0.1 -uroot -p

# 查看权限
show databases;
use mysql;
show tables;
select * from user;

# 授权指定IP访问
GRANT ALL PRIVILEGES ON *.* TO 'root'@'10.10.1.35' IDENTIFIED BY 'youpassword' WITH GRANT OPTION;
# 授权所有IP访问
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY 'youpassword' WITH GRANT OPTION;
# 刷新权限
flush privileges;

# 新版权限问题
create user 'root'@'192.168.46.20' identified by '123456';
grant ALL PRIVILEGES on *.* to 'root'@'192.168.46.20';
flush privileges;
```

### redis

```shell
# 搜索和下载
# docker hub 官方地址: https://hub.docker.com/_/redis
docker search redis
# docker pull redis
docker pull redis:latest

# 启动
docker run --name redis-test -itd redis
docker run --name redis-test -p 6379:6379 -itd redis # 暴露端口，可以通过宿主机访问

# 进入
root@docker:~# docker exec -it redis-test redis-cli  # 方式一
127.0.0.1:6379> exit
root@docker:~# docker exec -it redis-test /bin/bash  # 方式二
root@bd7a22d25773:/data# redis-cli
127.0.0.1:6379> 
```
