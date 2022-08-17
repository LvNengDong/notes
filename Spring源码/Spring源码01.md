# IOC

# Spring中bean的创建流程

## 全局概览

Spring 中创建 bean 的起点是从 XML 配置文件开始的，我们一般都会将 bean 的定义信息写在 XML 文件中，在 Spring 启动时将 XML 加载到内存中并解析，解析得到的内容会被封装成 BeanDefinition 对象，根据 beanDefinition 信息实例化对象，最后把实例对象放到容器中。这样，我们在程序中需要使用的地方，通过或 getBean 方法，或 @Autowired，或 @Bean 等注解直接从容器中获取 bean 对象即可。

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

首先，我们得先知道为什么要后修改 BeanDefinition 信息，直接让用户一次性敲定最终配置信息不香么？

其实这个答案非常简单，为了方便用户（程序员）使用，Spring 往往会简化用户要编辑的配置信息，让用户只需要关注不同，而对于一些通用的信息，可以在得到 BeanDefinition 之后，创建 bean 之前后加进去即可。这是一种比较常见的方式，而要实现这一功能，就要求 Spring 提供能够随时修改 beanDefinition 信息的入口，也就是——BeanFactoryPostProcessor。



## PostProcessor（后处理器/增强器）

PostProcessor 的概念在 Spring 中非常常见，顾名思义，你可以称其为“后处理器”，或者“增强器”。它的作用就是在你完成某些工作后，额外对其进行增强。Spring 中最常见的 PostProcessor 有两个，分别是 BeanFactoryPostProcessor 和 BeanPostProcessor。

BeanFactoryPostProcessor 的作用是增强 beanDefinition 的信息；BeanPostProcessor 的作用是增强 bean 的信息。

所以，在根据 BeanDefinition 创建实例 bean 的过程中，不可避免地会被很多 BeanFactoryPostProcessor 进行后处理，最终得到的 bean 对象与配置文件中的定义相比，会多出很多默认的、公共的属性。





## 创建bean实例，真的如你想象中那么简单吗？

在使用一系列 BeanFactoryPostProcessor 处理后，BeanDefinition 信息终于来到了最终版，接下来，我们就可以根据 beanFactoryPostProcess 信息通过反射的方式创建 bean 了。

但是，创建 bean 的过程又划分成多个步骤，常见的有：

1. **实例化 bean**：分配堆内存，创建初始对象，此时对象中的属性均为默认值。

2. **填充属性**：根据 beanDefinition 信息为实例 bean 填充属性。

3. **设置 Aware 接口**：Aware 是“感知”的意思，这一步就是让实例 bean 能感知到设置的 Aware 接口中的属性。

4. **BeanPostProcessor#before**：在完成上面步骤后，如果需要对 bean 进行增强，还可以调用 BeanPostProcessor 的 before 方法进行处理。顾名思义，既然有 before 方法，那一定还会有一个 after 方法，before 和 after 是相较于 init 而言的，before 方法在 init-method 方法之前执行，而 after 在其之后执行。

5. **init-method**：在 XML 配置文件中，我们还可以为每个 bean 配置一个 init-method 方法，这个方法将会在 bean 创建完成之后执行。其配置方式如下：

   ```xml
   <!--待创建的bean对象-->
   <bean id="自定义id" class="com.qunar.springmybatis.model.Employee" init-method="myInit">
   
   </bean>
   ```

6. **BeanPostProcessor#after**：与上面的 before 方法类似，after 方法是在 init-method 方法创建完成之后执行的，也是一个 bean 的增强方法

![image-20220815100318027](https://raw.githubusercontent.com/LvNengDong/pic-go/main/img/spring/image-20220815100318027.png)

得到了最终 bean 对象后，就可以把实例 bean 放到容器中了，之后在使用时通过 getBean 获取即可。

![image-20220815103435862](https://raw.githubusercontent.com/LvNengDong/pic-go/main/img/spring/image-20220815103435862.png)



## 补充1：Aware 接口到底有什么作用？

**问题：**现有一个普通对象 A，如果开发者想在使用 A 对象的过程中获取容器中的一些属性应该怎么办？

**解决：**

-  实现 Aware 接口：

```java
import org.springframework.beans.factory.BeanNameAware;

public class A implements BeanNameAware {

    private String beanName;

    @Override
    public void setBeanName(String name) {
        this.beanName = name;
    }

    public void show(){
        System.out.println(">>>>>>>>>>>" + this.beanName);
    }
}
```

- 注册成为 bean

```xml
<bean id="helloService" class="cn.lnd.service.HelloService"></bean>
```

- 测试

```java
public static void main(String[] args) {
    // 1、获取容器：解析配置文件，生成容器实例
    String xmlPath = "classpath:spring/spring-config.xml";
    ApplicationContext context = new FileSystemXmlApplicationContext(xmlPath);

    // 2、获取容器中的bean实例
    HelloService helloService = context.getBean(HelloService.class);
    helloService.show(); // >>>>>>>>>>>helloService
}
```

**分析：**

A 是一个普通的对象，它是不知道自己的 beanName 的，如果我们想让实例A能够感知到自己的 beanName，就可以让 A 实现 BeanNameAware 接口，并重新 setBeanName 方法，这个方法会从容器中拿到当前 bean 的 name 属性，之后我们就可以对这个 name 随意操作了。

同理，我们还可以实现 EnvironmentAware, ApplicationContextAware 等来获取当前的上下文环境或整个容器。

 

**总结：**当 Spring 容器中创建的 bean 对象在进行具体操作时需要使用到容器中的其它对象或容器中的某些属性时，可以让对象实现 Aware 接口，来满足业务需求。



## 补充2：在不同的阶段，要处理不同的工作，怎么办？——观察者模式

> 观察者模式
>
> - 监听器
> - 监听事件
> - 多播器（广播器）



## 补充3：BeanFactory 和 FactoryBean 的区别

共同点：都是用来创建对象的

FactoryBean 是一个工厂，它是通过工厂对象来创建实例 bean 的，而不需要遵循 Spring 创建 bean 的生命周期。

当使用 BeanFactory 创建对象的时候必须要遵循完成的 bean 创建流程，这个过程是由 Spring 来管理控制的。而使用 FactoryBean 只需要调用 getObject 就可以返回具体的对象，整个对象的创建过程是由用户自己来控制的，更加灵活。





----



![image-20220812171119901](https://raw.githubusercontent.com/LvNengDong/pic-go/main/img/spring/image-20220812171119901.png)



01：00

![image-20220812173111758](https://raw.githubusercontent.com/LvNengDong/pic-go/main/img/spring/image-20220812173111758.png)



# Debug Spring 流程概览

想当然的，Spring 启动的流程应该是从加载 XML 配置文件开始；实际则不然，根据配置文件生成的这些 bean 需要被放到一个容器中，那么 Spring 启动的第一步应该是创建这个容器——BeanFactory。



加载 XML 文件



BeanFactoryProcessor 后处理器



**实例化 bean 之前的准备工作**

在实例化 bean  之前，还有一系列的准备工作。

1. 比如，在实例化的时候我们会使用到 beanPostProcessor，那我们就需要先准备好；
2. 其次，如果我们想在对象创建的不同阶段触发一些不同的事件，那我们还需要准备一些监听器、监听器事件以及多播器（广播器）实例。
3. 最后，还有一个模板方法，供子类去扩展

对应在源码中的方法分别是：

```java
// 向容器中注入 beanPostProcessor 实例，用于对 bean 进行增强
>> registerBeanPostProcessors(beanFactory);
// 国际化
>> initMessageSource();
// 初始化事件多播器（广播器）
>> initApplicationEventMulticaster();
// 模板方法，默认什么都不做，交给子类去实现
>> onRefresh();
// 注册监听器
>> registerListeners();
```



**具体的实例化操作**

```xml
<bean id="welcomeService" class="cn.lnd.service.impl.WelcomeServiceImpl" abstract="true"></bean>
```



# Spring的启动流程细节分析



