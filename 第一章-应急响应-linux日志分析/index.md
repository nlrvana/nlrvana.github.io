# 第一章-应急响应-Linux日志分析

  
  
&lt;!--more--&gt;  
1.有多少IP在爆破主机ssh的root帐号，如果有多个使用&#34;,&#34;分割  
2.ssh爆破成功登陆的IP是多少，如果有多个使用&#34;,&#34;分割  
3.爆破用户名字典是什么？如果有多个使用&#34;,&#34;分割  
4.登陆成功的IP共爆破了多少次  
5.黑客登陆主机后新建了一个后门用户，用户名是多少  
## 基础日志  
```bash  
登陆失败记录 /var/log/btmp  
最后一次登录 /var/log/lastlog  
登录成功记录 /var/log/wtmp  
登录日志记录 /var/log/secure  
```  
## 日志分析技巧  
```bash  
/var/log/secure  
定位有多少IP在爆破主机的root帐号:  
grep &#34;Failed password for root&#34; /var/log/secure | awk &#39;{print $11}&#39; | sort | uniq -c | sort -nr | more  
定位有哪些IP在爆破：  
grep &#34;Failed password&#34; /var/log/secure|grep -E -o &#34;(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)&#34;|uniq -c  
登陆成功的IP有哪些  
grep &#34;Accepted &#34; /var/log/secure | awk &#39;{print $11}&#39; | sort | uniq -c | sort -nr | more  
  
```  
分许`auth.log`日志  
查看多少IP在爆破主机ssh的root帐号  
`grep -a &#34;Failed password for root&#34; /var/log/auth.log.1 | awk &#39;{print $11}&#39; | uniq -c | sort -nr | more`  
`flag{192.168.200.2,192.168.200.32,192.168.200.31}`  
登陆成功的IP有哪些  
`grep -a &#34;Accepted &#34; /var/log/auth.log.1 | awk &#39;{print $11}&#39; | uniq -c | sort -nr | more`  
`flag{192.168.200.2}`  
爆破用户名字典是什么？  
`grep -a &#34;Failed password&#34; /var/log/auth.log.1 | perl -e &#39;while($_=&lt;&gt;){ /for(.*?) from/; print &#34;$1\n&#34;;}&#39;| uniq -c | sort -nr`  
`flag{user,hello,root,test3,test2,test1}`  
登陆成功的IP共爆破了多少次  
`grep -a &#34;Failed password for root&#34; /var/log/auth.log.1 | awk &#39;{print $11}&#39; | uniq -c | sort -nr | more`  
`grep -a &#34;Accepted &#34; /var/log/auth.log.1 | awk &#39;{print $11}&#39; | uniq -c | sort -nr | more`  
`flag{4}`  
黑客登陆主机后新建了一个后门用户，用户名是多少  
`cat /etc/passwd` 查看可疑用户 或者利用 `cat /var/log/auth.log.1 |grep -a &#34;new user&#34;`  
`flag{test2}`  

---

> Author: N1Rvana  
> URL: https://nlrvana.github.io/%E7%AC%AC%E4%B8%80%E7%AB%A0-%E5%BA%94%E6%80%A5%E5%93%8D%E5%BA%94-linux%E6%97%A5%E5%BF%97%E5%88%86%E6%9E%90/  

