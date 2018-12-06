# 一、功能

返回路由后的可调用者（Invoker）列表

Directory中的destroyed和forbidden的区别，什么情况下会出现destroyed，什么情况下会出现forbidden

forbidden：

1、没有提供者

2、提供者都被禁了

RegistryDirectory中维护的Invokers，有两种来源，一种是在工程启动时加载的zookeeper本地缓存，另一种是zookeeper对应节点在发生变化的时候，通知给客户端的Notifier

## 1、RegistryDirectory

### a、list过程



### b、RegistryDirectory接收zookeeper通知的处理过程

1. 从接收到的变化的URL中解析出protocol和category

2. 根据protocol和category，分别将URL进行分类缓存

   1. 若catagory是routers或者protocol是router，则将该URL缓存入routerUrls
   2. 若category是configurators或者protocol是override，则将该URL缓存入configuratorUrls
   3. 若category是providers，则将该URL缓存入providers
   4. 如果不在上述三种情况内，则丢弃该URL，不进行处理

3. 将configuratorUrls中的URL转化成Configurator

4. 将routerUrls中的URL转化成Router，并传到AbstractDirectory中，因为这里做了Router的路由操作，该操作是通过配置`runtime=true`来激活此步骤的，如果被激活，那么每次请求都会进行多个路由器的路由操作，这样会影响性能，**生产上不建议这么做** 

5. 刷新Invoker列表

   1. 若列表中只有一个元素且protocol为empty的URL，则将forbidden置为true，说明服务都被禁止了，或者服务取消了注册

      将缓存的方法->Invokers列表置为null，这里存放的是某个方法对应的Invoker列表

      销毁所有的Invoker(调Invoker的destroy方法)

   2. 若不满足1的条件，则

      a. 将forbidden置为false

      b. 如果通知的URL列表为空，且当前缓存的URL列表不为空，则将当前缓存的URL列表填入通知URL列表中，并直接返回

      c. 如果通知的URL列表不为空，则将其缓存，以备后续使用

      d. 将最新的URL列表转换成url->Invoker映射列表

      e. 将d步骤的结果转换成method->Invoker列表

      f. 如果发现d、e两步的结果，有一个为空，则说明没有可用的服务列表，直接退出

      g. 对method->Invoker列表进行分组(group)处理

      ​    若需要分组：

      ​     i、若method->Invoker列表中只有一个元素，也就是所有的服务同处于同一分组中，那么就将分组对应的       Invoker列表遍历，再与method关联

      ​     ii、若method->Invoker列表中有多个元素，则将Invoker转换成**FailoverClustInvoker(或者其它集群Invoker)**,再与method关联

      ​    iii、若果method-Invoker列表中没有元素，则将原映射直接返回

      ​    若不需要分组：

      ​    直接返回原映射关系

      h. 将处理后的method->Invoker列表进行缓存

      i. 销毁无用的Invoker

      ​    i、若新method->Invoker列表或者老的列表有一个为空则，全部进行销毁

      ​    ii、分别遍历新映射表和老映射表，如果发现老列表中的某Invoker不在新列表中，则将将其记录

      ​    iii、遍历ii步骤中的元素，将老列表中对应元素删除，并执行该Invoker的destroy方法

      **RegistryDirectory在以下几个地方使用：**

      - DubboRegistry的createRegistry方法
      - RegistryProtocol的doRefer方法

## 2、StaticDirectory

StatiDirectory就是维护了一份服务列表，运行时不会发生变化