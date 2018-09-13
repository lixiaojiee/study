 Iterator模式用于在数据集合中按照顺序遍历集合

**Iterator模式中登场的角色：**

- Iterator（迭代器）

该角色负责定义按照顺序逐个遍历元素的接口

- ConcreteIterator（迭代器的实现）

该角色负责实现Iterator角色所定义的接口

- Aggregate（集合）

该角色负责定义创建Iterator角色的接口，它会创建出“按照顺序访问保存在我内部的元素的人”

- ConcreteAggregate（集合的实现）

该角色负责实现Aggregate角色所定义的接口。它会创建出具体的Iterator角色

**for循环也可以完成迭代实现，为什么要用实现复杂的Iterator模式呢？**

一个重要的理由就是，引入Iterator后可以将遍历与实现分离开来

**迭代器中的next：**

next返回的都是当前的元素，并指向下一个元素

**迭代器中的hasNext：**

确认接下来是否可以调用next方法

hasNext方法并不是真正的判断是否还有下一个元素，而是判断当前下标是否已经越界，如果没有，则可以通过next获取当前元素，否则，遍历停止