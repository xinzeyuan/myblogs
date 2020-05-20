# URL
  - https://www.jianshu.com/p/aa28bc3936cb

InstantiationAwareBeanPostProcessor源码解析

# 简介
* 继承BeanPostProcessor接口，在此基础上又定义了三个方法，分别在Bean实例化前后【不是初始化】执行。
* 从上面的介绍可以看到，这个接口相对于BeanPostProcessor功能更加强大，一个接口承担了Bean的实例化前后、初始化前后责任。

# Bean加载顺序
* ioc容器创建加载Bean的执行顺序如下：
+ InstantiationAwareBeanPostProcessor接口中的postProcessBeforeInstantiation，在实例化之前调用
+ Bean的实例化，调用构造方法
+ InstantiationAwareBeanPostProcessor接口中的postProcessAfterInstantiation，在实例化之后调用
+ InstantiationAwareBeanPostProcessor接口中的postProcessPropertyValues【当postProcessAfterInstantiation返回true才执行】
+ BeanPostProcessor接口中的postProcessBeforeInitialization，在初始化之前调用
+ InitializingBean中的afterProperties方法，执行初始化
+ BeanPostProcessor接口中的postProcessAfterInitialization，在实例化之后调用

# InstantiationAwareBeanPostProcessor接口方法的执行顺序
* 正常的执行顺序如下：
+ postProcessBeforeInstantiation
+ postProcessAfterInstantiation
+ postProcessProperties
+ postProcessBeforeInitialization
+ postProcessAfterInitialization

# 方法解析
* Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName)：在实例化之前执行【构造方法之前执行】
+ 返回值：如果返回的不为null，那么后续的Bean的创建流程【实例化、初始化afterProperties】都不会执行，而是直接使用返回的快捷Bean，此时的正常执行顺序如下：
InstantiationAwareBeanPostProcessor接口中的postProcessBeforeInstantiation，在实例化之前调用
BeanPostProcessor接口中的postProcessAfterInitialization，在实例化之后调用

````
/**
* org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.resolveBeforeInstantiation 
* 作用：在实例化之前解析是否有快捷创建的Bean，既是通过postProcessBeforeInstantiation返回的Bean
* 内部调用两个重要的方法：
*   1、applyBeanPostProcessorsBeforeInstantiation：内部遍历调用postProcessBeforeInstantiation方法【在实例化之前调用】
*   2、applyBeanPostProcessorsAfterInitialization：如果postProcessBeforeInstantiation方法返回了快捷的Bean，内部遍历调用postProcessBeforeInstantiation方法【在初始化之后调用】
*/
protected Object resolveBeforeInstantiation(String beanName, RootBeanDefinition mbd) {
        Object bean = null;
        if (!Boolean.FALSE.equals(mbd.beforeInstantiationResolved)) {
            // Make sure bean class is actually resolved at this point.
            if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
                Class<?> targetType = determineTargetType(beanName, mbd);
                if (targetType != null) {
                    //调用方法，内部遍历调用postProcessBeforeInstantiation方法【在实例化之前调用】
                    bean = applyBeanPostProcessorsBeforeInstantiation(targetType, beanName);
                    //如果返回了快捷的Bean
                    if (bean != null) {
                        //如果postProcessBeforeInstantiation方法返回了快捷的Bean，内部遍历调用postProcessBeforeInstantiation方法【在初始化之后调用】
                        bean = applyBeanPostProcessorsAfterInitialization(bean, beanName);
                    }
                }
            }
            mbd.beforeInstantiationResolved = (bean != null);
        }
        return bean;
    }
    
    /**
    *   org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.applyBeanPostProcessorsBeforeInstantiation
    *   作用：调用postProcessBeforeInstantiation方法
    */
    protected Object applyBeanPostProcessorsBeforeInstantiation(Class<?> beanClass, String beanName)
                throws BeansException {
            //遍历所有的后置处理器
            for (BeanPostProcessor bp : getBeanPostProcessors()) {
                //判断是否是InstantiationAwareBeanPostProcessor类型的，如果是的，调用postProcessBeforeInstantiation方法获取快捷Bean
                if (bp instanceof InstantiationAwareBeanPostProcessor) {
                    InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
                    Object result = ibp.postProcessBeforeInstantiation(beanClass, beanName);
                    if (result != null) {
                        return result;
                    }
                }
            }
            return null;
        }
    
    /**
    *   org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.applyBeanPostProcessorsAfterInitialization
    *   作用：遍历调用postProcessAfterInitialization
    */
    public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
                throws BeansException {
            Object result = existingBean;
            for (BeanPostProcessor beanProcessor : getBeanPostProcessors()) {
                result = beanProcessor.postProcessAfterInitialization(result, beanName);
                if (result == null) {
                    return result;
                }
            }
            return result;
        }
````
		
boolean postProcessAfterInstantiation(Object bean, String beanName) throws BeansException：正常情况下在实例化之后在执行populateBean之前调用
返回值：如果有指定的bean的时候返回false，那么后续的属性填充和属性依赖注入【populateBean】将不会执行，同时后续的postProcessPropertyValues将不会执行,但是初始化和BeanPostProcessor的仍然会执行。

````
/**
* org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.populateBean
* 填充指定Bean的属性
* 在该方法内部遍历所有的BeanPostPorcessor，调用postProcessAfterInstantiation方法
*/
protected void populateBean(String beanName, RootBeanDefinition mbd, BeanWrapper bw) {
        //获取属性
        PropertyValues pvs = mbd.getPropertyValues();
        if (bw == null) {
            if (!pvs.isEmpty()) {
                throw new BeanCreationException(
                        mbd.getResourceDescription(), beanName, "Cannot apply property values to null instance");
            }
            else {
                // Skip property population phase for null instance.
                return;
            }
        }

        //**********************逻辑开始执行********************
        //标志，判断是否继续执行属性填充，默认为false
        boolean continueWithPropertyPopulation = true;
        //判断ioc容器中是否存在InstantiationAwareBeanPostProcessors(
        if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
            //遍历所有的BeanPostProcessor
            for (BeanPostProcessor bp : getBeanPostProcessors()) {
                //判断类型是InstantiationAwareBeanPostProcessor
                if (bp instanceof InstantiationAwareBeanPostProcessor) {
                    InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
                    //执行postProcessAfterInstantiation方法
                    if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
                        //返回结果为false，那么赋值continueWithPropertyPopulation=false，表示不继续执行属性填充
                        continueWithPropertyPopulation = false;
                        break;
                    }
                }
            }
        }
        //如果continueWithPropertyPopulation为false，直接返回，不执行下面的步骤
        if (!continueWithPropertyPopulation) {
            return;
        }
        //
        if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME ||
                mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
            MutablePropertyValues newPvs = new MutablePropertyValues(pvs);

            // Add property values based on autowire by name if applicable.
            if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME) {
                autowireByName(beanName, mbd, bw, newPvs);
            }

            // Add property values based on autowire by type if applicable.
            if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
                autowireByType(beanName, mbd, bw, newPvs);
            }

            pvs = newPvs;
        }

        boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
        boolean needsDepCheck = (mbd.getDependencyCheck() != RootBeanDefinition.DEPENDENCY_CHECK_NONE);

        if (hasInstAwareBpps || needsDepCheck) {
            PropertyDescriptor[] filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
            if (hasInstAwareBpps) {
                //同样是遍历BeanPostProcessor
                for (BeanPostProcessor bp : getBeanPostProcessors()) {
                    if (bp instanceof InstantiationAwareBeanPostProcessor) {
                        InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
                        //执行postProcessPropertyValues方法
                        pvs = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
                        if (pvs == null) {
                            return;
                        }
                    }
                }
            }
            if (needsDepCheck) {
                checkDependencies(beanName, mbd, filteredPds, pvs);
            }
        }
        //重要的一步，设置属性
        applyPropertyValues(beanName, mbd, bw, pvs);
    }
````	
	
public PropertyValues postProcessPropertyValues(PropertyValues pvs, PropertyDescriptor[] pds, Object bean, String beanName)：实例化之后调用，在方法applyPropertyValues【属性填充】之前
返回值：如果返回null，那么将不会进行后续的属性填充，比如依赖注入等，如果返回的pvs额外的添加了属性，那么后续会填充到该类对应的属性中。
pvs：PropertyValues对象，用于封装指定类的对象，简单来说就是PropertyValue的集合，里面相当于以key-value形式存放类的属性和值
pds：PropertyDescriptor对象数组，PropertyDescriptor相当于存储类的属性，不过可以调用set，get方法设置和获取对应属性的值

````
/**
* org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.populateBean的代码片段
*/
if (hasInstAwareBpps || needsDepCheck) {
            PropertyDescriptor[] filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
            if (hasInstAwareBpps) {
                //遍历调用postProcessPropertyValues方法
                for (BeanPostProcessor bp : getBeanPostProcessors()) {
                    if (bp instanceof InstantiationAwareBeanPostProcessor) {
                        InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
                        pvs = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
                        //如果返回的pvs是null，直接返回
                        if (pvs == null) {
                            return;
                        }
                    }
                }
            }
            if (needsDepCheck) {
                checkDependencies(beanName, mbd, filteredPds, pvs);
            }
        }   
        //执行真正的属性填充
        applyPropertyValues(beanName, mbd, bw, pvs);
````		
实例
只是写了InstantiationAwareBeanPostProcessor定义的方法，另外的BeanPostProcessor的方法，请看上一篇文章

````
@Component
public class MyInstantiationAwareBeanPostProcessor implements InstantiationAwareBeanPostProcessor {

    /**
     * 在实例化之前调用，如果返回null，一切按照正常顺序执行，如果返回的是一个实例的对象，那么这个将会跳过实例化、初始化的过程
     * @param beanClass
     * @param beanName
     * @return
     */
    @Override
    public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {
        if (beanClass == User.class) {
            System.out.println("postProcessBeforeInstantiation执行");
            return null;
        }

        return null;
    }

    /**
     * 在实例化之后，postProcessBeforeInitialization之前执行
     * @param bean
     * @param beanName
     * @return
     * @throws BeansException
     */
    @Override
    public boolean postProcessAfterInstantiation(Object bean, String beanName) throws BeansException {
        if (bean instanceof User) {
            System.out.println("postProcessAfterInstantiation执行");
            return true;
        }

        return true;
    }

    /**
     * 实例化之后调用，属性填充之前
     * @param pvs PropertyValues对象，用于封装指定类的对象，简单来说就是PropertyValue的集合，里面相当于以key-value形式存放类的属性和值
     * @param pds PropertyDescriptor对象数组，PropertyDescriptor相当于存储类的属性，不过可以调用set，get方法设置和获取对应属性的值
     * @param bean 当前的bean
     * @param beanName beanName
     * @return 如果返回null，那么将不会进行后续的属性填充，比如依赖注入等，如果返回的pvs额外的添加了属性，那么后续会填充到该类对应的属性中。
     */
    @Override
    public PropertyValues postProcessPropertyValues(PropertyValues pvs, PropertyDescriptor[] pds, Object bean, String beanName) throws BeansException {
        if (pvs instanceof MutablePropertyValues&&bean instanceof User){
           MutablePropertyValues mutablePropertyValues= (MutablePropertyValues) pvs;
           HashMap<Object, Object> map = new HashMap<>();
           map.put("name","陈加兵");
           map.put("age",44);
           mutablePropertyValues.addPropertyValues(map);
           return mutablePropertyValues;
        }

        /**使用pds设置值
        if (bean instanceof User) {
            for (PropertyDescriptor descriptor:pds) {
                try {
                    if ("name".equals(descriptor.getName())) {
                        descriptor.getWriteMethod().invoke(bean, "陈加兵");
                    }else if("age".equals(descriptor.getName())){
                        descriptor.getWriteMethod().invoke(bean,40);
                    }
                }catch (Exception e){
                    e.printStackTrace();
                }
            }
            return null;
        }**/
        return pvs;
    }

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        if (bean instanceof User) {
            System.out.println("postProcessBeforeInitialization执行");
        }
        return bean;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        if (bean instanceof User) {
            System.out.println("postProcessAfterInitialization执行");
        }
        return bean;
    }
}
````

源码梳理
无论是BeanPostProcessor还是InstantiationAwareBeanPostProcessor都是在对象实例化和初始化前后执行的逻辑，因此我们主要的代码都在getBean，doGetBean，cerateBean方法中
BeanPostProcessor的两个方法的执行源码请看上一篇的文章
步骤如下：
[图片上传失败...(image-4658b9-1561472145733)]
[图片上传失败...(image-824031-1561472145733)]
[图片上传失败...(image-b7d4b0-1561472145733)]
[图片上传失败...(image-d4fb2f-1561472145733)]
[图片上传失败...(image-1ad844-1561472145733)]
[图片上传失败...(image-40efd3-1561472145733)]
[图片上传失败...(image-cd78c-1561472145733)]
[图片上传失败...(image-2b40a-1561472145733)]
[图片上传失败...(image-acc5c5-1561472145733)]
Autowired源码解析
从源码可以看出，Autowired的功能实现最重要的一个接口就是AutowiredAnnotationBeanPostProcessor，继承关系如下：
[图片上传失败...(image-518665-1561472145733)]
从继承关系图可以看出，实际上关键的实现了InstantiationAwareBeanPostProcessor这个接口。
源码实现如下图：
[图片上传失败...(image-f8c9e8-1561472145733)]
总结
源码：
ioc容器创建Bean的方法是从createBean方法进入的，真正执行创建的Bean的是doCreateBean方法，我们从createBean开始往下走
调用resolveBeforeInstantiation方法【在doCreatBean之前执行，即是实例化之前】，在内部遍历BeanPostProcessor调用postProcessBeforeInstantiation方法
如果postProcessBeforeInstantiation方法返回null，那么需要执行实例化的过程，调用doCreatBean实例化Bean。
doCreateBean内部分为两步：①调用createBeanInstance实例化Bean；②调用populateBean设置Bean的属性
在populateBean内部分为如下的步骤：
调用postProcessAfterInstantiation【实例化之后调用】，分为两种情况：①返回false，后续的postProcessPropertyValues将不再执行，属性也不在进行设置；②返回true，程序照常进行，调用postProcessPropertyValues，属性设置的过程正常进行
执行完populateBean之后将会调用initializeBean【初始化Bean，调用afterPropertiesSet方法】，在内部就涉及到BeanPostProcessor定义的接口了，步骤如下：
执行applyBeanPostProcessorsBeforeInitialization方法调用postProcessBeforeInitialization【在初始化之前调用】方法
执行invokeInitMethods方法，内部其实是调用afterPropeertiesSet方法，进行初始化
执行applyBeanPostProcessorsAfterInitialization，内部调用postProcessAfterInitialization【在实例化之后调用】方法
