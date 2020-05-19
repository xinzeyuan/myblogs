# URL
  - https://blog.csdn.net/qq_33322074/article/details/80091452

本文是转载原文出处：https://blog.csdn.net/zh15732621679/article/details/79074380
原文作者：https://blog.csdn.net/zh15732621679

# 相关概念
    在使用spring的配置文件时,新添加一个配置文件就需要添加相应的约束,一直没有研究过为什么要有这些约束,这些约束是干什么的。spring在启动的时候需要验证xml文档,约束的作用就是来验证配置文件的xml文档语法的正确性。

   在项目中其中的一个spring配置文件约束:

````
[html] view plain copy
<?xml version=“1.0” encoding=“UTF-8”?>  
<beans xmlns:xsi=“http://www.w3.org/2001/XMLSchema-instance”  
       xmlns:context=“http://www.springframework.org/schema/context”  
       xmlns:aop=“http://www.springframework.org/schema/aop”  
       xmlns=“http://www.springframework.org/schema/beans”  
       xsi:schemaLocation=”http://www.springframework.org/schema/beans  
        http://www.springframework.org/schema/beans/spring-beans.xsd  
        http://www.springframework.org/schema/context  
        http://www.springframework.org/schema/context/spring-context.xsd  
        http://www.springframework.org/schema/aop  
        http://www.springframework.org/schema/aop/spring-aop.xsd”>  
</beans>  

````

   XML文档的schema约束定义了文档的结构,内容和语法,元素和属性等,schema包括内容如下:
   1.XML配置文件中所有的标签和属性都需要schema来定义

   2.所有的schema都需要一个id,在XML配置文件中我们称之为namespace,其值是一个URI,通常是这个XML的xsd文件的地址
   3.引入一个schema约束使用属性xmlns,属性值为对应schema文件的命名空间namespace

   4.若引入的schema非w3c组织定义的,必须指定schema文件的位置,schema文件的位置由schemaLocation指定。

   5.引入多个schema需要使用别名,xmlns:alias

配置文件解析:

   1.beans:整个配置文件的根节点,包含一个或多个bean

   2.:xmlns:context=”http://www.springframework.org/schema/context”基本的命名空间

  3.xsi:schemaLocation:将命名空间和模式位置关联,提供了一个xml namespace到对应的xsd文件的一个映射,所以在xsi:schemaLocation后面配置的字符串都是成对的。

   4.使用别名引入多个schema文件

# spring如何检查xml
    spring在启动时会需要加载xsd文件来验证xml文件,有时候断网了,很可能项目就启动不了了。为了防止这种情况发生,spring提供了一种机制,默认先从本地加载xsd文件。
    目前我们项目中用到的spring版本是5.0.2.RELEASE版本,打开spring-context:5.0.2.RELEASE.jar,打开其下的META-INF文件夹,这下边有两个重要的文件,spring.handlers和spring.schemas

##     spring.handlers

````

[html] view plain copy
http\://www.springframework.org/schema/context=org.springframework.context.config.ContextNamespaceHandler  
http\://www.springframework.org/schema/jee=org.springframework.ejb.config.JeeNamespaceHandler  
http\://www.springframework.org/schema/lang=org.springframework.scripting.config.LangNamespaceHandler  
http\://www.springframework.org/schema/task=org.springframework.scheduling.config.TaskNamespaceHandler  
http\://www.springframework.org/schema/cache=org.springframework.cache.config.CacheNamespaceHandler  
http\://www.springframework.org/schema/context=org.springframework.context.config.ContextNamespaceHandler
http\://www.springframework.org/schema/jee=org.springframework.ejb.config.JeeNamespaceHandler
http\://www.springframework.org/schema/lang=org.springframework.scripting.config.LangNamespaceHandler
http\://www.springframework.org/schema/task=org.springframework.scheduling.config.TaskNamespaceHandler
http\://www.springframework.org/schema/cache=org.springframework.cache.config.CacheNamespaceHandler
````

  拿第一行举个例子,当我们需要http\://www.springframework.org/schema/context的schema时会使用org.springframework.context.config.ContextNamespaceHandler解析

##   spring.schemas:做了一个版本和本地xsd的映射

```` 
 
[html] view plain copy
http\://www.springframework.org/schema/context/spring-context-2.5.xsd=org/springframework/context/config/spring-context.xsd  
http\://www.springframework.org/schema/context/spring-context-3.0.xsd=org/springframework/context/config/spring-context.xsd  
http\://www.springframework.org/schema/context/spring-context-3.1.xsd=org/springframework/context/config/spring-context.xsd  
http\://www.springframework.org/schema/context/spring-context-3.2.xsd=org/springframework/context/config/spring-context.xsd  
http\://www.springframework.org/schema/context/spring-context-4.0.xsd=org/springframework/context/config/spring-context.xsd  
http\://www.springframework.org/schema/context/spring-context-4.1.xsd=org/springframework/context/config/spring-context.xsd  
http\://www.springframework.org/schema/context/spring-context-4.2.xsd=org/springframework/context/config/spring-context.xsd  
http\://www.springframework.org/schema/context/spring-context-4.3.xsd=org/springframework/context/config/spring-context.xsd  
http\://www.springframework.org/schema/context/spring-context.xsd=org/springframework/context/config/spring-context.xsd  
http\://www.springframework.org/schema/jee/spring-jee-2.0.xsd=org/springframework/ejb/config/spring-jee.xsd  
http\://www.springframework.org/schema/jee/spring-jee-2.5.xsd=org/springframework/ejb/config/spring-jee.xsd  
http\://www.springframework.org/schema/jee/spring-jee-3.0.xsd=org/springframework/ejb/config/spring-jee.xsd  
http\://www.springframework.org/schema/jee/spring-jee-3.1.xsd=org/springframework/ejb/config/spring-jee.xsd  
http\://www.springframework.org/schema/jee/spring-jee-3.2.xsd=org/springframework/ejb/config/spring-jee.xsd  
http\://www.springframework.org/schema/jee/spring-jee-4.0.xsd=org/springframework/ejb/config/spring-jee.xsd  
http\://www.springframework.org/schema/jee/spring-jee-4.1.xsd=org/springframework/ejb/config/spring-jee.xsd  
http\://www.springframework.org/schema/jee/spring-jee-4.2.xsd=org/springframework/ejb/config/spring-jee.xsd  
http\://www.springframework.org/schema/jee/spring-jee-4.3.xsd=org/springframework/ejb/config/spring-jee.xsd  
http\://www.springframework.org/schema/jee/spring-jee.xsd=org/springframework/ejb/config/spring-jee.xsd  
http\://www.springframework.org/schema/lang/spring-lang-2.0.xsd=org/springframework/scripting/config/spring-lang.xsd  
http\://www.springframework.org/schema/lang/spring-lang-2.5.xsd=org/springframework/scripting/config/spring-lang.xsd  
http\://www.springframework.org/schema/lang/spring-lang-3.0.xsd=org/springframework/scripting/config/spring-lang.xsd  
http\://www.springframework.org/schema/lang/spring-lang-3.1.xsd=org/springframework/scripting/config/spring-lang.xsd  
http\://www.springframework.org/schema/lang/spring-lang-3.2.xsd=org/springframework/scripting/config/spring-lang.xsd  
http\://www.springframework.org/schema/lang/spring-lang-4.0.xsd=org/springframework/scripting/config/spring-lang.xsd  
http\://www.springframework.org/schema/lang/spring-lang-4.1.xsd=org/springframework/scripting/config/spring-lang.xsd  
http\://www.springframework.org/schema/lang/spring-lang-4.2.xsd=org/springframework/scripting/config/spring-lang.xsd  
http\://www.springframework.org/schema/lang/spring-lang-4.3.xsd=org/springframework/scripting/config/spring-lang.xsd  
http\://www.springframework.org/schema/lang/spring-lang.xsd=org/springframework/scripting/config/spring-lang.xsd  
http\://www.springframework.org/schema/task/spring-task-3.0.xsd=org/springframework/scheduling/config/spring-task.xsd  
http\://www.springframework.org/schema/task/spring-task-3.1.xsd=org/springframework/scheduling/config/spring-task.xsd  
http\://www.springframework.org/schema/task/spring-task-3.2.xsd=org/springframework/scheduling/config/spring-task.xsd  
http\://www.springframework.org/schema/task/spring-task-4.0.xsd=org/springframework/scheduling/config/spring-task.xsd  
http\://www.springframework.org/schema/task/spring-task-4.1.xsd=org/springframework/scheduling/config/spring-task.xsd  
http\://www.springframework.org/schema/task/spring-task-4.2.xsd=org/springframework/scheduling/config/spring-task.xsd  
http\://www.springframework.org/schema/task/spring-task-4.3.xsd=org/springframework/scheduling/config/spring-task.xsd  
http\://www.springframework.org/schema/task/spring-task.xsd=org/springframework/scheduling/config/spring-task.xsd  
http\://www.springframework.org/schema/cache/spring-cache-3.1.xsd=org/springframework/cache/config/spring-cache.xsd  
http\://www.springframework.org/schema/cache/spring-cache-3.2.xsd=org/springframework/cache/config/spring-cache.xsd  
http\://www.springframework.org/schema/cache/spring-cache-4.0.xsd=org/springframework/cache/config/spring-cache.xsd  
http\://www.springframework.org/schema/cache/spring-cache-4.1.xsd=org/springframework/cache/config/spring-cache.xsd  
http\://www.springframework.org/schema/cache/spring-cache-4.2.xsd=org/springframework/cache/config/spring-cache.xsd  
http\://www.springframework.org/schema/cache/spring-cache-4.3.xsd=org/springframework/cache/config/spring-cache.xsd  
http\://www.springframework.org/schema/cache/spring-cache.xsd=org/springframework/cache/config/spring-cache.xsd  
http\://www.springframework.org/schema/context/spring-context-2.5.xsd=org/springframework/context/config/spring-context.xsd
http\://www.springframework.org/schema/context/spring-context-3.0.xsd=org/springframework/context/config/spring-context.xsd
http\://www.springframework.org/schema/context/spring-context-3.1.xsd=org/springframework/context/config/spring-context.xsd
http\://www.springframework.org/schema/context/spring-context-3.2.xsd=org/springframework/context/config/spring-context.xsd
http\://www.springframework.org/schema/context/spring-context-4.0.xsd=org/springframework/context/config/spring-context.xsd
http\://www.springframework.org/schema/context/spring-context-4.1.xsd=org/springframework/context/config/spring-context.xsd
http\://www.springframework.org/schema/context/spring-context-4.2.xsd=org/springframework/context/config/spring-context.xsd
http\://www.springframework.org/schema/context/spring-context-4.3.xsd=org/springframework/context/config/spring-context.xsd
http\://www.springframework.org/schema/context/spring-context.xsd=org/springframework/context/config/spring-context.xsd
http\://www.springframework.org/schema/jee/spring-jee-2.0.xsd=org/springframework/ejb/config/spring-jee.xsd
http\://www.springframework.org/schema/jee/spring-jee-2.5.xsd=org/springframework/ejb/config/spring-jee.xsd
http\://www.springframework.org/schema/jee/spring-jee-3.0.xsd=org/springframework/ejb/config/spring-jee.xsd
http\://www.springframework.org/schema/jee/spring-jee-3.1.xsd=org/springframework/ejb/config/spring-jee.xsd
http\://www.springframework.org/schema/jee/spring-jee-3.2.xsd=org/springframework/ejb/config/spring-jee.xsd
http\://www.springframework.org/schema/jee/spring-jee-4.0.xsd=org/springframework/ejb/config/spring-jee.xsd
http\://www.springframework.org/schema/jee/spring-jee-4.1.xsd=org/springframework/ejb/config/spring-jee.xsd
http\://www.springframework.org/schema/jee/spring-jee-4.2.xsd=org/springframework/ejb/config/spring-jee.xsd
http\://www.springframework.org/schema/jee/spring-jee-4.3.xsd=org/springframework/ejb/config/spring-jee.xsd
http\://www.springframework.org/schema/jee/spring-jee.xsd=org/springframework/ejb/config/spring-jee.xsd
http\://www.springframework.org/schema/lang/spring-lang-2.0.xsd=org/springframework/scripting/config/spring-lang.xsd
http\://www.springframework.org/schema/lang/spring-lang-2.5.xsd=org/springframework/scripting/config/spring-lang.xsd
http\://www.springframework.org/schema/lang/spring-lang-3.0.xsd=org/springframework/scripting/config/spring-lang.xsd
http\://www.springframework.org/schema/lang/spring-lang-3.1.xsd=org/springframework/scripting/config/spring-lang.xsd
http\://www.springframework.org/schema/lang/spring-lang-3.2.xsd=org/springframework/scripting/config/spring-lang.xsd
http\://www.springframework.org/schema/lang/spring-lang-4.0.xsd=org/springframework/scripting/config/spring-lang.xsd
http\://www.springframework.org/schema/lang/spring-lang-4.1.xsd=org/springframework/scripting/config/spring-lang.xsd
http\://www.springframework.org/schema/lang/spring-lang-4.2.xsd=org/springframework/scripting/config/spring-lang.xsd
http\://www.springframework.org/schema/lang/spring-lang-4.3.xsd=org/springframework/scripting/config/spring-lang.xsd
http\://www.springframework.org/schema/lang/spring-lang.xsd=org/springframework/scripting/config/spring-lang.xsd
http\://www.springframework.org/schema/task/spring-task-3.0.xsd=org/springframework/scheduling/config/spring-task.xsd
http\://www.springframework.org/schema/task/spring-task-3.1.xsd=org/springframework/scheduling/config/spring-task.xsd
http\://www.springframework.org/schema/task/spring-task-3.2.xsd=org/springframework/scheduling/config/spring-task.xsd
http\://www.springframework.org/schema/task/spring-task-4.0.xsd=org/springframework/scheduling/config/spring-task.xsd
http\://www.springframework.org/schema/task/spring-task-4.1.xsd=org/springframework/scheduling/config/spring-task.xsd
http\://www.springframework.org/schema/task/spring-task-4.2.xsd=org/springframework/scheduling/config/spring-task.xsd
http\://www.springframework.org/schema/task/spring-task-4.3.xsd=org/springframework/scheduling/config/spring-task.xsd
http\://www.springframework.org/schema/task/spring-task.xsd=org/springframework/scheduling/config/spring-task.xsd
http\://www.springframework.org/schema/cache/spring-cache-3.1.xsd=org/springframework/cache/config/spring-cache.xsd
http\://www.springframework.org/schema/cache/spring-cache-3.2.xsd=org/springframework/cache/config/spring-cache.xsd
http\://www.springframework.org/schema/cache/spring-cache-4.0.xsd=org/springframework/cache/config/spring-cache.xsd
http\://www.springframework.org/schema/cache/spring-cache-4.1.xsd=org/springframework/cache/config/spring-cache.xsd
http\://www.springframework.org/schema/cache/spring-cache-4.2.xsd=org/springframework/cache/config/spring-cache.xsd
http\://www.springframework.org/schema/cache/spring-cache-4.3.xsd=org/springframework/cache/config/spring-cache.xsd
http\://www.springframework.org/schema/cache/spring-cache.xsd=org/springframework/cache/config/spring-cache.xsd
````

    我们可以在指定的路径下找到本地的xsd,例如当我们需要http\://www.springframework.org/schema/context/spring-context-2.5.xsd时,我们可以在本地的org/springframework/context/config/目录下找到spring-context.xsd,spring是将xsd文件放到本地了,再在spring.schemas里做了一个映射,优先从本地加载xsd文件,所以当我们断网的时候一样可以启动成功,就是因为xsd加载的是本地的而不是走网络的。
    知道了为什么要写约束后,再用spring真是另一种感受啊,知其所以然,用的才顺心!
