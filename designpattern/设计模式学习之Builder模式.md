**Builder模式：**用于组装具有复杂结构的实例

**Builder的代码示例：**

builder类:

```java
package com.ateacer.study.designpattern.builder;

public interface Builder {

    void makeTitle(String title);

    void makeString(String str);

    void makeItems(String[] items);

    void close();
}
```

Director类：

```java
package com.ateacer.study.designpattern.builder;

public class Director {

    private Builder builder;

    public Director(Builder builder) {
        this.builder = builder;
    }

    public void constract() {
        builder.makeTitle("福尔摩斯探案集");
        builder.makeString("前五章");
        builder.makeItems(new String[]{"第一章", "第二章", "第三章", "第四章", "第五章"});
        builder.close();
    }
}
```

ConcreteDirector类:

```java
package com.ateacer.study.designpattern.builder;

public class ConcreteDirector implements Builder {

    private StringBuffer buffer = new StringBuffer();

    public void makeTitle(String title) {
        buffer.append("==============================\n");
        buffer.append("《" + title + "》\n");

    }

    public void makeString(String str) {
        buffer.append("<b>" + str + "<\b>\n");
    }

    public void makeItems(String[] items) {
        for (String item : items) {
            buffer.append(item + "\n");
        }
    }

    public void close() {
        buffer.append("==============================\n");
    }

    public String getResult(){
        return buffer.toString();
    }
}
```

Main：

```java
import com.ateacer.study.designpattern.builder.Builder;
import com.ateacer.study.designpattern.builder.Director;
import com.ateacer.study.designpattern.builder.StoryBuilder;

public class Main {

    public static void main(String[] args){
        ConcreteDirector builder = new ConcreteDirector();
        Director director = new Director(builder);

        director.constract();

        System.out.println(builder.getResult());
    }
}
```

Builder模式中的各种角色:****

- **Builder(建造者):**

  Builder角色负责定义用于生成实例的接口。Builder角色中准备了用友生成实例的方法

- **ConcreteBuilder(具体的建造者):**

  ConcreteBuilder角色是负责实现Builder角色的接口的类。这里定义了在生成实例时实际被调用的方法

- **Director(监工):**

  Director角色负责使用Builder角色的接口来生成实例。它并不依赖于ConcreteBuilder角色。为了确保不论ConcreteBuilder角色

