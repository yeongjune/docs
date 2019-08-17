# Linux下修改Mysql的用(root的密码及修改root登录权限
修改的用户都以root为列。

**知道原来的myql数据库的root密码；**  
①： 在终端命令行输入 mysqladmin -u root -p password "新密码" 回车 ，Enter password: 【输入原来的旧密码】  
 ②： 登录mysql系统修改， mysql -uroot -p 回车 Enter password: 【输入原来的密码】  

```   
      mysql> use mysql;
      mysql> update user set password=password("新密码") where user='root';        【密码注意大小写】
      mysql> flush privileges;
      mysql> exit; 
```

然后使用刚才输入的新密码即可登录。  

**不知道原来的myql的root的密码**

首先，你必须要有操作系统的root权限了。要是连系统的root权限都没有的话，先考虑root系统再走下面的步骤。 类似于安全模式登录系统。
需要先停止mysql服务，这里分两种情况，  
一种可以用service mysqld stop  
另外一种是/etc/init.d/mysqld stop  
当提示mysql已停止后进行下一步操作   Shutting down MySQL. SUCCESS!
在终端命令行输入一下命令  【登录mysql系统】

> mysqld_safe --skip-grant-tables &         【

输入mysql登录mysql系统
```
mysql> use mysql;
mysql> UPDATE user SET password=password("新密码") WHERE user='root';      【密码注意大小写】
mysql> flush privileges;
mysql> exit;
```

重新启动mysql服务
这样新的root密码就设置成功了。
 
**修改root登录权限**

当你修改好root密码后，很有可能出现这种情况  
`ERROR 1045 (28000): Access denied for user 'root'@'localhost' (using password: YES)`   
这是因为root登录权限不足，具体修改方法如下  
需要先停止mysql服务，这里分两种情况，   
一种可以用service mysqld stop， 
另外一种是/etc/init.d/mysqld stop    
当提示mysql已停止后进行下一步操作   Shutting down MySQL. SUCCESS! 
在终端命令行输入【登录mysql系统】 
> mysqld_safe --skip-grant-tables & 

输入mysql登录mysql系统

```
mysql>use mysql;
mysql>update user set host = '%' where user = 'root';
mysql>select host, user from user;
mysql> flush privileges;
mysql> exit;
```
然后重新启动mysql服务就可以了
