# Linux基础

  
  
&lt;!--more--&gt;  
## 开关机  
`shutdown`关机  
```shell  
sync # 将数据由内存同步到硬盘中  
shutdown # 关机指令  
```  
## 系统目录架构  
1、一切皆文件  
2、根目录/，所有的文件都挂载在这个节点下  
目录解释  
```shell  
/bin # bin是 binary 的缩写，这个目录存放着最常使用的命令  
/boot # 这里存放的是启动Linux时使用的一些核心文件，包括一些连接以及镜像文件  
/dev # dev是device的缩写，存放的是Linux的外部设备，在Linux中方问设备的方式和访问文件的方式是相同的  
/etc # 这个目录用来存放所有的系统管理所需要的配置文件和子目录  
/home # 用户的主目录，在linux 中，每个用户都有一个自己的目录，一般该目录名是以用户的账号命名的  
/lib # 这个目录存放着系统最基本的动态连接共享库  
/proc # 这个目录是一个虚拟的目录，他是系统内存的映射，我们可以直接访问这个目录来获取系统信息  
/root # 系统管理员的目录  
/usr # 用户的很多应用程序和文件都存在这个目录下  
/usr/bin # 系统用户使用的应用程序  
```  
## 常用的基本命令  
### 目录管理  
```shell  
cd / # 切换到根目录      绝对路径  
cd ../ #切换到上一级目录  相对路径  
  
ls # 列出目录  
-a # all 查看全部文件，包括隐藏文件  
-l # 列出所有文件，包括属性  
  
pwd # 显示当前所在的目录  
  
mkdir #创建目录  
-p # 创建多级目录  
  
rmdir #删除目录 仅能删除空的目录  
rmdir -p # 删除多级目录  
  
cp # 复制文件或者目录  
cp oldfile newfile  
  
rm # 移除文件或者目录  
-f # 忽略不存在的文件，不会出现警告，强制删除  
-r # 递归删除 mulu  
-i # 互动，删除询问是否删除  
  
mv # 移动文件或者目录 / 重命名文件  
-f # 强制移动  
-u # 只替换已经更新过的文件  
  
```  
### 基本属性  
```shell  
f10wers13eicheng@MacBookPro [21时49分24秒] [~/Desktop/题目]   
-&gt; % ls -l                      
total 8  
drwxr-xr-x  3 f10wers13eicheng  staff    96  2  2 00:17 assets  
drwxr-xr-x  6 f10wers13eicheng  staff   192  2  2 00:20 deploy  
-rw-r--r--@ 1 f10wers13eicheng  staff  2974  2  2 00:22 writeup.md  
  
// d 是目录  
// - 是文件  
// I 是链接文档  
// b 是配置文件里面的可供存储的接口设备  
// c 是配置文件里面的串行端口设备，例如键盘、鼠标等  
  
// 接下来的字符中，以三个为一组，且均为[rwx]的三个参数组合  
// 其中r代表可读 w代表可写 x代表可执行  
// 如果没有权限，则会出现&#34;-&#34;  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202402022153785.png)  
### 修改文件属性  
`chgrp` 更改文件属性  
`chgrp [-R] 属组名 文件名`  
`-R`递归更改文件属组  
`chown`更改文件属主，也可以同时更改文件属组  
`chown [-R] 属主名 文件名`  
`chown [-R] 属主名:属组名 文件名`  
`chmod [-R] xyz 文件/目录`  
`r:4 w:2 x:1`  
## 文件内容查看  
```shell  
cat #由第一行开始显示文件内容  
tac #由最后一行开始显示  
nl  #显示的时候，顺道输出行号  
more #一页页的显示文件内容  
less #可以往前翻页，其他与 more 相似  
head #只看头几行  
tail #只看尾巴几行  
```  
  
## Linux 链接概念  
Linux 链接分为两种 硬链接、软连接  
硬链接 A---B 假设 B 是 A 的硬链接，那么他们两个指向了同一个文件，允许一个文件拥有多个路径。  
软连接  类似 Windows 下的快捷方式，删除源文件，快捷方式也访问不了  
创建链接命令 ln  
```shell  
ln -s file link #软连接  
```  
## Linux账号管理  
添加用户  
`useradd 选项 用户名`  
-m 自动创建这个用户的主目录  
-G 指定用户组  
删除用户  
`userdel -r 用户名`  
-r 删除目录  
修改用户  
`usermod 选项 用户名`  
切换用户  
`su username`  
## Linux用户组管理  
`/etc/group `  
创建一个用户组  
`groupadd 组名`  
`-g id` 指定id  
删除用户组  
`groupdel 组名`  
修改用户组的权限和信息  
`groupmod -g id -n newname oldname`  
用户切换用户组  
```shell  
# 登录当前用户  
# newgrp root  
```  
  
## Linux磁盘管理  
`df`列出文件系统整体的磁盘使用量  
`du`检查磁盘空间使用量  
  
## Linux进程管理  
在 Linux 中，每一个程序都是有自己的一个进程，每一个进程都有一个 id 号  
每一个进程，都会有一个父进程  
`ps` 查看当前系统中正在进行的各种进程信息  
`-a` 显示当前终端运行的所有的进程信息  
`-u` 以用户的信息显示进程  
`-x` 显示后台运行进程的参数  
`ps -ef` 可以查看到父进程的信息  
`pstree -pu`  
`-p` 显示父id  
`-u` 显示用户组  
杀掉进程  
`kill -9 pid` 强制杀掉进程  
  

---

> Author: N1Rvana  
> URL: http://localhost:1313/linux%E5%9F%BA%E7%A1%80/  

