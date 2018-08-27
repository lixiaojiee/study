# 一、客户端脚本

## 1、客户端启动脚本

```shell
$ sh zkCli.sh
```

上边脚本是启动一个本地的zk服务器，如果想启动指定服务器地址，则可以执行如下命令：

```shell
$ sh zkCli.sh -server ip:port
```

## 2、创建节点

可以使用`create`命令，创建一个zookeeper节点，如下

```shell
$ create [-s] [-e] path data acl
```

其中，-s和-e分别指定节点特性：顺序或临时节点。默认情况下，创建的是持久节点，`create`命令的最后一个参数是acl，它是用来进行权限控制的，缺省情况下，不做任何权限控制

## 3、读取

与读取相关的命令包括`ls`命令和`get`命令

### 1）ls

使用`ls`命令，可以列出zookeeper指定节点下的所有**第一级**子节点，用法如下：

```shell
ls path [watch]
```

其中，path标识的是指定数据节点的节点路径

### 2）get

使用`get`命令，可以获取到zookeeper指定节点的数据内容和属性信息，用法如下：

```shell
get path [watch]
```

执行该命令后，除了获取到数据外，还获取到一些关于节点的信息：

a、创建该节点的事物ID：cZxid

b、最后一次更新该节点的事物ID：mZxid

c、最后一次更新该节点的时间：mtime

e、第一次更新该节点的时间：ctime

### 3）更新

使用`set`命令，可以更新指定节点的数据内容，用法如下：

```shell
set path data [version]
```

其中，data就是要更新的内容，version参数指的是恩赐更新操作是基于ZNode的哪一个数据版本进行的

### 4）删除

使用delete命令，可以删除zookeeper上的指定节点，用法如下：

```shell
delete path [version]
```

此处的version参数和`set`命令中的version参数的作用是一致的，但是要注意的是，**想要删除某一个指定节点，那么该节点必须没有子节点存在**

# 二、zookeeper的java客户端

zookeeper客户端和服务端会话的建立是一个异步的过程，也就是说在程序中，狗造方法会在处理完客户端初始化工作后立即返回，在大多数情况下，此时并没有真正建立好一个可用的会话，在会话的生命周期中处于“CONNECTING”状态。当该回话真正创建完毕后，zookeeper服务端会向会话对应的客户端发送一个事件通知，以告知客户端，客户端只有在获取这个通知之后，才算真正建立了会话