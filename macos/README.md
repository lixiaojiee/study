本目录主要记录一些关于在macos使用中遇到的问题

1、macos使用homebrew安装java，其安装目录为

```
/Library/Java/JavaVirtualMachines
```

2、使用homebrew安装mysql

- 安装

```
brew install mysql
```

- 启动mysql服务

```
brew services start mysql
```

- 使用root用户登陆

```
 mysql -u root
```

- 创建用户

```
create user 'username'@'localhost' identified by 'password';
```

- 创建数据库

```
create database databasename;
```

- 授予权限给用户

```
grant all privileges on database.tablename to 'username'@'% identified by 'password';
```
或者
```
grant select,insert,update,create,delete,drop on database.tablename to 'username'@'%' identified by 'password';
```

- 收回授权
```
revoke all on databasename.tablename to 'username'@'%';
```

- 查看授权（该命令需要 root 用户执行）
```
show grants for 'router'@'%';
```

- 刷新授权
```
flush privileges;
```

3、显示隐藏文件夹

```
command + shift + .
```
