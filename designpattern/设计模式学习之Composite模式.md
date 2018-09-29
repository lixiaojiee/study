

# 一、什么是Composite模式

像文件系统的目录和文件那样，能够使容器与内容具有一致性，创造出地柜结构的模式，是Composite结构

# 二、Composite模式的代码示例

**Entry类：** 标识条目目录的抽象类

```java
package com.ateacer.study.designpattern.composite;

public abstract class Entry {
    public abstract String getName();

    public abstract int getSize();

    public Entry add(Entry entry) throws FileTreatMentException {
    throw new FileTreatMentException();
    }

    public void printList() {
        printList("");
    }

    public abstract void printList(String prefex);

    public String toString() {
        return getName() + "(" + getSize() + ")";
    }
}
```

**File类：**  标识文件的类，是Entry的子类

```java
package com.ateacer.study.designpattern.composite;

public class File extends Entry {

    private String name;

    private int size;

    public File(String name, int size) {
        this.name = name;
        this.size = size;
    }

    public String getName() {
        return name;
    }

    public int getSize() {
        return size;
    }

    public void printList(String prefex) {
        System.out.println(prefex + "/" + this);
    }
}
```

**Directory类：** 标识文件夹的类，它也是Entry类的子类

```java
package com.ateacer.study.designpattern.composite;

import java.util.ArrayList;
import java.util.List;

public class Directory extends Entry {

    private String name;

    private List<Entry> directorys = new ArrayList<Entry>();

    public Directory(String name) {
        this.name = name;
    }

    public String getName() {
        return name;
    }

    public int getSize() {
        int size = 0;

        for (Entry entry : directorys) {
            size += entry.getSize();
        }
        return size;
    }

    @Override
    public Entry add(Entry entry) throws FileTreatMentException {
        directorys.add(entry);
        return entry;
    }

    public void printList(String prefex) {
        System.out.println(prefex + "/" + this);

        for (Entry entry : directorys) {
            entry.printList(prefex + "/" + name);
        }
    }
}
```

**FileTreatMentException类：** 异常处理类

```java
package com.ateacer.study.designpattern.composite;

public class FileTreatMentException extends RuntimeException {
    public FileTreatMentException() {
    }

    public FileTreatMentException(String msg) {
        super(msg);
    }
}
```

# 三、Composite模式中的相关角色

- **Leaf(树叶)**

  标识内容的角色。在该角色中不能放入其他对象。在上述程序中为File类

- **Composite(复合物)**

  标识容器的角色。可以在其中放入Leaf角色和Composite角色，上述程序中为Directory类

- **Component**

  使Leaf角色和Composite角色具有一致的角色。Composite角色是Leaf角色和Composite角色的父类，在上述程序中为Entry类

**其类结构图如下所示:**

![](C:\Users\Administrator\Desktop\ZooKeeper学习\Composite模式类关系图.png)

# 四、Composite模式的特点

- **多个和单个的一致性**

  使用Composite模式可以使容器与内容具有一致性，也可可以称其为多个和单个的一致性，即将多个对象结合在一起，当做一个对象进行处理

