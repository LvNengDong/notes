#### depends-on 提前初始化

可以使某个 bean 在创建前，先创建其它的 bean。



#### lazy



## 循环依赖复现



当Spring容器在创建A时，会发现其引用了B，从而会先去创建B。同样的，创建B时，会先去创建C，而创建C时，又先去创建A。最后A、B、C之间互相等待，谁都没法创建成功



### 构造器注入循环依赖

在 Spring 或 SpringBoot 项目中，创建 3 个对象 A、B 和 C，三个类互相引用。

```xml
    <!--
        当循环依赖的bean都是通过构造器注入依赖的时候，无论这些 bean 是 singleton 还是 prototype，
        都会因为循环依赖而导致创建 bean 失败
    -->
    <!-- Case1：构造器注入+单例-->
    <bean id="A" class="com.lnd.demo01.A"> <!-- scope 默认就是 singleton-->
        <constructor-arg ref="B"/>
    </bean>

    <bean id="B" class="com.lnd.demo01.B">
        <constructor-arg ref="C"/>
    </bean>
    <bean id="C" class="com.lnd.demo01.C">
        <constructor-arg ref="A"/>
    </bean>
    
    <!-- Case2：构造器注入+原型-->
    <bean id="A" class="com.lnd.demo01.A" scope="prototype">
        <constructor-arg ref="B"/>
    </bean>

    <bean id="B" class="com.lnd.demo01.B" scope="prototype">
        <constructor-arg ref="C"/>
    </bean>
    <bean id="C" class="com.lnd.demo01.C" scope="prototype">
        <constructor-arg ref="A"/>
    </bean>
```



当项目启动时，我们会看到如下错误信息：

```
Description:

The dependencies of some of the beans in the application context form a cycle:

┌─────┐
|  a (field private com.lnd.springboot01.demo01.B com.lnd.springboot01.demo01.A.b)
↑     ↓
|  b (field private com.lnd.springboot01.demo01.C com.lnd.springboot01.demo01.B.c)
↑     ↓
|  c (field private com.lnd.springboot01.demo01.A com.lnd.springboot01.demo01.C.a)
└─────┘

```

当循环依赖的 bean 都是通过构造器注入依赖的时候，无论这些 bean 是 singleton 还是prototype，在获取 bean 的时候都会失败。



### 通过属性注入循环依赖

#### 结论

- 循环依赖的 bean 都是 singleton 成功；
- 循环依赖的 bean 都是 prototype 失败；
- 同时有 singleton 和 prototype 时，先获取的那个 bean 是 singleton 时，就会成功，否则失败。



#### 循环依赖的bean都是singleton

```xml
<!-- 通过属性注入的循环依赖 -->
<bean id="A" class="com.lnd.demo01.A">
    <property name="b" ref="B"/>
</bean>

<bean id="B" class="com.lnd.demo01.B">
    <property name="c" ref="C"/>
</bean>
<bean id="C" class="com.lnd.demo01.C">
    <property name="a" ref="A"/>
</bean>
```

实验结果：bean 可以被正常创建出来

注意事项：注意此时 set 方法和构造方法不要同时存在，因为构造方法的优先级会更高。

#### 循环依赖的bean都是prototype

```xml
<bean id="A" class="com.lnd.demo01.A" scope="prototype">
    <property name="b" ref="B"/>
</bean>

<bean id="B" class="com.lnd.demo01.B" scope="prototype">
    <property name="c" ref="C"/>
</bean>
<bean id="C" class="com.lnd.demo01.C" scope="prototype">
    <property name="a" ref="A"/>
</bean>
```



#### 同时有 singleton 和 prototype

```xml
<!--Case3：选择性成功-->
<bean id="A" class="com.lnd.demo01.A" scope="singleton">
    <property name="b" ref="B"/>
</bean>

<bean id="B" class="com.lnd.demo01.B" scope="prototype">
    <property name="c" ref="C"/>
</bean>
<bean id="C" class="com.lnd.demo01.C" scope="prototype">
    <property name="a" ref="A"/>
</bean>
```

同时有 singleton 和 prototype 时，Spring 会按照配置文件从上到下的顺序创建 bean。

- 比如上面这个配置文件，最先创建的 bean A 是单例的，所以可以成功，
- 如果将顺序调整如下，让最先创建的 bean 不是单例，则会创建失败。

```xml
<bean id="A" class="com.lnd.demo01.A" scope="prototype">
    <property name="b" ref="B"/>
</bean>

<bean id="B" class="com.lnd.demo01.B" scope="singleton">
    <property name="c" ref="C"/>
</bean>
<bean id="C" class="com.lnd.demo01.C" scope="singleton">
    <property name="a" ref="A"/>
</bean>
```



