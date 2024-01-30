# Docker基础命令及原理

  
  
&lt;!--more--&gt;  
# Docker的常用命令  
## 帮助命令  
```shell  
docker version     #显示 docker 的版本  
docker info        #显示 docker 的系统信息：包括镜像和容器的数量  
docker 命令 --help  #帮助命令  
```  
## 镜像命令  
`docker images` 查看所有镜像  
```shell  
f10wers13eicheng@MacBookPro [12时12分59秒] [/Library/Java/JavaVirtualMachines]   
-&gt; % docker images  
REPOSITORY    TAG       IMAGE ID       CREATED       SIZE  
hello-world   latest    feb5d9fea6a5   2 years ago   13.3kB  
  
# 解释  
REPOSITORY 镜像的仓库源  
TAG        镜像的标签  
IMAGE ID   镜像的id  
CREATED    镜像的创建时间  
SIZE       镜像的大小  
  
-a # 列出所有镜像  
-q # 显示镜像id  
```  
`docker search 镜像名` 搜索镜像  
```shell  
f10wers13eicheng@MacBookPro [12时25分22秒] [/Library/Java/JavaVirtualMachines]   
-&gt; % docker search mysql  
NAME                            DESCRIPTION                                      STARS     OFFICIAL   AUTOMATED  
mysql                           MySQL is a widely used, open-source relation…   14805     [OK]         
mariadb                         MariaDB Server is a high performing open sou…   5653      [OK]         
percona                         Percona Server is a fork of the MySQL relati…   623       [OK]         
phpmyadmin                      phpMyAdmin - A web interface for MySQL and M…   934       [OK]         
bitnami/mysql                   Bitnami MySQL Docker Image                       106                  [OK]  
bitnami/mysqld-exporter                                                          6                      
cimg/mysql                                                                       3                      
ubuntu/mysql                    MySQL open source fast, stable, multi-thread…   58                     
rapidfort/mysql                 RapidFort optimized, hardened image for MySQL    25                     
rapidfort/mysql8-ib             RapidFort optimized, hardened image for MySQ…   9                      
google/mysql                    MySQL server for Google Compute Engine           25                   [OK]  
rapidfort/mysql-official        RapidFort optimized, hardened image for MySQ…   9                      
elestio/mysql                   Mysql, verified and packaged by Elestio          0                      
hashicorp/mysql-portworx-demo                                                    0                      
bitnamicharts/mysql                                                              0                      
databack/mysql-backup           Back up mysql databases to... anywhere!          105                    
linuxserver/mysql               A Mysql container, brought to you by LinuxSe…   41                     
mirantis/mysql                                                                   0                      
docksal/mysql                   MySQL service images for Docksal - https://d…   0                      
linuxserver/mysql-workbench                                                      54                     
vitess/mysqlctld                vitess/mysqlctld                                 1                    [OK]  
eclipse/mysql                   Mysql 5.7, curl, rsync                           1                    [OK]  
drupalci/mysql-5.5              https://www.drupal.org/project/drupalci          3                    [OK]  
drupalci/mysql-5.7              https://www.drupal.org/project/drupalci          0                      
datajoint/mysql                 MySQL image pre-configured to work smoothly …   2                    [OK]  
```  
`docker pull 镜像名[:tag]` 下载镜像  
```shell  
f10wers13eicheng@MacBookPro [12时25分27秒] [/Library/Java/JavaVirtualMachines]   
-&gt; % docker pull mysql  # 不写 tag 默认就是 latest  
Using default tag: latest  
latest: Pulling from library/mysql  
72a69066d2fe: Pull complete #分层下载 docker images的核心  
93619dbc5b36: Pull complete   
99da31dd6142: Pull complete   
626033c43d70: Pull complete   
37d5d7efb64e: Pull complete   
ac563158d721: Pull complete   
d2ba16033dad: Pull complete   
688ba7d5c01a: Pull complete   
00e060b6d11d: Pull complete   
1c04857f594f: Pull complete   
4d7cfa90e6ea: Pull complete   
e0431212d27d: Pull complete   
Digest: sha256:e9027fe4d91c0153429607251656806cc784e914937271037f7738bd5b8e7709 # 签名信息  
Status: Downloaded newer image for mysql:latest  
docker.io/library/mysql:latest #真实地址  
  
# 等价于  
docker pull mysql  
docker pull docker.io/library/mysql:latest  
```  
`docker rmi 镜像id` 删除镜像  
## 容器命令  
`docker pull centos` 下载centos镜像  
```shell  
docker run [可选参数] image  
# 参数说明  
--name=&#34;Name&#34; #容器名字  
-d            #后台方式运行  
-it           #是用交互方式运行，进入容器查看内容  
-p            #指定容器的端口 -p 8080:8080  
   -p ip:主机端口:容器端口  
   -p 主机端口:容器端口  
   -p 容器端口  
-P            #随机指定端口  
  
# 启动并进入容器  
f10wers13eicheng@MacBookPro [12时35分17秒] [/Library/Java/JavaVirtualMachines]   
-&gt; % docker run -it centos /bin/bash                           
[root@a9def1b9102e /]# ls # 查看容器内的目录  
bin  dev  etc  home  lib  lib64  lost&#43;found  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var  
[root@a9def1b9102e /]# whoami  
root  
[root@a9def1b9102e /]# exit #从容器中退回主机  
```  
**列出所有运行的容器**  
```shell  
docker ps 命令  
-a #列出当前正在运行的容器 &#43; 带历史运行过的容器  
-n=? #显示最近创建的容器  
-q #显示容器id  
```  
`exit` 直接容器停止并退出  
`Ctrl &#43; P &#43; Q` 容器不停止但退出  
**删除容器**  
```shell  
docker rm 容器id               #删除指定容器  
docker rm -f $(docker ps -aq) #删除所有容器  
```  
**启动和停止容器的操作**  
```shell  
docker start 容器id      #启动容器  
docker restart 容器id    #重启容器  
docker stop  容器id      #停止容器  
docker kill  容器id      #停止当前运行的容器并删除  
```  
  
## 常用的其他命令  
**后台启动容器**  
```shell  
# docker run -d 镜像名  
f10wers13eicheng@MacBookPro [12时46分50秒] [/Library/Java/JavaVirtualMachines]   
-&gt; % docker run -d centos        
8587f88f16bbc5fdb9dfb6de01c5173d9c25c23ed1f99d11b1d52575167cbefa  
f10wers13eicheng@MacBookPro [12时46分58秒] [/Library/Java/JavaVirtualMachines]   
-&gt; % docker ps             
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES  
  
# docker ps发现 centos停止了  
# 常见的坑 docker容器使用后台运行，就必须要有一个前台进程 docker 发现没有应用，就会自动停止  
# nginx，容器启动后，发现自己没有提供服务，就会立刻停止。  
```  
**查看日志**  
```shell  
docker logs # 查看日志信息  
docker logs -tf 容器id  
# 显示日志  
-tf      # 显示日志  
--tail   # 显示日志条数  
```  
**查看容器中进程信息**  
```shell  
docker top 容器id  
```  
**查看容器的元数据**  
`docker inspect 容器id`  
**进入当前正在运行的容器**  
```shell  
# 通常容器都是使用后台方式运行的，需要进入容器，修改一些配置  
# 命令  
docker exec -it 容器id /bin/bash #进入容器后开启一个新的终端  
  
docker attach 容器id # 进入容器正在执行的终端，不会启动一个新的进程  
```  
**从容器内拷贝文件到主机上**  
`docker cp 容器id:容器内路径 目的主机路径`  
  
## 部署 nginx - Docker练习  
`docker search nginx` 搜索镜像  
`docker pull nginx` 下载镜像  
`docker run -d --name nginx01 -p 3344:80 nginx` 启动镜像  
## 部署 tomcat - Docker练习  
`docker pull tomcat`  
`docker run -d --name tomcat01 -p 3345:8080 tomcat`  
`docker exec -it tomcat01 /bin/bash` 进入容器  
## Es&#43;Kibana - Docker练习  
```shell  
# --net 网络配置   
$ docker run -d --name elasticsearch -p 9200:9200 -p 9300:9300 -e &#34;discovery.type=single-node&#34; elasticsearch:7.6.2  
  
docker stats # 查看cpu状态  
-e #环境配置修改  
   
```  
# Docker 镜像  
如何得到镜像？  
- 从远程仓库下载
- 自己制作一个 Dockerfile
## 镜像加载原理  
`UnionFS(联合文件系统)`  
## commit镜像  
```shell  
docker commit 提交容器称为一个新的副本  
docker commit -m=&#34;提交的描述信息&#34; -a=&#34;作者&#34; 容器id 目标镜像名:[TAG]  
```  

---

> Author: N1Rvana  
> URL: https://nlrvana.github.io/docker%E5%9F%BA%E7%A1%80%E5%91%BD%E4%BB%A4%E5%8F%8A%E5%8E%9F%E7%90%86/  

