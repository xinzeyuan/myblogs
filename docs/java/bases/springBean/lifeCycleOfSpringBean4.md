# URL
  - https://www.jianshu.com/p/1417eefd2ab1
  
  
#  Spring探秘|妙用BeanPostProcessor


最近，在给项目组使用Spring搭建Java项目基础框架时，发现使用Spring提供的BeanPostProcessor可以很简单方便地解决很多看起来有点难解决的问题。本文将会通过一个真实案例来阐述BeanPostProcessor的用法

# BeanPostProcessor简介
BeanPostProcessor是Spring IOC容器给我们提供的一个扩展接口。接口声明如下：

````
public interface BeanPostProcessor {
    //bean初始化方法调用前被调用
    Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException;
    //bean初始化方法调用后被调用
    Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException;

}
````

如上接口声明所示，BeanPostProcessor接口有两个回调方法。当一个BeanPostProcessor的实现类注册到Spring IOC容器后，对于该Spring IOC容器所创建的每个bean实例在初始化方法（如afterPropertiesSet和任意已声明的init方法）调用前，将会调用BeanPostProcessor中的postProcessBeforeInitialization方法，而在bean实例初始化方法调用完成后，则会调用BeanPostProcessor中的postProcessAfterInitialization方法，整个调用顺序可以简单示意如下：

--> Spring IOC容器实例化Bean
--> 调用BeanPostProcessor的postProcessBeforeInitialization方法
--> 调用bean实例的初始化方法
--> 调用BeanPostProcessor的postProcessAfterInitialization方法

可以看到，Spring容器通过BeanPostProcessor给了我们一个机会对Spring管理的bean进行再加工。比如：我们可以修改bean的属性，可以给bean生成一个动态代理实例等等。一些Spring AOP的底层处理也是通过实现BeanPostProcessor来执行代理包装逻辑的。

# BeanPostProcessor实战
了解了BeanPostProcessor的相关知识后，下面我们来通过项目中的一个具体例子来体验一下它的神奇功效吧。

先介绍一下我们的项目背景吧：我们项目中经常会涉及AB 测试，这就会遇到同一套接口会存在两种不同实现。实验版本与对照版本需要在运行时同时存在。下面用一些简单的类来做一个示意：


````
public class HelloService{
     void sayHello();
     void sayHi();
}
````

HelloService有以下两个版本的实现：

````
@Service
public class HelloServiceImplV1 implements HelloService{
     public void sayHello(){
          System.out.println("Hello from V1");
     }
     public void sayHi(){
          System.out.println("Hi from V1");
     }
}
@Service
public class HelloServiceImplV2 implements HelloService{
     public void sayHello(){
          System.out.println("Hello from V2");
     }
     public void sayHi(){
          System.out.println("Hi from V2");
     }
}
````

做AB测试的话，在使用BeanPostProcessor封装前，我们的调用代码大概是像下面这样子的：

````
@Controller
public class HelloController{
     @Autowird
     private HelloServiceImplV1 helloServiceImplV1；
     @Autowird
     private HelloServiceImplV2 helloServiceImplV2；

     public void sayHello(){
          if(getHelloVersion()=="A"){
               helloServiceImplV1.sayHello();
          }else{
               helloServiceImplV2.sayHello();
          }
     }
     public void sayHi(){
          if(getHiVersion()=="A"){
               helloServiceImplV1.sayHi();
          }else{
               helloServiceImplV2.sayHi();
          }
     }
}
````

可以看到，这样的代码看起来十分不优雅，并且如果AB测试的功能点很多的话，那项目中就会充斥着大量的这种重复性分支判断，看到代码就想死有木有！！！维护代码也将会是个噩梦。比如某个功能点AB测试完毕，需要把全部功能切换到V2版本，V1版本不再需要维护，那么处理方式有两种：

把A版本代码留着不管：这将会导致到处都是垃圾代码从而造成代码臃肿难以维护
找到所有V1版本被调用的地方然后把相关分支删掉：这很容易在处理代码的时候删错代码从而造成生产事故。
怎么解决这个问题呢，我们先看代码，后文再给出解释：

````
@Target({ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component
public @interface RoutingInjected{
}
@Target({ElementType.FIELD,ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component
public @interface RoutingSwitch{
    /**
     * 在配置系统中开关的属性名称，应用系统将会实时读取配置系统中对应开关的值来决定是调用哪个版本
     * @return
     */
     String value() default "";
}

@RoutingSwitch("hello.switch")
public class HelloService{

    @RoutingSwitch("A")
    void sayHello();

    void sayHi();
}
@Controller
public class HelloController{
   
    @RoutingInjected
    private HelloService helloService；
    
    public void sayHello(){
        this.helloService.sayHello();
    }

    public void sayHi(){
        this.helloService.sayHi();
    }
}
````

现在我们可以停下来对比一下封装前后调用代码了，是不是感觉改造后的代码优雅很多呢？那么这是怎么实现的呢，我们一起来揭开它的神秘面纱吧，请看代码：


````
@Component
public class RoutingBeanPostProcessor implements BeanPostProcessor {

    @Autowired
    private ApplicationContext applicationContext;

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        return bean;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        Class clazz = bean.getClass();
        Field[] fields = clazz.getDeclaredFields();
        for (Field f : fields) {
            if (f.isAnnotationPresent(RoutingInjected.class)) {
                if (!f.getType().isInterface()) {
                    throw new BeanCreationException("RoutingInjected field must be declared as an interface:" + f.getName()
                                    + " @Class " + clazz.getName());
                }
                try {
                    this.handleRoutingInjected(f, bean, f.getType());
                } catch (IllegalAccessException e) {
                    throw new BeanCreationException("Exception thrown when handleAutowiredRouting", e);
                }
            }
        }
        return bean;
    }

    private void handleRoutingInjected(Field field, Object bean, Class type) throws IllegalAccessException {
        Map<String, Object> candidates = this.applicationContext.getBeansOfType(type);
        field.setAccessible(true);
        if (candidates.size() == 1) {
            field.set(bean, candidates.values().iterator().next());
        } else if (candidates.size() == 2) {
            Object proxy = RoutingBeanProxyFactory.createProxy(type, candidates);
            field.set(bean, proxy);
        } else {
            throw new IllegalArgumentException("Find more than 2 beans for type: " + type);
        }
    }
}

public class RoutingBeanProxyFactory {

    public static Object createProxy(Class targetClass, Map<String, Object> beans) {
        ProxyFactory proxyFactory = new ProxyFactory();
        proxyFactory.setInterfaces(targetClass);
        proxyFactory.addAdvice(new VersionRoutingMethodInterceptor(targetClass, beans));
        return proxyFactory.getProxy();
    }
    static class VersionRoutingMethodInterceptor implements MethodInterceptor {
        private String classSwitch;
        private Object beanOfSwitchOn;
        private Object beanOfSwitchOff;

        public VersionRoutingMethodInterceptor(Class targetClass, Map<String, Object> beans) {
            String interfaceName = StringUtils.uncapitalize(targetClass.getSimpleName());
            if(targetClass.isAnnotationPresent(RoutingSwitch.class)){
                this.classSwitch =((RoutingSwitch)targetClass.getAnnotation(RoutingSwitch.class)).value();
            }
            this.beanOfSwitchOn = beans.get(this.buildBeanName(interfaceName, true));
            this.beanOfSwitchOff = beans.get(this.buildBeanName(interfaceName, false));
        }
        
        private String buildBeanName(String interfaceName, boolean isSwitchOn) {
            return interfaceName + "Impl" + (isSwitchOn ? "V2" : "V1");
        }

        @Override
        public Object invoke(MethodInvocation invocation) throws Throwable {
            Method method = invocation.getMethod();
            String switchName = this.classSwitch;
            if (method.isAnnotationPresent(RoutingSwitch.class)) {
                switchName = method.getAnnotation(RoutingSwitch.class).value();
            }
            if (StringUtils.isBlank(switchName)) {
                throw new IllegalStateException("RoutingSwitch's value is blank, method:" + method.getName());
            }
            return invocation.getMethod().invoke(getTargetBean(switchName), invocation.getArguments());
        }

        public Object getTargetBean(String switchName) {
            boolean switchOn;
            if (RoutingVersion.A.equals(switchName)) {
                switchOn = false;
            } else if (RoutingVersion.B.equals(switchName)) {
                switchOn = true;
            } else {
                switchOn = FunctionSwitch.isSwitchOpened(switchName);
            }
            return switchOn ? beanOfSwitchOn : beanOfSwitchOff;
        }
    }
}
````

我简要解释一下思路：

+ 首先自定义了两个注解：RoutingInjected、RoutingSwitch，前者的作用类似于我们常用的Autowired，声明了该注解的属性将会被注入一个路由代理类实例；后者的作用则是一个配置开关，声明了控制路由的开关属性
+ 在RoutingBeanPostProcessor类中，我们在postProcessAfterInitialization方法中通过检查bean中是否存在声明了RoutingInjected注解的属性，如果发现存在该注解则给该属性注入一个动态代理类实例
+ RoutingBeanProxyFactory类功能就是生成一个代理类实例，代理类的逻辑也比较简单。版本路由支持到方法级别，即优先检查方法是否存在路由配置RoutingSwitch，方法不存在配置时才默认使用类路由配置

好了，BeanPostProcessor的介绍就到这里了。不知道看过后大家有没有得到一些启发呢？欢迎大家留言交流反馈。今天就到这里，我们下期再见吧 哈哈哈
