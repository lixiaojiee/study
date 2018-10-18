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
brew service start mysql
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
grant privileges/all/select/delete/create/update on 'tablename'@'localhost' to 'username';
```

