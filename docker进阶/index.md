# Docker进阶

  
  
&lt;!--more--&gt;  
## 容器数据卷  
容器之间可以有一个数据共享的技术  
Docker 容器中产生的数据，同步到本地  
目录的挂载，将我们容器内的目录，挂载到宿主机上面  
### 使用数据卷  
方式一：直接使用命令来挂载 `-v`  
`docker run -it -v 主目录:容器内目录 -p 主机端口:容器内端口`  
## 实战练习-MySQL  
```shell  
docker pull mysql  
docker run -d --name mysql -p 3310:3306 -v ./mysql/conf:/etc/mysql/conf -v ./mysql/data:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=huahua123 mysql   
```  
### 具名和匿名挂载  
```shell  
# 匿名挂载  
-v 容器内路径  
docker run -d -P --name nginx01 -v /etc/nginx nginx  
  
# 查看所有卷的情况  
docker volume ls  
  
# 具名挂载  
docker run -d -P --name nginx02 -v juming-nginx:/etc/nginx nginx  
-v 卷名:容器内路径  
```  
所有的 docker 容器内的卷，没有指定目录的情况下都是在`/var/lib/docker/volumes/xxx/_data`  
```shell  
-v 名字:容器内路径 #具名挂载  
-v 容器内路径     #匿名挂载  
-v /宿主机路径:容器内路径 #指定路径挂载  
  
# ro readonly 只读  
# rw readwrite 可读可写   
docker run -d -P --name nginx02 -v juming-nginx:/etc/nginx:ro nginx  
```  
## Dockerfile  
`Dockerfile`就是用来构建 docker 镜像的构建文件  
```dockerfile  
FROM centos  
VOLUME [&#39;volume01&#39;,&#39;volume02&#39;]  
CMD echo &#34;---end---&#34;  
CMD /bin/bash  
```  
构建步骤  
编写一个 dockerfile 文件  
- docker build 构建为一个镜像  
- docker run 运行镜像  
- docker push 发布镜像  
### Dockerfile构建过程  
基础知识  
1、每个保留关键字(命令)都必须是大写字母  
2、执行从上到下顺序执行  
3、#表示注释  
4、每一个指令都会创建一个新的镜像层，并提交  
dockerfile 是面向开发的，以后发布项目，做镜像，就需要编写 dockerfile 文件  
### Dockerfile的命令  
```dockerfile  
FROM       #基础镜像  
MAINTAINER #镜像是谁写的，姓名&#43;邮箱  
RUN        #镜像构建的时候需要执行的命令  
ADD        #步骤：tomcat容器，这个 tomcat压缩包！添加内容  
WORKDIR    #镜像的工作目录  
VOLUME     #设置卷，挂载主机目录  
EXPOSE     #暴露端口配置  
CMD        #指定这个容器启动的时候需要运行的命令，只有最后一个生效，可被替代  
ENTRYPOINT #指定这个容器启动的时候要运行的命令，可以追加命令  
ONBUILD    #当构建一个被继承Dockerfile 这个时候就会运行 ONBUILD 的命令，触发指令  
COPY       #类似 ADD，将文件拷贝到镜像中  
ENV        #构建的时候设置环境变量  
```  
### 实战练习-构建自己的 centos  
```dockerfile  
FROM centos  
MAINTAINER huahua&lt;example@qq.com&gt;  
  
ENV MYPATH /usr/local  
WORKDIR $MYPATH  
  
RUN yum -y install vim  
RUN yum -y install net-tools  
  
EXPOSE 80  
  
CMD echo $PATH  
CMD echo &#34;---end---&#34;  
CMD /bin/bash  
```  
`docker build -f dockerfile -t mycentos:0.1 .`  
`docker build -f dockerfile文件名 -t 镜像名[:TAG] .`  
`docker history 容器id`  
列出容器构建的记录  
### CMD和ENTRYPOINT的区别  
```shell  
CMD        #指定这个容器启动的时候需要运行的命令，只有最后一个生效，可被替代  
ENTRYPOINT #指定这个容器启动的时候要运行的命令，可以追加命令  
```  
dockerfile  
```dockerfile  
FROM centos  
CMD  [&#39;ls&#39;,&#39;-a&#39;]  
```  
`docker run 容器id ls -al`  
dockerfile  
```dockerfile  
FROM centos  
ENTRYPOINT [&#39;ls&#39;,&#39;-a&#39;]  
```  
`docker run 容器id -l`  
### 实战练习-Dockerfile制作tomcat镜像  
准备`apache-tomcat`和`jdk`的压缩包  
```dockerfile  
FROM centos  
MAINTAINER huahua&lt;beichenghua@gmail.com&gt;  
  
COPY readme.txt /usr/local/readme.txt  
  
ADD  jdk-8u111-linux-x64.tar.gz /usr/local/  
ADD  apache-tomcat-9.0.85.tar.gz /usr/local/  
  
RUN yum -y install vim  
  
RUN MYPATH /usr/local  
WORKDIR $MYPATH  
  
ENV JAVA_HOME /usr/local/jdk1.8.0_111  
ENV CLASSPATH $JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar  
ENV CATALINA_HOME /usr/local/apache-tomcat-9.0.85  
ENV CATALINA_BASH /usr/local/apache-tomcat-9.0.85  
ENV PATH $PATH:$JAVA_HOME/bin:$CATALINA_HOME/lib:CATALINA_HOME/bin  
  
EXPOSE 8080  
  
CMD /usr/local/apache-tomcat-9.0.85/bin/startup.sh &amp;&amp; tail -F /usr/local/apache-tomcat-9.0.85/logs/catalina.out  
```  
`docker build -t apache-tomcat:0.1 .`  
## 发布镜像到DockerHub/阿里镜像  
https://hub.docker.com/  
`docker tag imageId newImageName:tag`  
`docker push 镜像`  
  
## 容器互联 --link  
`docker run -d -P --name tomcat03 --link tomcat02 tomcat`  
tomcat03连接tomcat02，使用服务名连接  
原理:在 hosts 文件中进行了配置  
## 自定义网络  
`docker network ls` 查看 docker 所有的网络  
**网络模式**  
bridge：桥接 docker（默认）  
none：不配置网络  
host：和宿主机共享网络  
container：容器网络连通  
```shell  
--driver bridge 桥接模式  
--subnet 192.168.0.0/16 网段  
--gateway 192.168.0.1 网关(出网地址)  
  
docker network create --driver bridge --subnet 192.168.0.0/16 --gateway 192.168.0.1 mynet  
```  
## Docker-compose  
**配置案例**  
```yaml  
# yaml 配置实例  
version: &#39;3&#39;  
services:  
  web:  
    build: .  
    ports:  
    - &#34;5000:5000&#34;  
    volumes:  
    - .:/code  
    - logvolume01:/var/log  
    links:  
    - redis  
  redis:  
    image: redis  
volumes:  
  logvolume01: {}  
```  

---

> Author: N1Rvana  
> URL: http://localhost:1313/docker%E8%BF%9B%E9%98%B6/  

