# Git学习记录

  
  
&lt;!--more--&gt;  
## Git的必要配置  
`git config -l` 查看配置文件  
`git config --system --list` 查看系统的git配置文件  
`git config --global --list` 查看当前用户配置  
设置用户名和邮箱  
```git  
user.email=beichenghua@gmail.com  
user.name=nlrvana  
```  
## Git项目创建  
`git init` 初始化文件   
`git clone` 克隆文件到本地  
`git add .` 放至暂存区  
`git commit -m &#34;message&#34;` 提交暂存区的内容到本地仓库  
## Git分支  
```git  
# 列出所有本地分支  
git branch  
# 列出所有远程分支  
git branch -r   
# 新建一个分支，并切换到该分支  
git checkout -b [branch]  
# 合并指定分支到当前分支  
git merge [branch]  
# 删除分支  
git branch -d [branch-name]  
# 删除远程分支  
git push origin --delete [branch-name]  
git branch -dr [remote/branch]  
```  

---

> Author: N1Rvana  
> URL: https://nlrvana.github.io/git%E5%AD%A6%E4%B9%A0%E8%AE%B0%E5%BD%95/  

