在数据结构中保存着很多元素 ，我们会对这些元素进行“处理”，这时，”处理“代码需要放在哪里呢？通常的做法是将它们放在标识数据结构的类中。但是，如果”处理“有许多种呢？这种情况下，每当增加一种处理，我们就不得不去修改表示数据结构的类

**在visitor模式中，数据结构与处理被分离开来**，我们编写一个表示visitor的类来访问数据结构中的元素，并把对各个元素的处理交个visitor类。这样，当需要增加新的处理时，我们只需要编写新的访问者，然后让数据结构可以接受visitor的访问即可

# 一、Visitor模式的示例代码如下

## 1、Visitor类

Visitor类是访问者的抽象类，访问者依赖于它所访问的数据结构

```java
package com.ateacer.study.designpattern.visitor;

import com.ateacer.study.designpattern.visitor.Directory;

public abstract class Visitor {

    public abstract void visit(File file);

    public abstract void visit(Directory directory);
}
```

## 2、Element接口

Visitor类表示访问者的类，而Element接口则是接受访问者的访问的接口

```java
package com.ateacer.study.designpattern.visitor;

public interface Element {

    void accept(Visitor visitor);

}
```

## 3、Entry类

Entry类是对Element接口的进一步抽象

```java
package com.ateacer.study.designpattern.visitor;

import com.ateacer.study.designpattern.composite.FileTreatMentException;

import java.util.Iterator;

public abstract class Entry implements Element {

    public abstract String getName();

    public abstract int getSize();

    public Entry add(Entry entry) throws FileTreatMentException {
        throw new FileTreatMentException();
    }

    public Iterator<Entry> iterator() throws FileTreatMentException {
        throw new FileTreatMentException();
    }

    @Override
    public String toString() {
        return getName() + " (" + getSize() + ")";
    }
}
```

## 4、File类

```java
package com.ateacer.study.designpattern.visitor;

public class File extends Entry {

    private String name;

    private int size;

    public File(String name, int size) {
        this.name = name;
        this.size = size;
    }

    public String getName() {
        return null;
    }

    public int getSize() {
        return size;
    }

    public void accept(Visitor visitor) {
        visitor.visit(this);
    }
}
```

## 5、Directory类

```java
package com.ateacer.study.designpattern.visitor;

import java.util.ArrayList;
import java.util.Iterator;

public class Directory extends Entry {

    private String name;

    private ArrayList<Entry> dir = new ArrayList<Entry>();

    public Directory(String name) {
        this.name = name;
    }

    public Iterator iterator() {
        return dir.iterator();
    }

    public String getName() {
        return name;
    }

    public int getSize() {
        int size = 0;
        Iterator<Entry> iterator = iterator();
        while (iterator.hasNext()) {
            Entry entry = iterator.next();
            size += entry.getSize();
        }
        return size;
    }

    public void accept(Visitor visitor) {
        visitor.visit(this);
    }

    public Entry add(Entry entry) {
        dir.add(entry);
        return this;
    }
}
```

## 6、ListVisitor类

ListVisitor类是Visitor的实现，如下：

```java
package com.ateacer.study.designpattern.visitor;

import java.util.Iterator;

public class ListVisitor extends Visitor {

    private String currentDir = "";

    public void visit(File file) {
        System.out.println(currentDir + "/" + file);
    }

    public void visit(Directory directory) {
        System.out.println(currentDir + "/" + directory);
        String savedir = currentDir;
        currentDir = currentDir + "/" + directory.getName();

        Iterator<Entry> iterator = directory.iterator();
        while (iterator.hasNext()) {
            Entry entry = iterator.next();
            entry.accept(this);
        }

        currentDir = savedir;
    }
}
```

# 二、Visitor模式中的登场角色

-  **Visitor（访问者）**	

  Visitor角色负责对数据结构中每个具体的元素（ConcreteElement角色）声明一个用于访问xxx的visit(xxx)方法。visit(xxx)是用于处理xxx的方法，负责实现该方法的ConcreteVisitor角色

- **ConcreteVisitor（具体的访问者）**

  ConcreteVisitor角色负责实现Visitor角色所定义的接口。它要实现所有的visit(xxx)方法，即实现如何处理 每个ConcreteElement角色

- **Element（元素）**

  Element角色标识Visitor角色的访问对象。它声明了接收访问者accept方法。accept方法接收到的参数是Visitor角色

- **ConcreteElement**

  ConcreteElement角色负责实现Element角色所定义的接口

- **ObjectStructure（对象结构）**

  ObjectStructur角色负责处理Element角色的集合。ConcreteVisitor角色为每个Element都准备了处理方法。在示例程序中，Directory扮演此角色

其类关系图如下：

![](../pictures/ZooKeeper学习\Vistor模式类关系图.png)
