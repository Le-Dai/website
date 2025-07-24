---
slug: framework-source-modification-classpath-loading-order
title: 关于修改三方框架源码 classpath加载顺序问题总结
authors: [ledai]
tags: [源码修改, maven 加载机制]
date: 2019-01-03
---

<h1>关于修改三方框架源码 classpath加载顺序问题总结</h1>
<!-- truncate -->

  之前提到过如何优雅的修改第三方以来源码 实现自己流程控制的方法,今天在修改源码的时候遇到了一个问题,源码修改完后 没有生效 运行时加载了 源class
文件, 首先解决这个问题要先了解jvm 的类加载机制,首先 运行时检测到class信息时 jvm会查看内部是否有该class 类加载信息 如果有是不会再次
加载的,所以说class 只会加载一次 也就是说在加载class的时候 源依赖加载的顺序优先于我们自己修改的源码 下面以idea 设置为例讲解 eclipse
同理只不过设置的地方不一样。


<h1>maven 项目的以来加载顺序</h1>
  这里主要讲解maven project  因为目前java 项目maven 毕竟是主流 首先是maven的 依赖原则
  
1.间接依赖路径最短优先

一个项目test依赖了a和b两个jar包。其中a-b-c1.0 ， d-e-f-c1.1 。由于c1.0路径最短，所以项目test最后使用的是c1.0。

2.pom文件中申明顺序优先

有人就问了如果 a-b-c1.0 ， d-e-c1.1 这样路径都一样怎么办？其实maven的作者也没那么傻，会以在pom文件中申明的顺序那选，如果pom文件中先申明了d再申明了a，test项目最后依赖的会是c1.1

所以maven依赖原则总结起来就两条：路径最短，申明顺序其次。

根据上面的两大原则  则不难找到问题的原因  但是每次手动去找maven 依赖的顺序 还是比较蛋疼的  毕竟大项目里面pom文件的数量你懂的
下面以idea为例  下面是打开idea model 找到设置内 依赖设置的顺序  里面可以找到该依赖是由那个 model 引用 何时加载  这样就很好找到问题

 ![源码](https://raw.githubusercontent.com/MrDLontheway/mrdlontheway.github.io/master/images/mavenproject.png)

这里我具体的问题是因为  我的项目A 内 引用了 框架B 的源码   然后我的项目C 内也引用了B 并且复写了B的部分源码  比如我复写了B 的class Demo

这时候 我的项目C 需要加载Demo类的时候 因为在pom引用的 A 在 C 的前面  所以先从A 项目中拿了 B框架中Demo的源码 到我复写C 项目的时候已经加载过了就不需要再次加载了

所以解决方法就是把我的C 加载顺序 调整到A的前面 这里有以下解决方法 

1.将C 顺序调整到A的前面（不建议 因为更改的pom顺序可能会引发其他的jar 冲突问题）

2.也就是我这次解决方案  因为A项目中的 yilai.jar 优先加载了  我只需要在A 中将 yilai.jar这个model 排除掉 使用exclusion 标签
这样在加载依赖时候因为项目A内排除了框架源码  这样就会使用下面的 项目C的复写源码  


经过这次的问题 也了解了maven的加载机制.