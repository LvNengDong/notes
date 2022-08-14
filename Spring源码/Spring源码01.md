# IOC

# Spring中bean创建的流程

## 全局概览

Spring 中创建 bean 的起点是从 XML 配置文件开始的，我们一般都会将 bean 的定义信息写在 XML 文件中，在 Spring 启动时将 XML 加载到内存中，并解析 XML，将其封装成 BeanDefinition 对象，根据 beanDefinition 的信息实例化对象，最后把实例化的对象放到容器中。这样，我们在程序中需要使用的地方，通过或 getBean 方法，或 @Autowired，或 @Bean 等注解直接从容器中获取 bean 对象即可。

 ![image-20220814233204020](https://raw.githubusercontent.com/LvNengDong/pic-go/main/img/spring/image-20220814233204020.png)



这里我们说的容器，实际上就是一个 Map 集合，对于这个容器，我们最常见的获取 bean 的方法有两种，其一是根据 String 类型的 id 获取 bean 实例，其二是根据 Class 类型获取 bean 实例，这对应了常见的 getXxxByName 和 getXxxByType 方法。但实际上，key 的类型除了这两种外，还有几类不太常见的 key 类型，比如：ObjectFactory 和 BeanDefinition。

![image-20220814234443148](https://raw.githubusercontent.com/LvNengDong/pic-go/main/img/spring/image-20220814234443148.png)

从开头到现在，我们已经提到两次 BeanDefinition 了，那么，BeanDefinition 到底是什么呢？我认为可以这样理解：

> **BeanDefinition：**
>
> 我们在xml配置文件中写的信息都是字符串字面量，对Java程序而言是无法直接处理的，而Java程序可以直接处理Java中的对象，如果我们能把这些字符串信息按照一定的规则封装成一个Java对象，那程序就可以很方便的调用这个封装后的对象。BeanDefinition就是这样的一个Java对象，我们在程序中可以直接使用Java对象中的属性、方法等。



---

## 加载、解析配置文件——BeanDefinitionReader

在 Spring 中，我们最常见的做法就是把 bean 的定义信息写在 XML 文件中，让 Spring 在启动时加载。当然，我们的想法不能太过狭隘，除了 bean 的定义信息外，我们还会加载其它一些外部信息到 Spring 容器中，比如 SpringBoot 常见的 properties 和 yaml 信息，甚至还可以自定义一些其它类型的外部配置信息，比如 json。

那么这就会产生一个问题，如何解析不同类型的配置文件。答案其实也很简单——就是定义一个高层次的抽象类或接口，为每种不同的配置文件格式开发不同的实现类。

![image-20220815000819319](https://raw.githubusercontent.com/LvNengDong/pic-go/main/img/spring/image-20220815000819319.png)

Spring 采用的是接口的方式，它提供了一个 BeanDefinitionReader 接口，该接口的作用就是：解析配置文件，将其转换成一个 BeanDefinition 对象。可以看到，对于不同的配置文件（比如 xml、properties 和 groovy 文件）都提供了对应的实现类。

![image-20220812165435656](https://raw.githubusercontent.com/LvNengDong/pic-go/main/img/spring/image-20220812165435656.png)

## new ? 反射——Spring创建bean的方式

在得到了 BeanDefinition 对象之后，Spring 就可以根据 BeanDefinition 中的信息来创建 bean 对象了。与传统的通过 new 创建对象的方式不同，Spring 选择了反射的方式来创建 bean 对象。通过反射创建对象常见的几种方式如下：

![image-20220815002219917](https://raw.githubusercontent.com/LvNengDong/pic-go/main/img/spring/image-20220815002219917.png)



## 扩展点1：我想在创建bean时修改BeanDefinition怎么办？

我们知道，BeanDefinition 就是某个配置文件的 Java 对象映射，并且 Spring 会根据 BeanDefinition 去创建对应的实例 bean。那么，如果想要在创建 bean 之前动态的修改 bean 的信息该怎么办呢？

首先，我们得先知道为什么要存在这个扩展点，直接让用户一次性敲定最终配置信息不香么？

其实这个答案非常简单，为了方便用户（程序员）使用，Spring 往往会简化用户要编辑的配置信息，让用户只需要关注不同，而对于一些通用的信息，可以在得到 BeanDefinition 之后，创建 bean 之前后加进去。这是一种比较常见的方式，而要实现这一功能，就要求 Spring 提供能够随时修改 beanDefinition 信息的入口，也就是——BeanFactoryPostProcessor。

## PostProcessor（后处理器/增强器）

PostProcessor 的概念在 Spring 中非常常见，顾名思义，你可以称其为“后处理器”，或者“增强器”。它的作用就是在你完成某些工作后，额外对其进行增强。Spring 中最常见的 PostProcessor 有两个，分别是 BeanFactoryPostProcessor 和 BeanPostProcessor。

BeanFactoryPostProcessor 的作用是增强 beanDefinition 的信息；BeanPostProcessor 的作用是增强 bean 的信息。

所以，在根据 BeanDefinition 创建实例 bean 的过程中，不可避免地会被很多 BeanFactoryPostProcessor 进行后处理，最终得到的 bean 对象与配置文件中的定义相比，会多出很多默认的、公共的属性。



## 实例化bean，真的如你想象中那么简单吗？

![image-20220812171119901](https://raw.githubusercontent.com/LvNengDong/pic-go/main/img/spring/image-20220812171119901.png)



01：00

![image-20220812173111758](https://raw.githubusercontent.com/LvNengDong/pic-go/main/img/spring/image-20220812173111758.png)





![image-20220812182106354](https://raw.githubusercontent.com/LvNengDong/pic-go/main/img/spring/image-20220812182106354.png)

