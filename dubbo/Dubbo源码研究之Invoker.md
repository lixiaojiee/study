# 一、Invoker在Dubbo中所扮演的角色是什么

Invoker在Dubbo中，代表的是一个可调用的远程服务，它包装了该调用者相关的所有参数，在负载均衡、容错等逻辑中均有重要应用

**关于FailoverClusterInvoker的相关注意点：**

在进行负载均衡选择的时候，如果选择出来的Invoker已经被选择过了，或者该Invoker当前不可用且开启了可用性检查，则会触发reselect逻辑，如果通过reselect得到的Invoker不为空，则使用这个Invoker，否则，选择该Invoker在列表中位置的下一个索引Invoker

**什么是可用性检查？**

availablecheck：



**什么是sticky？**

sticky：