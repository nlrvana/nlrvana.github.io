# 内网渗透-Linux信息收集(11)

  
  
&lt;!--more--&gt;  
进入Linux系统第一步，设置不记录系统命令  
```  
unset HISTORY HISTFILE HISTSAVE HISTZONE HISTORY HISTLOG; export HISTFILE=/dev/null;  
export HISTSIZE=0;  
export HISTFILESIZE=0  
```  
常规需要收集的信息  
查看历史记录  
`history`  
查看ssh连接记录，  
`/root/.ssh`  
## 系统信息  
### 操作系统基本信息  
```  
uname -a 打印所有可用信息  
uname -r 内核版本信息  
uname -n 系统主机名  
uname -m linux内核架构  
hostname 主机名  
cat /proc/version 内核信息  
cat /etc/*-release 发布信息  
cat /etc/issue 发布信息  
cat /proc/cpuinfo cpu信息  
df -a 文件信息  
```  
**获取系统环境信息**  
```  
env 输出系统含经信息  
set 打印系统环境信息  
echo $PATH 输出环境变量中的路径信息  
pwd 当前路径  
cat /etc/profile 全局环境变量  
```  
**获取系统服务信息**  
```  
ps -aux 查看进程信息  
top 当前进程  
netstat -antpu 查看当前交互端口  
```  
**获取系统软件信息**  
```  
dpkg -l 已经安装的软件列表(debian ubuntu)  
rpm -qa 已经安装的软件列表(redhat centos)  
```  
**获取系统任务和作业**  
```  
crontab -l -u 用户名   
/etc/crontab 文件中可能记录了用户自行添加的任务  
jobs -l 列出放在后台的进程  
ls -al /etc/crontab 列出所有计划任务  
```  
**其他信息**  
```  
#查看开机启动项  
systemctl list-unit-files  
#用rc3.d这个目录为例，这个目录里面记录的是进入init 3时需要停止和启动的哪些服务  
ls /etc/rc3.d  
```  
K开头代表这个启动级别需要停止的服务，编号是停止的时候执行的顺序。  
S开头则是要启动哪些服务。  
**网络信息**  
```bash  
/sbin/ifconfig -a #列出网络接口信息  
cat /etc/network/interfaces #列出网络接口信息  
arp -a #查看系统ARP表  
route #打印路由信息  
cat /etc/resolv.conf #查看dns配置信息  
netstat -an #打印本地端口开放信息  
iptables -L #列出iptables的配置规则  
cat /etc/services #查看端口服务映射  
ifconfig #查看所有网络接口的属性  
route -n #查看路由表  
netstat -lntp #查看所有监听端口  
netstat -antp #查看所有已经建立的连接  
```  
`mimipenguin` 工具获取密码  
`bashark.sh`  
  
  

---

> Author: N1Rvana  
> URL: http://localhost:1313/%E5%86%85%E7%BD%91%E6%B8%97%E9%80%8F-linux%E4%BF%A1%E6%81%AF%E6%94%B6%E9%9B%8611/  

