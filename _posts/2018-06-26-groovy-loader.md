---
layout: post
title: "基于spring的groovy动态加载"
description: "基于spring的groovy动态加载"
categories: [groovy]
tags: [groovy]
redirect_from:
  - /2018/06/26/
---
# 1 groovy loader背景
由于groovy动态语言的特性，使用方式与java一致，同时又特别适合与Spring的动态语言支持一起使用，所以基于java的groovy脚本的动态加载的使用场景还是比较多的

# 2 简介
动态加载指定目录下的groovy脚本，并将其注册为groovy bean，放置于ApplicationContext容器中，并使用命名空间进行分类区分(一个namespace对应于一个ApplicationContext)。同时能够动态感知到groovy脚本的新增、修改以及删除事件，并自动重新加载。

# 3 原理
- 使用spring配置文件来管理注册groovy bean：每一个spring配置文件作为一个ApplicationContext，管理一个namespace下的groovy bean
- spring配置文件使用标签```<lang:groovy>```，通过指定script-source来加载指定路径下的groovy脚本，通过refresh-check-delay属性来定时动态加载每个groovy bean
- 通过扫描监听指定路径下spring配置文件的变更，来接受groovy脚本的新增、删除事件

## 3.1 监听器listener
由于我们需要动态获取groovy脚本的变更，包含更新、新增、删除。接收到groovy脚本变更后需要触发用户自定义事件，所以我们先提供一个待用户实现的监听器接口，用于接受groovy变更事件，并执行相应代码

- GroovyRefreshedEvent

```
public class GroovyRefreshedEvent {

    private ApplicationContext ancestorContext;

    private Map<String, FileSystemXmlApplicationContext> namespacedContextMap;

    public GroovyRefreshedEvent(ApplicationContext ancestorContext, Map<String, FileSystemXmlApplicationContext> contextMap) {
        this.ancestorContext = ancestorContext;
        this.namespacedContextMap = contextMap;
    }
    ...
}
```

上述提到过，指定目录下的groovy脚本会被注册为groovy bean，并使用Applocation容器进行管理，同时还引入了namespace的概念，每个namespace分别对应一个Application容器，允许namespace不同但是类名相关的groovy bean存在。于是监听器的Event对象被设计成包含一个父容器以及namespace为key，对应Applocation容易为value的Map。

- GroovyRefreshedListener
```
public interface GroovyRefreshedListener {

    void groovyRefreshed(GroovyRefreshedEvent event);
}
```

这个需要用户来实现

## 3.2 触发器trigger
由于需要动态去发现groovy脚本的变更，需要通过定时轮训使用触发器去判断groovy脚本是否完成了变更。上述提到过，框架通过```<lang:Groovy>```来定时加载groovy脚本，通过扫描监听指定路径下spring配置文件的变更。通过这样的方式来接受groovy脚本的新增、删除事件。同样触发器也允许用户自定义，框架提供了一个默认的触发器

- GroovyRefreshedListener
```
public interface GroovyRefreshTrigger {

    boolean isTriggered(Map<String, Long> resourcesLastModifiedMap, String groovyResourcesDir);
}
```

- ResourceModifiedTrigger
```
public class ResourceModifiedTrigger implements GroovyRefreshTrigger {
    public boolean isTriggered(Map<String, Long> resourcesLastModifiedMap, String groovyResourcesDir) {
        if (StringUtils.isBlank(groovyResourcesDir)) {
            groovyResourcesDir = this.getClass().getClassLoader().getResource("").getPath() + "/spring/groovy";
        }
        File groovyFileDir = new File(groovyResourcesDir);
        List<File> groovyFileList = NamespacedGroovyLoader.getResourceListFromDir(groovyFileDir);
        for (File file : groovyFileList) {
            //新增
            if (!resourcesLastModifiedMap.containsKey(file.getName())) {
                return true;
            } else {
                //修改
                if (resourcesLastModifiedMap.get(file.getName()) != file.lastModified()) {
                    return true;
                }
            }
        }

        //删除
        if (resourcesLastModifiedMap.size() != groovyFileList.size()) {
            return true;
        }
        return false;
    }
}
```
默认的触发器就是通过spring配置文件上一次修改的时间，去判断文件是否被修改

## 3.3 groovy loader
初始化时会将指定目录下的groovy脚本动态加载成bean，并根据namespace注册到不同的Application容器中。注：以spring配置文件的名称作为namespace
```
private void initLoadResources() {
    if (MapUtils.isNotEmpty(this.namespacedContext)) {
        toDestoryContext = new ArrayList<FileSystemXmlApplicationContext>(this.namespacedContext.values());
    }
    this.namespacedContext = new HashMap<String, FileSystemXmlApplicationContext>();
    this.resourcesLastModifiedMap = new ConcurrentHashMap<String, Long>();
    //定位资源文件路径
    if (StringUtils.isBlank(groovyResourcesDir)) {
        groovyResourcesDir = this.getClass().getClassLoader().getResource("").getPath() + "/spring/groovy";
    }
    File groovyFileDir = new File(groovyResourcesDir);
    List<File> groovyFileList = getResourceListFromDir(groovyFileDir);//获取指定目录下所有groovy脚本文件
    for (File file : groovyFileList) {
        FileSystemXmlApplicationContext context = new FileSystemXmlApplicationContext(new String[] {file.toURI().toString()}, true, parentContext);
        this.namespacedContext.put(file.getName().replace("xml", ""), context);
        this.resourcesLastModifiedMap.put(file.getName(), file.lastModified());
    }

    //触发监听器事件
    listener.groovyRefreshed(new GroovyRefreshedEvent(parentContext, this.namespacedContext));

    if (CollectionUtils.isNotEmpty(toDestoryContext)) {
        for (FileSystemXmlApplicationContext fileSystemXmlApplicationContext : toDestoryContext) {
            fileSystemXmlApplicationContext.close();
        }
    }
}
```

## 3.4 spring配置文件
```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:lang="http://www.springframework.org/schema/lang" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
    http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd
    http://www.springframework.org/schema/lang http://www.springframework.org/schema/lang/spring-lang.xsd ">

    <context:annotation-config />

    <lang:groovy id="test" refresh-check-delay="2000" proxy-target-class="true"
                 script-source="classpath:groovy/one/Test.groovy" />

</beans>
```

# 4 使用
```
<bean id="listener" class="org.cuner.groovy.loader.test.TestListener"/><!--需要实现org.cuner.groovy.loader.listener.GroovyRefreshedListener -->
<bean id="groovyLoader" class="org.cuner.groovy.loader.NamespacedGroovyLoader">
    <property name="groovyResourcesDir" value=""/><!--指定spring groovy配置文件目录，若不设置或者为空则默认为classpath下/spring/groovy目录-->
    <property name="listener" ref="listener"/>
    <property name="trigger" ref="trigger"/>
</bean>
```

# 5 源码&Demo
代码托管在Github上，并附有使用demo，欢迎下载运行: https://github.com/Cuner/groovy-loader

# 6 版本升级 groovy-loader-v2
v1版本存在的缺陷在于：由于使用了```<lang:groovy>```标签，spring定时去重加载groovy bean，即使这个bean没有被修改，由此会产生一些性能消耗问题。为了解决这点，v2实现了groovy脚本的加载以及groovy bean的注入。同时由定时扫描spring配置改为扫描目录下的所有groovy脚本来实现触发器。

- GroovyScriptsModifiedTrigger
```
public boolean isTriggered(Map<String, Long> lastScriptsModified, String baseDir) {
    if (StringUtils.isBlank(baseDir)) {
        baseDir = this.getClass().getClassLoader().getResource("").getPath() + "/groovy";
    }

    List<File> fileList = getFileList(new File(baseDir));
    for (File file : fileList) {
        if (lastScriptsModified.get(file.getPath()) == null) {
            return true;
        }
        if (file.lastModified() != lastScriptsModified.get(file.getPath())) {
            return true;
        }
    }

    if (fileList.size() != lastScriptsModified.size()) {
        return true;
    }

    return false;
}
```
具体实现基本一致，区别在于外部传参baseDir由spring配置文件路径改为groovy脚本根目录路径。接下来我们看看是如何实现groovy脚本的加载以及groovy bean的注入。

- NamespacedGroovyLoader

```
/**
 * 递归查找并加载namespace下的所有groovy文件
 * @param file
 * @param namespace
 */
private void scanGroovyFiles(File file, String namespace) throws Exception {
    if (!file.exists()) {
        return;
    }

    if (file.isFile() && file.getName().endsWith(".groovy")) {
        ApplicationContext context = getOrCreateContext(namespace);
        DefaultListableBeanFactory beanFactory = (DefaultListableBeanFactory) context.getAutowireCapableBeanFactory();

        String scriptLocation = file.toURI().toString();
        if (scriptNotExists(context, scriptLocation)) {
            throw new IllegalArgumentException("script not exists : " + scriptLocation);
        }
        scriptLastModifiedMap.put(file.getPath(), file.lastModified());

        String className = StringUtils.removeEnd(scriptLocation.substring(scriptLocation.indexOf(baseDir) + baseDir.length() + 1).replace("/", "."), ".groovy");
        // 只有GroovyBean声明的类才实例化
        DSLScriptSource scriptSource = new DSLScriptSource(new ResourceScriptSource(context.getResource(scriptLocation)), className);

        Class scriptClass = groovyClassLoader.parseClass(scriptSource.getScriptAsString(), scriptSource.suggestedClassName());

        // Tell Groovy we don't need any meta
        // information about these classes
        GroovySystem.getMetaClassRegistry().removeMetaClass(scriptClass);
        groovyClassLoader.clearCache();

        // Create script factory bean definition.
        GroovyScriptFactory groovyScriptFactory = new GroovyScriptFactory(scriptLocation);
        groovyScriptFactory.setBeanFactory(beanFactory);
        groovyScriptFactory.setBeanClassLoader(urlClassLoader);
        Object bean = groovyScriptFactory.getScriptedObject(scriptSource);
        if (bean == null) {
            //只有静态方法的groovy脚本(没有类声明)
            return;
        }

        // Tell Groovy we don't need any meta
        // information about these classes
        GroovySystem.getMetaClassRegistry().removeMetaClass(bean.getClass());
        groovyScriptFactory.getGroovyClassLoader().clearCache();

        String beanName = StringUtils.removeEnd(file.getName(), ".groovy").toLowerCase();
        if (beanFactory.containsBean(beanName)) {
            beanFactory.destroySingleton(beanName); //移除单例bean
            removeInjectCache(context, bean); //移除注入缓存 否则Caused by: java.lang.IllegalArgumentException: object is not an instance of declaring class
        }
        beanFactory.registerSingleton(beanName, bean); //注册单例bean
        beanFactory.autowireBean(bean); //自动注入

    } else if (file.isDirectory()) {
        File[] subFiles = file.listFiles();
        for (File subFile : subFiles) {
            scanGroovyFiles(subFile, namespace);
        }
    }
}
```

groovy文件夹下第一级文件夹名称拼接namespacePrefix作为namespace，每个namespace分配一个ApplicationContext容器，每个namespace对应的bean都由各自的beanfactory加载注入，避免同名类加载导致的错误。同时groovy脚本更新后产生新的groovy bean之后，还需要移除之前的groovy bean，值得额外注意的是需要移除注入缓存，否则会报错：Caused by: java.lang.IllegalArgumentException: object is not an instance of declaring class
注意：由于我们不需要groovy的metaclass信息，这里对metaClass进行清除来进行优化

最后，附上代码:https://github.com/Cuner/groovy-loader-v2