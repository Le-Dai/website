---
slug: scala-java-static-methods-not-available-issue
title: Scala 引用Java 父类static methods and static variable not available 问题
authors: [ledai]
tags: [学习历程, scala问题]
date: 2018-10-30
---

<h1>Scala 引用Java 父类static methods and static variable not available 问题</h1>
<!-- truncate -->

  今天,在搞accumulo 与spark 整合的时候  在scala code中引用 java时发现了一个算是比较严重的问题, 情况是这样的,AccumuloOutputFormat 是一个类暂时不用管他是干嘛的,
他继承了InputFormatBase 也就是他的父类  在父类中有很多的静态方法  在java中 通过集成AccumuloOutputFormat应该是会继承父类的静态方法与静态属性的,然后在我的java client中调用
AccumuloOutputFormat的父类静态方法也是没有问题的,但是在写scala中 需要引用到java  在scala中报了一个 can not symbol *  也就是说无法获取到该类的静态方法,但是照理说他继承了父类应该
是由方法的,所以我做了下面的测试  首先我写了一个父类（JAVA）
```java
package com.wxstc.dl;


public class Father {
    public static String v1 = "haha";
    public static void sayHello() {
        System.out.println("hello");
    }
}
```

然后是子类(JAVA)
```java
package com.wxstc.dl;



public class Tjava extends Father{
    public static void TJavaSayHello(){
        System.out.println("tjava hello");
    }
    public static void main(String[] args) {
       //在方法中引用父类静态方法
       Tjava.sayHello();
       //在方法中引用自己的静态方法
       Tjava.TJavaSayHello();

    }
}
```

到这里 在java中 均为报错  编译通过 语法没有任何问题  接下来在scala中

![Pulsar](https://raw.githubusercontent.com/MrDLontheway/mrdlontheway.github.io/master/images/issue1.png)

图中代码提示助手 提示中是没有父类的静态方法的 我以为是idea scala插件的问题所以 手写上去 进行编译

![Pulsar](https://raw.githubusercontent.com/MrDLontheway/mrdlontheway.github.io/master/images/issue2.png)

编译错误中将Tjava 当作了object  因为object是不支持继承object 所以无法继承到父类静态方法 这时候我想测试如果我使用scala object 去继承java class 能否解决这个问题

接下来 Test1为scala object

```scala

package com.wxstc

import com.wxstc.dl.Test2

object Test1 extends Test2{
  def MysayHello(): Unit = {
    System.out.println("hello")
  }
}
```

```java
package com.wxstc.dl;

public class Test2 {
    public static void sayHello(){
        System.out.println("hello");
    }
}
```

结果依旧报错  编译无法通过

最后我尝试了用scala的class 去集成 java的class  然后达到继承父类静态方法的目的

结果依然是无法继承  所以总结出 因为scala 中是没有static关键字的  静态是通过object 伴生对象来实现(个人理解)  所以只要在scala中 引用了静态的方法或者变量 scala都会把引用类作为
object半生对象去解析 但是object是不支持继承的 所以无法获取到父类的静态方法 编译器无法通过  从语法的角度我觉得这对于scala来说应该算是一个问题  已经在scala 提交 issues.. 


```java
object AccumuloTest {
  def main(args: Array[String]): Unit = {
    //无法引用 父类静态方法
    Tjava.sayHello()
    Tjava.TJavaSayHello()

    //无法引用继承的静态变量
    val v = Tjava.v1
    val v1 = new Tjava
    Test1.MysayHello()
    //无法引用 父类静态方法
    Test1.sayHello()
  }

}

public class Father {
    public static String v1 = "haha";
    public static void sayHello() {
        System.out.println("hello");
    }
}

object Test1 extends Test2{
  def MysayHello(): Unit = {
    System.out.println("hello")
  }
}

public class Test2 {
    public static void sayHello(){
        System.out.println("hello");
    }
}

//object 不支持 继承object
object Test3 extends Test4{
  def MysayHello(): Unit = {
    System.out.println("hello")
  }
}

object Test4 {
  def sayHello(): Unit = {
    System.out.println("hello")
  }
}


public class Tjava extends Father{
    public static void TJavaSayHello(){
        System.out.println("tjava hello");
    }
    public static void main(String[] args) {
        Tjava.sayHello();
        Tjava.TJavaSayHello();

    }
}

```

 
<h1>Scala issues</h1>

https://github.com/scala/bug/issues/11230

目前针对这个问题的讨论

https://github.com/scala/bug/issues/11230

设计者的目前想法是 因为语言的特性导致了scala 与 java的不兼容  但是scala并不需要全部去 模仿 适应 java 也提到如果有人有完整设计解决思路的话 会考虑pull request

就我目前个人的解决方案 scala中引用java 的父类继承静态方法时 可以在引用静态方法层使用java操作  也就是将这个操作使用java中转  注意要使用非静态方法中转 因为只要是静态方法

在scala中还是会以object方式去解析  然后在scala中 通过 new java实例去调用方法执行java内的静态继承方法