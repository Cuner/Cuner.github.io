---
layout: post
title: "配置管理系统-client"
description: "配置管理系统-client"
categories: [Metabase]
tags: [Metabase]
redirect_from:
  - /2017/05/26/
---
# 1 什么是Metabase client

Metabase是一套内部的配置管理系统，能够对配置进行实时更新与获取。常用于使用一个配置，在不重启进程的情况下依然能够修改配置并让应用进程实时感知的场景。在配置变更时可以用listener来执行配置变更时需要执行的业务操作。      

Metabase client是Metabase的一个模块，主要负责配置分组下配置信息的拉取、异步线程定时check配置的变化(新增，删除，修改)，并触发对应的事件通知、更新配置本地文件(快照)等。  

# 2 Metabase的配置结构
Metabase使用配置组(configGroup)来区分一类配置信息（一般一个配置组对应一个应用的所有相关配置），而一个配置组里面以key-value的方式来存储各个配置，其中key作为配置组里面配置的唯一标识，value作为对应配置的值（以字符串的形式存储，不允许超过64kb）

# 3 Metabase client的启动

## 3.1 启动方式
Metabase client通过MetaClient实例的启动来完成任务，Metabase client内部每个configGroup下只有唯一的一个MetaClient实例，MetaClient提供一系列静态方法来获取MetaClient实例，通过传入的不同参数来选择不同的任务执行方式

```
public static MetaClient client(final String configGroup) {
    return innerClient(configGroup, true, false, null, null, true, ClientVersion.V1);
}

public static MetaClient client(final String configGroup, boolean autoStart) {
    return innerClient(configGroup, true, false, null, null, autoStart, ClientVersion.V1);
}

public static MetaClient client(final String configGroup, boolean triggerListenerWhenInit, boolean autoStart) {
    return innerClient(configGroup, true, triggerListenerWhenInit, null, null, autoStart, ClientVersion.V1);
}

public static MetaClient client(final String configGroup, List<SingleChangedListener> listeners,
                                GlobalChangedListener globalListener) {
    return innerClient(configGroup, true, false, listeners, globalListener, true, ClientVersion.V1);
}

public static MetaClient client(final String configGroup, boolean triggerListenerWhenInit,
                                List<SingleChangedListener> listeners, GlobalChangedListener globalListener) {
    return innerClient(configGroup, true, triggerListenerWhenInit, listeners, globalListener, true, ClientVersion.V1);
}

public static MetaClient client(final String configGroup, boolean triggerListenerInThreadPool,
                                boolean triggerListenerWhenInit, List<SingleChangedListener> listeners,
                                GlobalChangedListener globalListener, boolean autoStart) {
    return innerClient(configGroup, triggerListenerInThreadPool, triggerListenerWhenInit, listeners
            , globalListener, autoStart, ClientVersion.V1);
}
```

以下是参数说明
- configGroup:指定MetaClient对应的配置组，一个configGroup只对应唯一一个MetaClient实例。MetaClient初次创建后会以configGroup作为key存储到一个静态的ConcurrentMap中，当再次对同一个configGroup调用MetaClient.client()时，会直接从ConcurrentMap中取到之前创建的实例并返回，同时不会重复调用该实例的start方法避免重复启动
- triggerListenerInThreadPool:监听到配置的变更后是否在线程池中触发监听事件（后面详述）
- triggerListenerWhenInit:是够在初始化时触发监听事件（初始化获取配置信息）
- listeners:SingleChangedListener用于监听单个配置信息的变化，而传入的listeners包含多个SingleChangedListener（若想要监听某个配置信息的变化，需要实现接口SingleChangedListener的getMetaKey()方法指定监控配置的key以及configChanged()方法实现配置变化所触发的操作）
- golbalListener:用于监听全局配置信息的变化，需要实现接口GlobalChangedListener的configChanged()方法
- autoStart:是否自行启动，即是否自动调用MetaClient的start()方法
- version:默认ClientVersion.V1

## 3.2 实例化过程
接下来我们看看MetaClient的实例化做了哪些事情：（实例化之前先从静态ConcurrentMap对象中获取configGroup对应的MetaClient实例，如果获取到则直接返回，而如果获取不到则进行实例化操作，再将实例化的对象放置到ConcurrentMap中）
- 将配置参数赋值给MetaClient对象的成员变量:configGroup等
- 初始化监听器：接收传入的List<SingleChangedListener>单一监听器对象以及GlobalChangedListener全局监听器对象，并分别使用ConcurrentMap<String,CopyOnWriteArrayList<SingleChangedListener>>和CopyOnWriteArrayList<GlobalChangedListener>类型的成员变量存储
- 通过Executors.newScheduleThreadPool()得到调度线程池放置在成员变量scheduleExecutorService中
- 得到ClientMode实例并放置在变量clientMode
```
// local mode data file path
File localFile = new File(Client.META_DIR + Client.FILE_SEPARATOR + configGroup + ".json");
scheduledExecutorService = Executors.newScheduledThreadPool(1, new PollingThreadFactory(configGroup));
if (localFile.exists()) {
    checkInterval = 1000;
    clientMode = new LocalMode(this, localFile, executorService, triggerListenerInThreadPool,
            triggerListenerWhenInit, configGroup);
} else {
    checkInterval = 1;
    clientMode = new RemoteMode(this, configGroup, executorService, triggerListenerInThreadPool,
            triggerListenerWhenInit);
}
```

## 3.3 客户端启动
```
public void start() {
    checkOnlyStartedOnce();
    if (cv == ClientVersion.V1) {
        // 客户端实例化的时候做一次全量配置的fetch,
        // 当triggerListenerWhenInit＝true的时候，所有的配置均会触发一次EventType=INIT的事件
        configs = Collections.unmodifiableMap(this.clientMode.doFetch());
        scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
            public void run() {
                // do schedule
                try {
                    clientMode.doSchedule();
                } catch (Throwable e) {
                    logger.error(e.getMessage(), e);
                }
            }
        }, 0, checkInterval, TimeUnit.MILLISECONDS);
    } else {
        //处理轮询线程合并版本的启动,会检查启动状态,不会重复启动
        metaClientManager.startIfNot();
    }
}
```

- 检验是否只启动过一次:MetaClient对象有一个静态AtomicBoolean对象started，初始化为false，执行start方法前首先调用started.compareAndSet(false,true)方法来保证start()方法只能被执行一次，当设置autoStart为true后，手动调用start方法会报错，而设置autoStart为false的时候，也只能调用一次start方法。
- 调用clientMode的doFetch()方法获取配置组下的所有配置信息，放置在configs成员变量中
- 使用调度线程池scheduleExecutorService.scheduleAtFixedRate()，定时调用clientMode的doSchedule()方法实时拉取最新配置，检查配置信息的变更

## 3.4 配置信息

此外，MetaClient还提供getConfigurations()方法返回配置组下的所有配置信息configs

# 4 客户端模式（ClientMode）

## 4.1 ClientMode
MetaClient实际上是通过ClientMode来完成一系列配置操作，包括初始配置的拉取、配置信息的定时check以及触发listener等等，ClientMode分为LocalMode和RemoteMode，分别为本地模式与远程模式，主要实现以下接口:

### 4.1.1 Map<string,String> doFetch()
初始化时获取一份全量的配置
```
public Map<String, String> doFetch() {
    synchronized (this) {
        if (fetched) {
            return configs;
        }
        // begin
        doFinalFetch();
        // end
        fetched = true;
    }
    if (triggerListenerWhenInit) {
        fireInitEvents(dataMap);
    }
    return configs;
}
```
ClientMode的doFetch在抽象父类AbstractClientMode中实现，进行同步加锁主要为了避免多次重复拉取配置信息，当已经拉取过配置时直接返回配置信息，同时当配置初始处罚监听时，调用fireInitEvents方法。这里用到模板方法模式，控制了配置信息的拉取以及初始事件的触发，同时将不同模式下的配置拉取的实现推迟到子类LocalMode和RemoteMode.

### 4.1.2 void doSchedule()
【localMode】每1000ms定时检查本地配置信息的变更； 【remoteMode】每1ms定时轮训Metabase server上配置信息的变更

### 4.1.3 void fireInitEvents(Map<String,MetaData> configs)
针对第一次拉取的变量触发初始事件 当传入的triggerListenerWhenInit变量为true时调用

### 4.1.4 void triggerListener()
当检查到配置变化时，触发相应的listener

```
@Override
public void fireInitEvents(Map<String, MetaData> configs) {
    if (CollectionUtils.isEmpty(configs)) {
        return;
    }
    for (MetaData data : configs.values()) {
        triggerListener(new MetaEvent(data.getId(), data.getMetaKey(), 
                data.getMetaValue(), data.getVersion(), EventType.INIT));
    }
}

@Override
public void triggerListener(final MetaEvent event) {
    MetaEventPublisher.triggerListener(client, event, 
            orderedExecutorFactory, orderedExecutorForGlobalListener);
}
```
fireInitEvents()以及triggerListener()也是在抽象父类AbstractClientMode中实现。触发初始事件即是针对配置信息的每一个key-value创建一个初始事件，然后触发监听器；触发监听器则是传入了不同类型的事件MetaEvent，同时分别传入了两个执行器orderdExecutorFactory和orderExecutorForGlobalListener。【后续着重介绍】

## 4.2 LocalModel
本地模式是指从本地的配置文件里面获取配置信息以及信息的变更，所读取的文件是指定路径下以configGroup命名的json文件，在初始化ClientMode时，如果判断该文件存在，则启动本地模式。

### 4.2.1 doFinalFetch()
读取本地指定路径下的文件，获取配置信息，分别塞到变量ConcurrentMap<String, String> configs以及Map<String, MetaData> dataMap中并返回


### 4.2.3 void doSchedule()
通过比较上次check本地配置文件修改的时间与文件的最新修改时间(File.lastModefied)来判断是否需要对本地文件进行check。在对本地文件进行check的过程中，获取配置项的更新事件、新增事件以及删除事件，分别触发这些事件的监听器，同时更新配置信息configs
```
public void doSchedule() {
    if (!fetched) {
        return;
    }

    if (null == localFile || !localFile.exists()) {
        logger.error("local metabase data not exists!");
        return;
    }

    final long lastModified = localFile.lastModified();

    if (lastModified <= this.localFileLastModified) {// 本地文件没有发生变更
        return;
    }
    // 更新最后修改时间
    this.localFileLastModified = lastModified;
    // 处理本地模式下的配置变更
    dealLocalConfigChanged(fetchLocal());
}
```

## 4.3 RemoteMode
远程模式相对本地模式要复杂的多，远程模式使用HTTP协议与远程服务器进行通信。当启用远程模式时，会实例化一个ServerListServiceProxyForMultiEnv类型的服务器代理对象proxy，这个自定义的对象proxy首先通过http访问某个静态地址，从而得到一份IP地址列表，进而为远程模式按序分配IP地址。

### 4.3.1 doFinalFetch()
通过httpGet与指定服务器进行通信，访问由configGroup控制的路由，接受的返回值即是指定configGroup下的配置信息。获取配置信息使用了重试机制，用于处理访问异常的情况，重试时，使用proxy重新分配的IP地址再次进行http通信，重试的最大次数取决于代理对象proxy能够提供的服务器IP数目。当成功获取配置信息或者超过最大重试次数时，将配置信息初始化到内存，并以json快照文件的形式保存当前配置到本地【当前分组下配置总量比较大的情况下，每次把所有配置序列化成json，并覆盖掉硬盘上之前旧的snapshot，这里会成为性能瓶颈，所以在异步线程中进行】（当远程fetch失败，即重试次数超过最大次数时，会试图从本地快照读取配置信息）
```
protected void doFinalFetch() {
    int times = proxy.getServerIps().size();
    boolean fetchRemoteResult = false;
    List<MetaData> dataList = null;
    CloseableHttpResponse response = null;
    // 重试机制
    while (times-- > 0) {
        try {
            serverHostRecord = serverHost;
            response = MbHttpClientFactory.getClient().execute(createGetForFetch(null));
            int respHttpCode = response.getStatusLine().getStatusCode();
            if (200 == respHttpCode) {// success
                dataList = getDataList(response);
                fetchRemoteResult = true;
                break;
            } else if (204 == respHttpCode) {// 配置为空
                logger.warn("have no any config in metabase server!");
                fetchRemoteResult = true;
                break;
            } else if (400 == respHttpCode) {// 参数错误
                throw new RuntimeException("request paramters is not valid,please check!");
            } else if (503 == respHttpCode) {// 服务不可达,切换serverHost进行重试
                String tempHost = serverHost;
                serverHost = proxy.nextIp();
                logger.warn("current server host:" + tempHost + "unavailable, so change server host to " + serverHost);
            } else {
                serverHost = proxy.nextIp();
                logger.warn("fetch data fail, the response code is:" + respHttpCode + " and the detail msg is: "
                        + new String(getRespDataFromServer(response), Charset.forName("UTF-8")));
            }
        } catch (IOException e) {
            logAndReselectIp(e, logger);
        } catch (Throwable e) {
            throw new RuntimeException(e);
        } finally {
            NetUtils.reuseHttpClientConn(response, logger);
        }
    }
    // 初始化配置到内存
    initConfigs(fetchRemoteResult, dataList);
}
```

### 4.3.2 doSchedule()
通过httpPost与指定服务器进行通信，访问由configGroup控制的路由，(与doFinalFetch访问的路由不是同一个)，接受的返回值即是指定configGroup下的各个配置的变化，根据请求的返回值触发监听器、更新配置信息以及更新本地快照。同时对请求返回异常进行计数，当返回404或者异常次数超过20时，在一定时间内阻塞当前线程，而由于doSchedule本身就是被定时调用的，自身也就不需要重试机制。

## 4.4 triggerListener
之前有提到过，在实例MetaClient的时候，能指定触发监听器的方式【是否在线程池中触发，由变量triggerListenerInThreadPool控制】。触发器在接受到配置信息的变化事件后会拿到全局监听器以及修改配置信息对应的单一监听器，进而触发这些监听器。
- 不在线程池中触发：触发器将按照顺序逐一的触发监听器，listener之间相互阻塞
- 在线程池中触发：针对SingleListener保证多个key的并发执行，单个key的listener顺序执行，同时合并单个key的多个listener；针对GlobalListener将所有事件添加到同一个LinkedList中，有单个线程顺序执行，同时合并同一个key的多个事件
```
public static void triggerListener(final MetaClient client, final MetaEvent event, 
                            OrderedExecutorFactory orderedExecutorFactory, 
                            OrderedExecutorForGlobalListener orderedExecutorForGlobalListener) {
    final List<GlobalChangedListener> globalChangedListeners = client.getGlobalListeners();
    List<SingleChangedListener> listeners = client.getSingleListeners(event.getMetaKey());
    //v2版本triggerListenerInThreadPool属性无效,强制listener在thread pool里执行
    if(!client.isTriggerListenerInThreadPool() && ClientVersion.V2 != client.getVersion()){
        if (CollectionUtils.isNotEmpty(listeners)) {
            for (SingleChangedListener listener : listeners) {
                triggerSingle(listener, client.getConfigGroup(), event);
            }
        }
        if(CollectionUtils.isNotEmpty(globalChangedListeners)){
            for(GlobalChangedListener globalChangedListener : globalChangedListeners){
                triggerGlobal(globalChangedListener, client.getConfigGroup(), event);
            }
        }
        return;
    }
    
    if (CollectionUtils.isNotEmpty(listeners)) {
        for (final SingleChangedListener listener : listeners) {
            orderedExecutorFactory.getExecutor(String.valueOf(UniqueObjectId.getId(listener))).execute(new Runnable() {
                public void run() {
                    triggerSingle(listener, client.getConfigGroup(), event);
                }
            });
        }
    }
    if(CollectionUtils.isNotEmpty(globalChangedListeners)){
        orderedExecutorForGlobalListener.execute(new Task(event.getMetaKey(),new Runnable() {
            public void run() {
                for(GlobalChangedListener globalChangedListener : globalChangedListeners){
                    triggerGlobal(globalChangedListener, client.getConfigGroup(), event);
                }
            }
        }));
    }
}
```

- OrderedExecutorFactory:负责触发SingleListener，能够维持多个key之间的listener的并发执行，单个key的listener顺序执行，同时对同一个key的listener进行合并

```
public OrderedExecutor(Executor parent) {
    this.parent = parent;
    runner = new Runnable() {
        @Override
        public void run() {
            for (;;) {
                Runnable currentTask;
                synchronized (lock) {
                    currentTask = OrderedExecutor.this.task;
                    if (currentTask == null) {
                        running = false;
                        return;
                    }
                    OrderedExecutor.this.task = null;
                }
                try {
                    currentTask.run();
                } catch (Throwable t) {
                    log.error("Caught unexpected Throwable", t);
                }
            }
        }
    };
}

public void execute(Runnable command) {
    synchronized (lock) {
        this.task = command;
        if (!running) {
            running = true;
            parent.execute(runner);
        }
    }
}
```

OrderedExecutorFactory中维护了一个ConcurrentMap，使用configGroup下的所有配置key做为Map的键值，分别对应一个执行器OrderedExecutor，这样保证多个key之间的修改能够使用不同的执行器进行并发执行；同一个key的配置修改没有直接采用并发的方式(因为直接并发会影响到监听器执行的顺序)，而是在主线程中顺序完成任务的替换。    

每当接受配置信息的变化事件时，将该事件的监听器触发事件作为Runnable传入执行器execute方法。在执行器的execute方式中，由于使用了关键字synchronized使得同一个key下的listener触发严格按照顺序执行【顺序错乱会导致老的修改覆盖掉新的修改】。execute方法主要按顺序指定下个待执行的listener，保证后续的listener能够覆盖当前的listener，同时控制执行动作的启动。其中parent变量是一个缓存线程池执行器【Executors.newCachedThreadPool()】,在缓存池中指定监听器的触发。    

runner.run里的关键字synchronized是为保证task变量的读取以及赋值操作互相阻塞(能够避免task变量在读取为空之后的的赋值操作被遗漏)，因为不同的线程是存在不同的变量缓存的，如果两者不相互阻塞，某个线程的赋值操作后，随即另外一个线程的读操作可能读取不到，因为线程中的缓存变量可能未及时同步过去，这样会导致处理遗漏的问题。 两者相互阻塞后能够保证给task变量的赋值的线程完成赋值操作后，释放掉锁同时同步task的变量值到其他线程，此时另外的线程就能够读取到最新的task了。    


- OrderedExecutorForGlobalListener:某个负责触发GlobalListener，将所有的修改事件添加到同一个LinkedList中，有单个线程顺序执行，同一个key的多个事件会被合并

```
public OrderedExecutorForGlobalListener(Executor parent) {
    this.parent = parent;
    runner = new Runnable() {
        @Override
        public void run() {
            for (;;) {
                final Task task;
                synchronized (tasks) {
                    task = tasks.poll();
                    if (task == null) {
                        running = false;
                        return;
                    }
                }
                try {
                    task.getCommand().run();
                } catch (Throwable t) {
                    logger.error("Caught unexpected Throwable", t);
                }
            }
        }
    };
}

public void execute(Task task) {
    synchronized (tasks) {
        //检查相同key的task是否已经在list中,不存在则添加，存在则更新
        int index = tasks.indexOf(task);
        if(-1 == index){//add
            tasks.add(task);
        }else{//update
            tasks.get(index).setCommand(task.getCommand());
        }
        if (!running) {
            running = true;
            parent.execute(runner);
        }
    }
}
```

OrderedExecutorForGlobalListener的execute中的关键字synchronized同样也是为了保证任务的顺序执行，使用LinkedList存储task，使用对应配置信息的key传入tast.key，listener触发作为Runnable传入tast.command，用来保证同个key的多个事件被合并

# 5 总结

- 远程模式中拉取配置信息以及定时check配置变更的时候使用HTTP长连接，好处在于使用方便，编码简单，能够通过状态码获取访问服务器的状态
- 在远程重试获取配置信息并失败时，采用读取本地json文件的容灾方案
- 本次client学习只是学习的client的整体流程以及一些并发的思想，其中包括远程模式的网络通信（控制漂移）以及序列化还不能完全掌握，此外MetaServer之间基于vertx的http通信也是下一步学习的重点。
