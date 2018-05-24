---
layout: post
title: "SPI框架（一）"
description: "SPI框架"
categories: [SPI]
tags: [SPI]
redirect_from:
  - /2018/05/24/
---
SPI的全名为Service Provider Interface。根据Java的SPI规范，我们可以定义一个服务接口，具体的实现由对应的实现者去提供，即Service Provider（服务提供者）。然后在使用的时候只要根据SPI的规范去获取对应的服务提供者的服务实现即可。简单来说，SPI就是一种为接口寻找服务的机制，基于这种机制，本文提出一种基于动态代理的spi框架的实现方式。

## 1 基于统一的接口
该SPI框架的核心思想是通过JDK动态代理（代理对象必须为某个接口的实现），为接口寻找满足条件的服务实现。基于上述要求，需要提供一个统一的接口。
```
public interface SpiBase<T, R> {

    /**
     * 是否执行当前实现的条件
     * @param query
     * @return
     */
    boolean condition(T query);

    /**
     * 具体操作
     * @param query
     * @return
     */
    List<R> invoke(T query);

    /**
     * spi执行时候的配置
     * @param query
     * @return
     */
    SpiConfig config(T query);
}
```

作为框架，需要尽可能的符合各类场景
- 提供condition方法以及模板类T参数，让用户自定义符合接口实现的要求
- 提供统一的invoke方法来统一服务的某一类行为
- 提供spi执行的config，用来扩展寻求实现的策略

```
public class SpiConfig {

    /**
     * 是否互斥
     * false:满足执行条件的其他实现也一并调用
     * true:当前实现满足执行要求并执行成功后直接返回，忽略满足执行条件的其他实现
     */
    private boolean mutex;

    /**
     * 优先值
     * 该值越高，执行优先级越高
     */
    private int priority;

    /**
     * 名称 作为标识
     */
    private String name;
    
    ....
}
```

- 参数mutex，决定了某个接口，当有多个满足条件的服务实现时的执行策略，是只选择优先级最高的服务实现去执行，还是说按照优先级顺序多个服务实现顺序执行后将结果合并返回（这也是为什么invoke方法返回list的原因）。
- 参数priority，决定某个服务实现的执行优先级，优先级越高，越先执行

## 2 获取所有用户自定义的服务实现并实例化

2.1 提供注解@BizSpi
```
@Target({ElementType.TYPE})//目标作用于类
@Retention(RetentionPolicy.RUNTIME)// 注解在class字节码文件中存在，在运行时可以通过反射获取到
@Documented
@Service
public @interface BizSpi {
}
```

为了方便的找到用户提供的服务实现类，框架约束用户在其提供的服务实现类上加上注解@BizSpi，当然所有的服务实现类都需要实现统一的接口SpiBase，也可以实现某个继承SpiBase接口的自定义接口。

2.2 实例化服务实现并保存至内存
```
public class SpiManager implements ApplicationListener<ContextRefreshedEvent> {

    private static Map<Class, List<SpiBase>> spiProviderMap = new ConcurrentHashMap<Class, List<SpiBase>>();

    private static volatile boolean isLoaded = false;

    public void onApplicationEvent(ContextRefreshedEvent contextRefreshedEvent) {
        if (!isLoaded) {
            Map<String, Object> spiInstanceMap = contextRefreshedEvent.getApplicationContext().getBeansWithAnnotation(BizSpi.class);

            for (Object spiInstance : spiInstanceMap.values()) {
                Class spiInterface = spiInstance.getClass().getInterfaces()[0];
                if (spiProviderMap.containsKey(spiInterface) && spiProviderMap.get(spiInterface).size() > 0) {
                    spiProviderMap.get(spiInterface).add((SpiBase) spiInstance);
                } else {
                    List<SpiBase> spiProviderList = new ArrayList<SpiBase>();
                    spiProviderList.add((SpiBase) spiInstance);
                    spiProviderMap.put(spiInterface, spiProviderList);
                }
            }

            for (List<SpiBase> spiProviderList : spiProviderMap.values()) {
                Collections.sort(spiProviderList, new Comparator<SpiBase>() {
                    public int compare(SpiBase spiBase1, SpiBase spiBase2) {
                        return spiBase2.config(null).getPriority() - spiBase1.config(null).getPriority();
                    }
                });
            }
        }
        isLoaded = true;
    }

    public static List<SpiBase> getSpiProviderMap(Class spiInterface) {
        return spiProviderMap.get(spiInterface);
    }

}
```

这里通过约束的注解@BizSpi来拿到所有服务实现实例化后的对象，通过spiProviderMap来保存管理，其中spiProviderMap的key的集合是所有服务实现所实现的第一级接口（可以是SpiBase，也可以是继承自SpiBase的自定义接口），value则是key代表的接口所对应的所有服务实现实例化后的对象列表（可以看到列表通过SpiConfig中的priority值进行了重排序）。

## 3.动态代理
```
public class SpiProviderFactory<T> implements FactoryBean<T> {

    private Logger logger = LoggerFactory.getLogger(SpiProviderFactory.class);

    private Class<T> targetClass;

    public T getObject() throws Exception {
        InvocationHandler invocationHandler = new InvocationHandler() {
            public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                List<SpiBase> spiList = SpiManager.getSpiProviderMap(targetClass);

                // 支持组合,所以必须返回list; 每一个功能点实现也必须返回list
                List combinationResult = new ArrayList();
                for (SpiBase spi : spiList) {
                    if (SpiBase.class.isInstance(spi)) {
                        if (args != null && spi.condition(args[0])) {
                            SpiConfig spiConfig = spi.config(args[0]);
                            List result = (List) method.invoke(spi, args);
                            if (result != null) {
                                combinationResult.addAll(result);
                                if (spiConfig.isMutex()) {
                                    break;
                                }
                            }
                        }
                    }
                }
                return combinationResult;
            }
        };
        return (T) Proxy.newProxyInstance(Thread.currentThread().getContextClassLoader(),
                new Class[]{targetClass},
                invocationHandler);
    }
   ...
}
```

这里用到了工厂，这里需要传入参数targetClass，因为我们需要代理的接口可能是继承了SpiBase的自定义接口。通过服务实现类提供的condition方法可以从spiProviderMap中找到满足条件的服务，在通过SpiConfig的中参数决定服务执行的策略，最终将结果返回。

## 4.使用步骤
1.继承SpiBase得到自定义的SPI接口(可以添加自定义方法)
2.为SPI接口提供多个服务实现类(注意类需要加上注解@BizSpi)，condition方法决定了服务是否满足执行条件，config方法返回的SpiConfig对象决定了服务执行的策略
3.spring配置
```
    <!--扫描spi框架-->
    <context:component-scan base-package="org.cuner.spi.framework"/>

    <!--还需要扫描到你自己的服务实现类-->
    
    <bean id="demoSpi" class="org.cuner.spi.framework.core.SpiProviderFactory">
        <property name="targetClass" value="org.cuner.spi.framework.demo.api.demoSpi"/>
    </bean>
```
4.spring注入后使用
```
    @Resource(name = "demoSpi")
    private DemoSpi demoSpi;
```

5.demo
代码托管在Github上，并附有使用demo，欢迎下载运行: https://github.com/Cuner/spi-framework