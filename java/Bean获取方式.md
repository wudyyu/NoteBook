# Bean获取方式

## 1.ApplicationContext
> 最简单的一种方法就是通过直接注入的方式获取 ApplicationContext 对象，然后就可以通过 ApplicationContext 对象获取 Bean

```
@Component
public class InterfaceInject {
    @Autowired
    private ApplicationContext applicationContext;//注入

    public Object getBean(){
        return applicationContext.getBean("wolf1Bean");//获取bean
    }
}
```

## 2.ApplicationContextAware
> 通过实现 ApplicationContextAware 接口来获取 ApplicationContext 对象，从而获取 Bean。

> 需要注意的是，实现 ApplicationContextAware 接口的类也需要加上注解，以便交给 Spring 统一管理（这种方式也是项目中使用比较多的一种方式）

```
@Component
public class SpringContextUtil implements ApplicationContextAware {
    private static ApplicationContext applicationContext = null;

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
    }

    /**
     * 通过名称获取bean
     */
    public static <T>T getBeanByName(String beanName){
        return (T) applicationContext.getBean(beanName);
    }

    /**
     * 通过类型获取bean
     */
    public static <T>T getBeanByType(Class<T> clazz){
        return (T) applicationContext.getBean(clazz);
    }
}
```

> 封装之后，我们就可以直接调用对应的方法获取 Bean 了：
```
Wolf2Bean wolf2Bean = SpringContextUtil.getBeanByName("wolf2Bean");
Wolf3Bean wolf3Bean = SpringContextUtil.getBeanByType(Wolf3Bean.class);
```

## 3.ApplicationObjectSupport 和 WebApplicationObjectSupport 获取
> 这两个对象中，WebApplicationObjectSupport 继承了 ApplicationObjectSupport，所以并无实质的区别。

> 同样的，下面这个工具类也需要增加注解，以便交由 Spring 进行统一管理：
```
Component
public class SpringUtil extends /*WebApplicationObjectSupport*/ ApplicationObjectSupport {
    private static ApplicationContext applicationContext = null;

    public static <T>T getBean(String beanName){
        return (T) applicationContext.getBean(beanName);
    }

    @PostConstruct
    public void init(){
        applicationContext = super.getApplicationContext();
    }
}
```

> 有了工具类，在方法中就可以直接调用了：

```
@RestController
@RequestMapping("/hello")
@Qualifier
public class HelloController {
    @GetMapping("/bean3")
    public Object getBean3(){
        Wolf1Bean wolf1Bean = SpringUtil.getBean("wolf1Bean");
        return wolf1Bean.toString();
    }
}
```