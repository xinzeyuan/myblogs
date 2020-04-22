# URL
  - https://www.jianshu.com/p/be6c818fe6ff


动态管理配置文件扩展接口EnvironmentPostProcessor


# EnvironmentPostProcessor功能说明
EnvironmentPostProcessor从名字上看，叫做"环境后置处理器"，它是一个接口，它可以再spring上下文启动的时候，去初始化一些基本配置信息，将某些变量信息，加载到spring容器上下文中，更加通俗的理解就是它可以用来解析加载我们自定义额外properties

SpringBoot支持动态的读取文件，留下的扩展接口org.springframework.boot.env.EnvironmentPostProcessor。这个接口是spring包下的，使用这个进行配置文件的集中管理，而不需要每个项目都去配置配置文件。这种方法也是springboot框架留下的一个扩展（可以自己去扩展）

# demo
在/Users/naeshihiroshi/study/studySummarize/SpringBoot/（自己测试也可以随机在一个目录下建立一文件），目录下建立一个名为springboot.properties文件，

springboot.properties中定义一些配置，配置如下：

```
jdbc.root.user=zhihao.miao
jdbc.root.password=242312321
```

定义MyEnvironmentPostProcessor实现EnvironmentPostProcessor接口

```
@Component
public class MyEnvironmentPostProcessor implements EnvironmentPostProcessor {

    @Override
    public void postProcessEnvironment(ConfigurableEnvironment environment, SpringApplication application) {
        try{
            InputStream inputStream = new FileInputStream("/Users/naeshihiroshi/study/studySummarize/SpringBoot/springboot.properties");
            Properties properties = new Properties();
            properties.load(inputStream);
            PropertiesPropertySource propertiesPropertySource = new PropertiesPropertySource("my",properties);
            environment.getPropertySources().addLast(propertiesPropertySource);
        }catch (IOException e){
            e.printStackTrace();
        }
    }
}
```

在classpath定义一个META-INF文件夹然后在其下面先建spring.factories文件，在其中指定：

![](5225109-bfc0276363c4a59b.webp)

```
org.springframework.boot.env.EnvironmentPostProcessor=com.zhihao.miao.processor.MyEnvironmentPostProcessor‘
```

启动类测试：


```
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        ConfigurableApplicationContext context = SpringApplication.run(Application.class,args);
        String username = context.getEnvironment().getProperty("jdbc.root.user");
        String password = context.getEnvironment().getProperty("jdbc.root.password");
        System.out.println("username==="+username);
        System.out.println("password==="+password);
        context.close();
    }
}
```

打印结果：

```
username===zhihao.miao
password===242312321
```

# EnvironmentPostProcessor接口官网说明
官方文档如下

```
Allows for customization of the application's {@link Environment} prior to the application context being refreshed.
允许定制应用的上下文的应用环境优于应用的上下文之前被刷新。（意思就是在spring上下文构建之前可以设置一些系统配置）

EnvironmentPostProcessor implementations have to be registered in
META-INF/spring.factories, using the fully qualified name of this class as the key.
EnvironmentPostProcessor的实现类必须要在META-INF/spring.factories文件中去注册，并且注册的是全类名。

EnvironmentPostProcessor processors are encouraged to detect whether Spring's
org.springframework.core.Ordered Ordered interface has been implemented or if the @org.springframework.core.annotation.Order Order annotation is present and to sort instances accordingly if so prior to invocation.
```

鼓励EnvironmentPostProcessor处理器检测Org.springframework.core.Ordered注解，这样相应的实例也会按照@Order注解的顺序去被调用。
