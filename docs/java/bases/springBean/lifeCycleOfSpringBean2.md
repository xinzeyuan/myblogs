# URL
  - https://www.cnblogs.com/zrtqsk/p/3735273.html

Spring Bean的生命周期（非常详细）
Spring作为当前Java最流行、最强大的轻量级框架，受到了程序员的热烈欢迎。准确的了解Spring Bean的生命周期是非常必要的。我们通常使用ApplicationContext作为Spring容器。这里，我们讲的也是 ApplicationContext中Bean的生命周期。而实际上BeanFactory也是差不多的，只不过处理器需要手动注册。


一、生命周期流程图：

　　Spring Bean的完整生命周期从创建Spring容器开始，直到最终Spring容器销毁Bean，这其中包含了一系列关键点。


   ![](181453414212066.png)

   ![](181454040628981.png)

若容器注册了以上各种接口，程序那么将会按照以上的流程进行。下面将仔细讲解各接口作用。

 

# 二、各种接口方法分类

Bean的完整生命周期经历了各种方法调用，这些方法可以划分为以下几类：

* 1、Bean自身的方法　　：　　这个包括了Bean本身调用的方法和通过配置文件中<bean>的init-method和destroy-method指定的方法

* 2、Bean级生命周期接口方法　　：　　这个包括了BeanNameAware、BeanFactoryAware、InitializingBean和DiposableBean这些接口的方法

* 3、容器级生命周期接口方法　　：　　这个包括了InstantiationAwareBeanPostProcessor 和 BeanPostProcessor 这两个接口实现，一般称它们的实现类为“后处理器”。

* 4、工厂后处理器接口方法　　：　　这个包括了AspectJWeavingEnabler, ConfigurationClassPostProcessor, CustomAutowireConfigurer等等非常有用的工厂后处理器　　接口的方法。工厂后处理器也是容器级的。在应用上下文装配配置文件之后立即调用。

　　

三、演示

我们用一个简单的Spring Bean来演示一下Spring Bean的生命周期。

1、首先是一个简单的Spring Bean，调用Bean自身的方法和Bean级生命周期接口方法，为了方便演示，它实现了BeanNameAware、BeanFactoryAware、InitializingBean和DiposableBean这4个接口，同时有2个方法，对应配置文件中<bean>的init-method和destroy-method。如下：

 

````
 1 package springBeanTest;
 2 
 3 import org.springframework.beans.BeansException;
 4 import org.springframework.beans.factory.BeanFactory;
 5 import org.springframework.beans.factory.BeanFactoryAware;
 6 import org.springframework.beans.factory.BeanNameAware;
 7 import org.springframework.beans.factory.DisposableBean;
 8 import org.springframework.beans.factory.InitializingBean;
 9 
10 /**
11  * @author qsk
12  */
13 public class Person implements BeanFactoryAware, BeanNameAware,
14         InitializingBean, DisposableBean {
15 
16     private String name;
17     private String address;
18     private int phone;
19 
20     private BeanFactory beanFactory;
21     private String beanName;
22 
23     public Person() {
24         System.out.println("【构造器】调用Person的构造器实例化");
25     }
26 
27     public String getName() {
28         return name;
29     }
30 
31     public void setName(String name) {
32         System.out.println("【注入属性】注入属性name");
33         this.name = name;
34     }
35 
36     public String getAddress() {
37         return address;
38     }
39 
40     public void setAddress(String address) {
41         System.out.println("【注入属性】注入属性address");
42         this.address = address;
43     }
44 
45     public int getPhone() {
46         return phone;
47     }
48 
49     public void setPhone(int phone) {
50         System.out.println("【注入属性】注入属性phone");
51         this.phone = phone;
52     }
53 
54     @Override
55     public String toString() {
56         return "Person [address=" + address + ", name=" + name + ", phone="
57                 + phone + "]";
58     }
59 
60     // 这是BeanFactoryAware接口方法
61     @Override
62     public void setBeanFactory(BeanFactory arg0) throws BeansException {
63         System.out
64                 .println("【BeanFactoryAware接口】调用BeanFactoryAware.setBeanFactory()");
65         this.beanFactory = arg0;
66     }
67 
68     // 这是BeanNameAware接口方法
69     @Override
70     public void setBeanName(String arg0) {
71         System.out.println("【BeanNameAware接口】调用BeanNameAware.setBeanName()");
72         this.beanName = arg0;
73     }
74 
75     // 这是InitializingBean接口方法
76     @Override
77     public void afterPropertiesSet() throws Exception {
78         System.out
79                 .println("【InitializingBean接口】调用InitializingBean.afterPropertiesSet()");
80     }
81 
82     // 这是DiposibleBean接口方法
83     @Override
84     public void destroy() throws Exception {
85         System.out.println("【DiposibleBean接口】调用DiposibleBean.destory()");
86     }
87 
88     // 通过<bean>的init-method属性指定的初始化方法
89     public void myInit() {
90         System.out.println("【init-method】调用<bean>的init-method属性指定的初始化方法");
91     }
92 
93     // 通过<bean>的destroy-method属性指定的初始化方法
94     public void myDestory() {
95         System.out.println("【destroy-method】调用<bean>的destroy-method属性指定的初始化方法");
96     }
97 }
````
 

2、接下来是演示BeanPostProcessor接口的方法，如下：

````
 1 package springBeanTest;
 2 
 3 import org.springframework.beans.BeansException;
 4 import org.springframework.beans.factory.config.BeanPostProcessor;
 5 
 6 public class MyBeanPostProcessor implements BeanPostProcessor {
 7 
 8     public MyBeanPostProcessor() {
 9         super();
10         System.out.println("这是BeanPostProcessor实现类构造器！！");
11         // TODO Auto-generated constructor stub
12     }
13 
14     @Override
15     public Object postProcessAfterInitialization(Object arg0, String arg1)
16             throws BeansException {
17         System.out
18         .println("BeanPostProcessor接口方法postProcessAfterInitialization对属性进行更改！");
19         return arg0;
20     }
21 
22     @Override
23     public Object postProcessBeforeInitialization(Object arg0, String arg1)
24             throws BeansException {
25         System.out
26         .println("BeanPostProcessor接口方法postProcessBeforeInitialization对属性进行更改！");
27         return arg0;
28     }
29 }
````

如上，BeanPostProcessor接口包括2个方法postProcessAfterInitialization和postProcessBeforeInitialization，这两个方法的第一个参数都是要处理的Bean对象，第二个参数都是Bean的name。返回值也都是要处理的Bean对象。这里要注意。

 

3、InstantiationAwareBeanPostProcessor 接口本质是BeanPostProcessor的子接口，一般我们继承Spring为其提供的适配器类InstantiationAwareBeanPostProcessor Adapter来使用它，如下：

````
 1 package springBeanTest;
 2 
 3 import java.beans.PropertyDescriptor;
 4 
 5 import org.springframework.beans.BeansException;
 6 import org.springframework.beans.PropertyValues;
 7 import org.springframework.beans.factory.config.InstantiationAwareBeanPostProcessorAdapter;
 8 
 9 public class MyInstantiationAwareBeanPostProcessor extends
10         InstantiationAwareBeanPostProcessorAdapter {
11 
12     public MyInstantiationAwareBeanPostProcessor() {
13         super();
14         System.out
15                 .println("这是InstantiationAwareBeanPostProcessorAdapter实现类构造器！！");
16     }
17 
18     // 接口方法、实例化Bean之前调用
19     @Override
20     public Object postProcessBeforeInstantiation(Class beanClass,
21             String beanName) throws BeansException {
22         System.out
23                 .println("InstantiationAwareBeanPostProcessor调用postProcessBeforeInstantiation方法");
24         return null;
25     }
26 
27     // 接口方法、实例化Bean之后调用
28     @Override
29     public Object postProcessAfterInitialization(Object bean, String beanName)
30             throws BeansException {
31         System.out
32                 .println("InstantiationAwareBeanPostProcessor调用postProcessAfterInitialization方法");
33         return bean;
34     }
35 
36     // 接口方法、设置某个属性时调用
37     @Override
38     public PropertyValues postProcessPropertyValues(PropertyValues pvs,
39             PropertyDescriptor[] pds, Object bean, String beanName)
40             throws BeansException {
41         System.out
42                 .println("InstantiationAwareBeanPostProcessor调用postProcessPropertyValues方法");
43         return pvs;
44     }
45 }
````

这个有3个方法，其中第二个方法postProcessAfterInitialization就是重写了BeanPostProcessor的方法。第三个方法postProcessPropertyValues用来操作属性，返回值也应该是PropertyValues对象。

 

4、演示工厂后处理器接口方法，如下：

````
 1 package springBeanTest;
 2 
 3 import org.springframework.beans.BeansException;
 4 import org.springframework.beans.factory.config.BeanDefinition;
 5 import org.springframework.beans.factory.config.BeanFactoryPostProcessor;
 6 import org.springframework.beans.factory.config.ConfigurableListableBeanFactory;
 7 
 8 public class MyBeanFactoryPostProcessor implements BeanFactoryPostProcessor {
 9 
10     public MyBeanFactoryPostProcessor() {
11         super();
12         System.out.println("这是BeanFactoryPostProcessor实现类构造器！！");
13     }
14 
15     @Override
16     public void postProcessBeanFactory(ConfigurableListableBeanFactory arg0)
17             throws BeansException {
18         System.out
19                 .println("BeanFactoryPostProcessor调用postProcessBeanFactory方法");
20         BeanDefinition bd = arg0.getBeanDefinition("person");
21         bd.getPropertyValues().addPropertyValue("phone", "110");
22     }
23 
24 }
````
 

5、配置文件如下beans.xml，很简单，使用ApplicationContext,处理器不用手动注册：

````
<?xml version="1.0" encoding="UTF-8"?>

<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:p="http://www.springframework.org/schema/p"
    xmlns:aop="http://www.springframework.org/schema/aop" xmlns:tx="http://www.springframework.org/schema/tx"
    xsi:schemaLocation="
            http://www.springframework.org/schema/beans 
            http://www.springframework.org/schema/beans/spring-beans-3.2.xsd">

    <bean id="beanPostProcessor" class="springBeanTest.MyBeanPostProcessor">
    </bean>

    <bean id="instantiationAwareBeanPostProcessor" class="springBeanTest.MyInstantiationAwareBeanPostProcessor">
    </bean>

    <bean id="beanFactoryPostProcessor" class="springBeanTest.MyBeanFactoryPostProcessor">
    </bean>
    
    <bean id="person" class="springBeanTest.Person" init-method="myInit"
        destroy-method="myDestory" scope="singleton" p:name="张三" p:address="广州"
        p:phone="15900000000" />

</beans>
````
 

 

6、下面测试一下：

````
 1 package springBeanTest;
 2 
 3 import org.springframework.context.ApplicationContext;
 4 import org.springframework.context.support.ClassPathXmlApplicationContext;
 5 
 6 public class BeanLifeCycle {
 7 
 8     public static void main(String[] args) {
 9 
10         System.out.println("现在开始初始化容器");
11         
12         ApplicationContext factory = new ClassPathXmlApplicationContext("springBeanTest/beans.xml");
13         System.out.println("容器初始化成功");    
14         //得到Preson，并使用
15         Person person = factory.getBean("person",Person.class);
16         System.out.println(person);
17         
18         System.out.println("现在开始关闭容器！");
19         ((ClassPathXmlApplicationContext)factory).registerShutdownHook();
20     }
21 }
````

关闭容器使用的是实际是AbstractApplicationContext的钩子方法。

我们来看一下结果：

````
现在开始初始化容器
2014-5-18 15:46:20 org.springframework.context.support.AbstractApplicationContext prepareRefresh
信息: Refreshing org.springframework.context.support.ClassPathXmlApplicationContext@19a0c7c: startup date [Sun May 18 15:46:20 CST 2014]; root of context hierarchy
2014-5-18 15:46:20 org.springframework.beans.factory.xml.XmlBeanDefinitionReader loadBeanDefinitions
信息: Loading XML bean definitions from class path resource [springBeanTest/beans.xml]
这是BeanFactoryPostProcessor实现类构造器！！
BeanFactoryPostProcessor调用postProcessBeanFactory方法
这是BeanPostProcessor实现类构造器！！
这是InstantiationAwareBeanPostProcessorAdapter实现类构造器！！
2014-5-18 15:46:20 org.springframework.beans.factory.support.DefaultListableBeanFactory preInstantiateSingletons
信息: Pre-instantiating singletons in org.springframework.beans.factory.support.DefaultListableBeanFactory@9934d4: defining beans [beanPostProcessor,instantiationAwareBeanPostProcessor,beanFactoryPostProcessor,person]; root of factory hierarchy
InstantiationAwareBeanPostProcessor调用postProcessBeforeInstantiation方法
【构造器】调用Person的构造器实例化
InstantiationAwareBeanPostProcessor调用postProcessPropertyValues方法
【注入属性】注入属性address
【注入属性】注入属性name
【注入属性】注入属性phone
【BeanNameAware接口】调用BeanNameAware.setBeanName()
【BeanFactoryAware接口】调用BeanFactoryAware.setBeanFactory()
BeanPostProcessor接口方法postProcessBeforeInitialization对属性进行更改！
【InitializingBean接口】调用InitializingBean.afterPropertiesSet()
【init-method】调用<bean>的init-method属性指定的初始化方法
BeanPostProcessor接口方法postProcessAfterInitialization对属性进行更改！
InstantiationAwareBeanPostProcessor调用postProcessAfterInitialization方法
容器初始化成功
Person [address=广州, name=张三, phone=110]
现在开始关闭容器！
【DiposibleBean接口】调用DiposibleBean.destory()
【destroy-method】调用<bean>的destroy-method属性指定的初始化方法
````