# 第二章 日志分析 Apache日志分析

  
  
&lt;!--more--&gt;  
1、提交当天访问次数最多的IP，即黑客IP：  
2、黑客使用的浏览器指纹是什么，提交指纹的md5：  
3、查看index.php页面被访问的次数，提交次数：  
4、查看黑客IP访问了多少次，提交次数：  
5、查看2023年8月03日8时这一个小时内有多少IP访问，提交次数:  
## Apache日志分析技巧  
```bash  
1、列出当天访问次数最多的IP命令：  
cut -d- -f 1 log_file|uniq -c | sort -rn | head -20  
  
2、查看当天有多少个IP访问：  
awk &#39;{print $1}&#39; log_file|sort|uniq|wc -l  
  
3、查看某一个页面被访问的次数：  
grep &#34;/index.php&#34; log_file | wc -l  
  
4、查看每一个IP访问了多少个页面：  
awk &#39;{&#43;&#43;S[$1]} END {for (a in S) print a,S[a]}&#39; log_file  
  
5、将每个IP访问的页面数进行从小到大排序：  
awk &#39;{&#43;&#43;S[$1]} END {for (a in S) print S[a],a}&#39; log_file | sort -n  
  
6、查看某一个IP访问了哪些页面：  
grep ^111.111.111.111 log_file| awk &#39;{print $1,$7}&#39;  
  
7、去掉搜索引擎统计当天的页面：  
awk &#39;{print $12,$1}&#39; log_file | grep ^\&#34;Mozilla | awk &#39;{print $2}&#39; |sort | uniq | wc -l  
  
8、查看2018年6月21日14时这一个小时内有多少IP访问:  
awk &#39;{print $4,$1}&#39; log_file | grep 21/Jun/2018:14 | awk &#39;{print $2}&#39;| sort | uniq | wc -l  
```  

---

> Author: N1Rvana  
> URL: https://nlrvana.github.io/%E7%AC%AC%E4%BA%8C%E7%AB%A0-%E6%97%A5%E5%BF%97%E5%88%86%E6%9E%90-apache%E6%97%A5%E5%BF%97%E5%88%86%E6%9E%90/  

