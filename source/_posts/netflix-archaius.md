---
title: 微服务动态配置组件netflix archaius
date: 2018-05-14 12:03:13
tags: [netflix,archaius,config]
categories: spring
---

# archaius介绍

archaius是Netflix公司开源项目之一，基于java的配置管理类库，主要用于多配置存储的动态获取。主要功能是对apache common configuration类库的扩展。在云平台开发中可以将其用作分布式配置管理依赖构件。同时，它有如下一些特性：

- 动态获取属性
- 高效和线程安全的配置操作
- 配置改变时提供回调机制
- 可以通过jmx操作配置
- 复合配置

<!--more-->

官网给出的结构图：
![](/images/archaius/archaius.png)

# 适用场景

对于传统的单体应用，properties等配置文件可以解决配置问题，同时也可以通过maven profile配置来区别各个环境，但在一个几百上千节点的的微服务生态中，如何把每个微服务的配置文件都进行更新，并且很多时候还需要重启服务，是一件无法忍受的事情。
对于微服务架构而言，一个通用的配置中心是必不可少的。zookeeper、consul、etcd都可以作为集中的配置中心，但是他们的客户端api不是为配置而编写的，使用到配置场景时，会显得非常的臃肿，Archaius可以与这些配置中心结合，它提供的动态配置api，使用起来非常简单方便。
spring cloud的spring-cloud-netflix-core组件中集成了archaius，hystrix使用archaius动态修改各commandKey对应的参数（并发数和超时时间），用简单优雅的代码实现不重启生效配置项。

# 快速入门

在maven工程中加入依赖的配置：
```
<dependency>
  <groupId>com.netflix.archaius</groupId>
  <artifactId>archaius-core</artifactId>
  <version>0.7.4</version>
 </dependency>
```
archaius已经发展到2.x版本，但是spring cloud集成的还是0.7.X  

编写示例代码：
```
public class Main {
    //获取一个Long型的动态配置项，默认值是1000。
    private static DynamicLongProperty timeToWait =
            DynamicPropertyFactory.getInstance().getLongProperty("lock.waitTime", 1000);

    public static void main(String[] args) throws Exception {
        //设置回调
        timeToWait.addCallback(() -> {
            System.out.println("timeToWait callback, new value: " + timeToWait.get());
        });
        //每秒将timeToWait动态配置值打印到控制台，timeToWait.get()会动态的获取最新的配置
        for (int i = 0; i < 100000; i++) {
            Thread.sleep(1000);
            System.out.println("timeToWait: " + timeToWait.get());
        }
    }
}

```
archaius默认读取classpath下的config.properties，所以在resources目录下增加config.properties文件，文件中增加配置：

```
lock.waitTime=4
```

然后运行Main，会在控制台打印：

```
timeToWait: 4
timeToWait: 4
timeToWait: 4
。。。
```
让main继续运行，修改config.properties配置文件内容：
```
lock.waitTime=5
```
重新编译工程，等待1分钟左右，控制台会打印：

```
timeToWait: 4
timeToWait callback, new value: 5
timeToWait: 5
timeToWait: 5
。。。
```
一分钟等得有点长，在启动main时加入VM参数

```
-Darchaius.fixedDelayPollingScheduler.delayMills=2000 -Darchaius.fixedDelayPollingScheduler.initialDelayMills=2000
```
只要修改config.properties配置文件，两秒后就能生效。

# api说明

## 基本类型动态配置

- DynamicFloatProperty
- DynamicDoubleProperty
- DynamicBooleanProperty
- DynamicStringProperty
- DynamicIntProperty
- DynamicLongProperty

类图如下：
![](/images/archaius/property.png)

基本类型的动态配置继承`PropertyWrapper`类，实现接口`Property`，方法说明：

```
public interface Property<T> {
    //获取动态配置值
    T getValue();
    //获取默认值
    T getDefaultValue();
    //获取配置名
    String getName();
    //获取修改时间戳
    long getChangedTimestamp();
    //增加修改回调
    void addCallback(Runnable callback);
    //删除所有回调
    void removeAllCallbacks();
}
```
通过`DynamicPropertyFactory`的单例获取配置项实例


## 扩展类型动态配置

- DynamicStringListProperty
可以动态配置String list
```
//String list
DynamicStringListProperty prop = new DynamicStringListProperty("test2", "0|1", "\\|");
List<String> list = prop.get();//获取包含"0","1"的字符串列表
```
- DynamicStringMapProperty
可以动态配置String Map
```
//String map
DynamicStringMapProperty prop = new DynamicStringMapProperty("test3", "key1=1,key2=2,key3=3");
Map<String, String> map = prop.get();
```

- DynamicStringSetProperty
可以动态配置String set
```
//String set
DynamicStringSetProperty prop = new DynamicStringSetProperty("test4", "a,b,c");
```
这几个扩展类型内部有个`private DynamicStringProperty delegate;`属性，通过解析配置字符串创建需要的类型。

## 变更回调

动态配置项通过`addCallback`方法增加回调函数：
```
//设置回调
timeToWait.addCallback(() -> {
    System.out.println("timeToWait callback, new value: " + timeToWait.get());
});
```

# 指定配置源

## 指定配置文件名称
archauis默认配置源是classpath下的config.properties，可以增加VM启动参数   
`-Darchaius.configurationSource.defaultFileName=test.properties`来修改文件名称

## 指定配置url
通过启动参数：  
`-Darchaius.configurationSource.additionalUrls=file:///c:/config.properties,https://raw.githubusercontent.com/along101/spring-boot-test/master/actuator-test/src/main/resources/application.properties`指定配置源的url，以逗号分隔。
配置多个url相同配置项，后面的配置会覆盖前面的，classpath下配置文件优先级最低。



# spring boot集成
spring bean初始化完成，属性值从配置文件注入之后，如果修改配置文件，属性值是不会修改的。如果我们想在运行过程中动态的获取配置项，我们可以将spring的Environment注入到bean中，在代码逻辑中从Environment中获取配置：
```
@Autowired
private Environment env;
public void someMethod(){
  ...
  String config = environment.getProperty("someConfig.name");
  ...
}
```
这样的代码显得有些臃肿，并且性能也不高，因为environment是由多个配置源组合起来的。我们使用archaius动态配置就简单很多：
```
private DynamicStringProperty someConfig=DynamicPropertyFactory.getInstance().getStringProperty("someConfig.name", "default");
public void someMethod(){
  ...
  String config = someConfig.get();
  ...
}
```
如果想参数变更后做些业务操作，archaius非常简单：
```
//某个bean的内部
private DynamicStringProperty someConfig=DynamicPropertyFactory.getInstance().getStringProperty("someConfig.name", "default");

@PostConstruct
public void someMethod(){
  someConfig.addCallback(() -> {
            System.out.println("new value: " + someConfig.get());
            //业务代码
        });
}
```
spring boot中如何集成archaius呢？ spring-cloud-netflix-core自动配置了archaius，集成spring boot，只要加入如下依赖：
```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-netflix-core</artifactId>
    <version>1.2.7.RELEASE</version>
</dependency>
<dependency>
    <groupId>com.netflix.archaius</groupId>
    <artifactId>archaius-core</artifactId>
    <version>0.7.4</version>
</dependency>
```

编写测试类TestApplication验证:
```
@SpringBootApplication
public class TestApplication {

    @Autowired
    Environment env;

    @Autowired
    ArchaiusAutoConfiguration archaiusAutoConfiguration;

    private static final DynamicStringProperty archaiusTest =
            DynamicPropertyFactory.getInstance().getStringProperty("archaius.test", "test1");

    public static void main(String[] args) {
        SpringApplication.run(TestApplication.class, args);
    }

    @PostConstruct
    public void init() {
        System.out.println("env config: " + env.getProperty("archaius.test"));
        System.out.println("archaius config: " + archaiusTest.get());
    }
}
```
在spring boot配置文件中加入配置：
```
archaius.test=archaius-test
```
运行测试类TestApplication，会在控制台打印出
```
env config: archaius-test
archaius config: archaius-test
```
说明archaius从spring 的env中拿到配置了。  

但是这里有个问题，spring boot本身使用了外部配置，比如集成了apollo，**修改配置，archaius配置项是不生效的**。分析spring-cloud-netflix-core的源代码，里面有个ConfigurableEnvironmentConfiguration继承了apache的AbstractConfiguration，将spring Environment进行了封装，实例化后作为配置源加到archaius的组合配置中，并没有像DynamicURLConfiguration那样进行schedule。   

我们前面对archaius的源代码进行了详细的分析，可以对此进行改进，让spring更新env时，archaius配置项能感知到。首先增加一个配置源类，实现PolledConfigurationSource接口：
```
@Slf4j
public class SpringEnvConfigurationSource implements PolledConfigurationSource {
    //spring的Environment
    private ConfigurableEnvironment environment;
    private PropertySourcesPropertyResolver resolver;

    SpringEnvConfigurationSource(ConfigurableEnvironment environment) {
        this.environment = environment;
        this.resolver = new PropertySourcesPropertyResolver(this.environment.getPropertySources());
        this.resolver.setIgnoreUnresolvableNestedPlaceholders(true);

    }

    @Override
    public PollResult poll(boolean initial, Object checkPoint) throws Exception {
        log.debug("archaius config refresh.");
        //poll是从spring env中拉取配置
        return PollResult.createFull(getProperties());
    }

    public Map<String, Object> getProperties() {
        Map<String, Object> properties = new LinkedHashMap<>();
        //spring env 里面也是多个source组合的
        for (Map.Entry<String, PropertySource<?>> entry : getPropertySources().entrySet()) {
            PropertySource<?> source = entry.getValue();
            if (source instanceof EnumerablePropertySource) {
                EnumerablePropertySource<?> enumerable = (EnumerablePropertySource<?>) source;
                for (String name : enumerable.getPropertyNames()) {
                    if (!properties.containsKey(name)) {
                        properties.put(name, resolver.getProperty(name));
                    }
                }
            }
        }
        return properties;
    }

    //PropertySource也可能是组合的，通过递归获取
    private Map<String, PropertySource<?>> getPropertySources() {
        Map<String, PropertySource<?>> map = new LinkedHashMap<String, PropertySource<?>>();
        MutablePropertySources sources = null;
        if (environment != null) {
            sources = environment.getPropertySources();
        } else {
            sources = new StandardEnvironment().getPropertySources();
        }
        for (PropertySource<?> source : sources) {
            extract("", map, source);
        }
        return map;
    }

    private void extract(String root, Map<String, PropertySource<?>> map,
                         PropertySource<?> source) {
        if (source instanceof CompositePropertySource) {
            for (PropertySource<?> nest : ((CompositePropertySource) source)
                    .getPropertySources()) {
                extract(source.getName() + ":", map, nest);
            }
        } else {
            map.put(root + source.getName(), source);
        }
    }
}

```
这段代码是参考spring-cloud-netflix-core中的ConfigurableEnvironmentConfiguration实现的。  
然后编写自动配置类代码：
```
@Configuration
@ConfigurationProperties("archaius")
public class ArchaiusConfig implements EnvironmentAware, InitializingBean {
    private static final String CONFIG_NAME = "springEnv";
    @Setter
    private int pollDelayMillis = 5000;
    @Setter
    private int poolInitialDelayMillis = 5000;
    @Setter
    private boolean pollIgnoreDeletesFromSource = false;

    private Environment environment;

    @Override
    public void setEnvironment(Environment environment) {
        this.environment = environment;
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        SpringEnvConfigurationSource springEnvConfigurationSource = new SpringEnvConfigurationSource((ConfigurableEnvironment) this.environment);
        //新建一个PollingScheduler
        FixedDelayPollingScheduler scheduler = new FixedDelayPollingScheduler(poolInitialDelayMillis, pollDelayMillis, pollIgnoreDeletesFromSource);
        ConcurrentMapConfiguration configuration = new ConcurrentCompositeConfiguration();
        //开始轮询，从SpringEnvConfigurationSource中拉取，archaius内部会比较配置的变化
        scheduler.startPolling(springEnvConfigurationSource, configuration);

        //初始化archaius，这段代码也是参考spring-cloud-netflix-core的
        if (ConfigurationManager.isConfigurationInstalled()) {
            AbstractConfiguration installedConfiguration = ConfigurationManager.getConfigInstance();
            if (installedConfiguration instanceof ConcurrentCompositeConfiguration) {
                ConcurrentCompositeConfiguration configInstance = (ConcurrentCompositeConfiguration) installedConfiguration;
                if (configInstance.getConfiguration(CONFIG_NAME) == null)
                    configInstance.addConfigurationAtFront(configuration, CONFIG_NAME);
            }
        } else {
            ConcurrentCompositeConfiguration concurrentCompositeConfiguration = new ConcurrentCompositeConfiguration();
            concurrentCompositeConfiguration.addConfiguration(configuration, CONFIG_NAME);
            ConfigurationManager.install(concurrentCompositeConfiguration);
        }
    }
}
```
修改启动类进行测试：
```
@SpringBootApplication
public class TestApplication {

    @Autowired
    ConfigurableEnvironment env;

    @Autowired
    ArchaiusAutoConfiguration archaiusAutoConfiguration;

    private static final DynamicStringProperty archaiusTest =
            DynamicPropertyFactory.getInstance().getStringProperty("archaius.test", "test1");

    public static void main(String[] args) throws InterruptedException {
        SpringApplication.run(TestApplication.class, args);
        Thread.sleep(50000 * 1000);
    }

    @PostConstruct
    public void init() {
        System.out.println("env config: " + env.getProperty("archaius.test"));
        System.out.println("archaius config: " + archaiusTest.get());
        //给env中增加一个配置源
        MutablePropertySources sources = env.getPropertySources();
        Map<String, Object> config = new HashMap<>();
        config.put("archaius.test", "map change");
        //自定义的配置源加到最前面
        sources.addFirst(new MapPropertySource("myMap", config));
        new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    int i = 0;
                    while (true) {
                        Thread.sleep(2 * 1000);
                        //打印archaius配置项
                        System.out.println("archaius config: " + archaiusTest.get());
                        i++;
                        //修改配置源
                        config.put("archaius.test", "map change " + i);
                    }
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }).start();
    }
}
```
运行结果：
```
env config: archaius-test
archaius config: archaius-test
...
archaius config: map change
archaius config: map change
archaius config: map change 2
archaius config: map change 2
archaius config: map change 2
archaius config: map change 5
archaius config: map change 5
archaius config: map change 7
archaius config: map change 7

```
可以看到不是每次修改都会生效，轮询时间在ArchaiusConfig中进行的配置。代码地址：https://github.com/along101/spring-boot-test/tree/5ad5158732ea363aa4fb5b9b965c08977699a9c6/archaius-test


就这么几十行代码就可以将archaius与spring boot结合起来了。


# 源码分析
archaius源码不多，重点在配置的初始化和动态变更这块。

## 初始化配置
初始化入口是`DynamicPropertyFactory.getInstance()`，来看看该方法源码
```
    public static DynamicPropertyFactory getInstance() {
        if (config == null) {
            synchronized (ConfigurationManager.class) {
                if (config == null) {
                    //先初始化AbstractConfiguration
                    AbstractConfiguration configFromManager = ConfigurationManager.getConfigInstance();
                    if (configFromManager != null) {
                        //初始化自己
                        initWithConfigurationSource(configFromManager);
                        initializedWithDefaultConfig = !ConfigurationManager.isConfigurationInstalled();
                        logger.info("DynamicPropertyFactory is initialized with configuration sources: " + configFromManager);
                    }
                }
            }
        }
        return instance;
    }
```

`ConfigurationManager`的静态代码块中执行
```
static{
    ...省略部分代码

    //设置部署上下文，会初始化environment、datacenter、applicationId、serverId、region等等，不配置的话，就为空
    setDeploymentContext(new ConfigurationBasedDeploymentContext());
}
```

经过一层层调用，调用到`createDefaultConfigInstance`来创建`AbstractConfiguration`：
```
    private static AbstractConfiguration createDefaultConfigInstance() {
        //新建一个组合配置，配置列表中自带一个ConcurrentMapConfiguration，这个配置是containerConfiguration，容器自己的配置
        ConcurrentCompositeConfiguration config = new ConcurrentCompositeConfiguration();  
        try {
            //增加一个DynamicURLConfiguration，获取默认配置文件url和配置系统属性archaius.configurationSource.additionalUrls得到的url
            //构造函数最后，开启线程定时从url里面更新配置
            DynamicURLConfiguration defaultURLConfig = new DynamicURLConfiguration();
            config.addConfiguration(defaultURLConfig, URL_CONFIG_NAME);
        } catch (Throwable e) {
            logger.warn("Failed to create default dynamic configuration", e);
        }
        if (!Boolean.getBoolean(DISABLE_DEFAULT_SYS_CONFIG)) {
            //增加SystemConfiguration
            SystemConfiguration sysConfig = new SystemConfiguration();
            config.addConfiguration(sysConfig, SYS_CONFIG_NAME);
        }
        if (!Boolean.getBoolean(DISABLE_DEFAULT_ENV_CONFIG)) {
            //增加环境配置
            EnvironmentConfiguration envConfig = new EnvironmentConfiguration();
            config.addConfiguration(envConfig, ENV_CONFIG_NAME);
        }
        //增加app的组合配置
        ConcurrentCompositeConfiguration appOverrideConfig = new ConcurrentCompositeConfiguration();
        config.addConfiguration(appOverrideConfig, APPLICATION_PROPERTIES);
        config.setContainerConfigurationIndex(config.getIndexOfConfiguration(appOverrideConfig));
        return config;
    }
```
初始化完后，得到一个组合配置，包含5个配置，顺序为：
- URL_CONFIG_NAME 通过url拉取的配置
- SYS_CONFIG_NAME 系统属性配置,通过`System.getProperties()`获取的配置
- ENV_CONFIG_NAME 环境配置,通过`System.getenv()`获取的配置
- containerConfiguration 容器配置，`ConfigurationManager.getConfigInstance().setProperty(key,value)`设置的配置
- APPLICATION_PROPERTIES 调用`configurationManager.loadAppOverrideProperties`设置的配置

![](/images/archaius/config.png)

这样就生成了`AbstractConfiguration`，保存在`ConfigurationManager.instance`静态变量上。  

回到`DynamicPropertyFactory`代码里面，生成了`AbstractConfiguration configFromManager`之后，调用方法`initWithConfigurationSource`初始化：

```
    public static DynamicPropertyFactory initWithConfigurationSource(AbstractConfiguration config) {

            。。。
            //包装config为DynamicPropertySupport
            return initWithConfigurationSource(new ConfigurationBackedDynamicPropertySupportImpl(config));
        }
    }
```
后面会设置`setDirect`，将`DynamicPropertySupport`注册到`DynamicProperty`中
```
    static void setDirect(DynamicPropertySupport support) {
        synchronized (ConfigurationManager.class) {
            config = support;
            DynamicProperty.registerWithDynamicPropertySupport(support);
            initializedWithDefaultConfig = false;
        }
    }
```
`DynamicProperty`注册`DynamicPropertySupport`的过程是增加一个`DynamicPropertyListener`，更新所有的属性，也就是更新他的静态变量`ALL_PROPS`里面的配置项
```
    static synchronized void initialize(DynamicPropertySupport config) {
        dynamicPropertySupportImpl = config;
        config.addConfigurationListener(new DynamicPropertyListener());
        updateAllProperties();
    }
```
到这里就初始化完了。根据以上源代码的分析，我们画出类图如下：

![](/images/archaius/init.png)

这里最重要的两个类是
- `ConfigurationManager` 静态变量`AbstractConfiguration instance为`所有配置项信息
- `DynamicProperty` 静态变量`DynamicPropertySupport dynamicPropertySupportImpl`用于动态更新，静态变量`ConcurrentHashMap<String, DynamicProperty> ALL_PROPS`为被使用的动态属性


## 获取配置

通过以下代码获取动态配置：
```
DynamicLongProperty timeToWait =
            DynamicPropertyFactory.getInstance().getLongProperty("lock.waitTime", 1000);
```
进入`getLongProperty`方法：

```
    public DynamicLongProperty getLongProperty(String propName, long defaultValue, final Runnable propertyChangeCallback) {
        //检查初始化
        checkAndWarn(propName);
        //新建一个动态配置
        DynamicLongProperty property = new DynamicLongProperty(propName, defaultValue);
        //增加回调
        addCallback(propertyChangeCallback, property);
        return property;
    }
```

`DynamicLongProperty`继承了`PropertyWrapper`，含有属性`DynamicProperty prop`，先初始化该属性：
```
this.prop = DynamicProperty.getInstance(propName);
```
再往下看：
```
    public static DynamicProperty getInstance(String propName) {
        //dynamicPropertySupportImpl为空，先初始化DynamicPropertyFactory
        if (dynamicPropertySupportImpl == null) {
            DynamicPropertyFactory.getInstance();
        }
        //从ALL_PROPS中获取prop，没有的话，新建，并放到ALL_PROPS中
        //这里没有加同步设置，但是做了个小动作，没有并发问题
        DynamicProperty prop = ALL_PROPS.get(propName);
        if (prop == null) {
            prop = new DynamicProperty(propName);
            //ALL_PROPS是ConcurrentHashMap，putIfAbsent是线程安全的，存在的话返回老的值
            DynamicProperty oldProp = ALL_PROPS.putIfAbsent(propName, prop);
            //oldProp不为null，返回oldProp，解决了并发的问题，副作用是多建了DynamicProperty对象
            if (oldProp != null) {
                prop = oldProp;
            }
        }
        return prop;
    }
```
再看`new DynamicProperty(propName)`

```
    private DynamicProperty(String propName) {
        //设置属性名称
        this.propName = propName;
        updateValue();
    }

//更新属性值
private boolean updateValue() {
        String newValue;
        try {
            if (dynamicPropertySupportImpl != null) {
                //从dynamicPropertySupportImpl中获取配置项，这里是包装AbstractConfiguration的，也就是在ConfigurationManager中的instance
                newValue = dynamicPropertySupportImpl.getString(propName);
            } else {
                return false;
            }
        } catch (Exception e) {
            e.printStackTrace();
            logger.error("Unable to update property: " + propName, e);
            return false;
        }
        return updateValue(newValue);
    }

boolean updateValue(Object newValue) {
        String nv = (newValue == null) ? null : newValue.toString();
        synchronized (lock) {
            if ((nv == null && stringValue == null)
               || (nv != null && nv.equals(stringValue))) {
                return false;
            }
            //更新stringValue
            stringValue = nv;
            //刷新缓存
            cachedStringValue.flush();
            booleanValue.flush();
            integerValue.flush();
            floatValue.flush();
            classValue.flush();
            doubleValue.flush();
            longValue.flush();
            changedTime = System.currentTimeMillis();
            return true;
        }
    }
```

到这里`DynamicLongProperty`对象就新建了，里面包含一个`DynamicProperty`，配置值的更新、回调都是由这个`DynamicProperty`完成的，他的属性有：

```
    //配置名称
    private String propName;
    //配置的值，原始string
    private String stringValue = null;
    //更新时间
    private long changedTime;
    //回到函数表
    private CopyOnWriteArraySet<Runnable> callbacks = new CopyOnWriteArraySet<Runnable>();

    //缓存的StringValue
    private CachedValue<String> cachedStringValue = new CachedValue<String>() {
        protected String parse(String rep) {
            return rep;
        }
    };
    //缓存的integerValue
    private CachedValue<Integer> integerValue = new CachedValue<Integer>() {
        protected Integer parse(String rep) throws NumberFormatException {
            return Integer.valueOf(rep);
        }
    };

    private CachedValue<Long> longValue = new CachedValue<Long>() {
        protected Long parse(String rep) throws NumberFormatException {
            return Long.valueOf(rep);
        }
    };

    private CachedValue<Float> floatValue = new CachedValue<Float>() {
        protected Float parse(String rep) throws NumberFormatException {
            return Float.valueOf(rep);
        }
    };

    private CachedValue<Double> doubleValue = new CachedValue<Double>() {
        protected Double parse(String rep) throws NumberFormatException {
            return Double.valueOf(rep);
        }
    };

    private CachedValue<Class> classValue = new CachedValue<Class>() {
        protected Class parse(String rep) throws ClassNotFoundException {
            return Class.forName(rep);
        }
    };

```
这个`CachedValue`是`DynamicLongProperty`的内部类，缓存了配置的实际值，通过解析`DynamicLongProperty`的stringValue值获取不同类型的配置值。下面我们来分析这一过程。`DynamicLongProperty.get()`获取动态配置项的值代码如下：
```
    public long get() {
        return prop.getLong(defaultValue).longValue();
    }
```
通过内置的prop属性即`DynamicProperty`获取的配置项，prop中通过属性longValue获取配置项
```
    public Long getLong(Long defaultValue) {
        return longValue.getValue(defaultValue);
    }
```
前面初始化prop时，会初始化属性`CachedValue<Long> longValue`：

```
    private CachedValue<Long> longValue = new CachedValue<Long>() {
        protected Long parse(String rep) throws NumberFormatException {
            return Long.valueOf(rep);
        }
    };
```
longValue是一个`CachedValue`类型，getValue方法如下：

```
        public T getValue() throws IllegalArgumentException {
            // Not quite double-check locking -- since isCached is marked as volatile
            if (!isCached) {
                synchronized (lock) {
                    try {
                        //如果配置值DynamicProperty.stringValue为null，则没有该配置项，返回null，否则解析stringValue返回
                        //这里的parse方法自然是调用longValue的parse方法返回Long值
                        value = (stringValue == null) ? null : parse(stringValue);
                        exception = null;
                    } catch (Exception e) {
                        value = null;
                        exception = new IllegalArgumentException(e);
                    } finally {
                        isCached = true;
                    }
                }
            }
            if (exception != null) {
                throw exception;
            } else {
                return value;
            }
        }
```
到这里，通过`DynamicLongProperty.getLongProperty`获取的配置值就能拿到了。其他的`DynamicIntProperty`、`DynamicStringProperty`等配置项的获取与此相同。



## 动态更新
archaius最有价值的特点是能动态更新配置项，这里动态更新的配置项是指前面我们分析初始化是的`DynamicURLConfiguration`，我们先看看`DynamicURLConfiguration`的是如何动态更新的，构造函数：

```
    public DynamicURLConfiguration() {
        URLConfigurationSource source = new URLConfigurationSource();
        if (source.getConfigUrls() != null && source.getConfigUrls().size() > 0) {
            startPolling(source, new FixedDelayPollingScheduler());
        }
    }
```
这里的startPolling开始轮询url资源内容，然后更新：

```
    public synchronized void startPolling(PolledConfigurationSource source, AbstractPollingScheduler scheduler) {
        this.scheduler = scheduler;
        this.source = source;
        init(source, scheduler);
        //然后schedule调度轮询
        scheduler.startPolling(source, this);        
    }

    public void startPolling(final PolledConfigurationSource source, final Configuration config) {
        //先初始化
        initialLoad(source, config);
        Runnable r = getPollingRunnable(source, config);
        schedule(r);
    }

    @Override
    protected synchronized void schedule(Runnable runnable) {
    //线程池调度
        executor = Executors.newScheduledThreadPool(1, new ThreadFactory() {
            @Override
            public Thread newThread(Runnable r) {
                Thread t = new Thread(r, "pollingConfigurationSource");
                t.setDaemon(true);
                return t;
            }
        });
        executor.scheduleWithFixedDelay(runnable, initialDelayMillis, delayMillis, TimeUnit.MILLISECONDS);
    }

    protected Runnable getPollingRunnable(final PolledConfigurationSource source, final Configuration config) {
        return new Runnable() {
            public void run() {
                log.debug("Polling started");
                PollResult result = null;
                try {
                    //重点是这里，拉取配置结果
                    result = source.poll(false, getNextCheckPoint(checkPoint));
                    checkPoint = result.getCheckPoint();
                    fireEvent(EventType.POLL_SUCCESS, result, null);
                } catch (Throwable e) {
                    log.error("Error getting result from polling source", e);
                    fireEvent(EventType.POLL_FAILURE, null, e);
                    return;
                }
                try {
                    //更新配置
                    populateProperties(result, config);
                } catch (Throwable e) {
                    log.error("Error occured applying properties", e);
                }                 
            }

        };   
    }
```
以上代码是先初始化，从url中拉取配置，然后开启一个线程，间隔一段时间拉取，然后调用populateProperties方法更新配置。  
source.poll代码比较简单，获取url资源，得到Properties，转换为map，封装在`PollResult`中

```
    public PollResult poll(boolean initial, Object checkPoint)
            throws IOException {    
        if (configUrls == null || configUrls.length == 0) {
            return PollResult.createFull(null);
        }
        Map<String, Object> map = new HashMap<String, Object>();
        for (URL url: configUrls) {
            InputStream fin = url.openStream();
            Properties props = ConfigurationUtils.loadPropertiesFromInputStream(fin);
            for (Entry<Object, Object> entry: props.entrySet()) {
                map.put((String) entry.getKey(), entry.getValue());
            }
        }
        return PollResult.createFull(map);
    }
```

populateProperties更新配置的代码：
```
protected void populateProperties(final PollResult result, final Configuration config) {
        if (result == null || !result.hasChanges()) {
            return;
        }
        if (!result.isIncremental()) {
            //不是增量结果，即全量结果，默认使用的是全量结果
            Map<String, Object> props = result.getComplete();
            if (props == null) {
                return;
            }
            //遍历每个配置项，进行更新
            for (Entry<String, Object> entry: props.entrySet()) {
                propertyUpdater.addOrChangeProperty(entry.getKey(), entry.getValue(), config);
            }
            //下面这段逻辑是删除配置项
            HashSet<String> existingKeys = new HashSet<String>();
            for (Iterator<String> i = config.getKeys(); i.hasNext();) {
                existingKeys.add(i.next());
            }
            if (!ignoreDeletesFromSource) {
                for (String key: existingKeys) {
                    if (!props.containsKey(key)) {
                        propertyUpdater.deleteProperty(key, config);
                    }
                }
            }
        } else {
            //增量结果，默认情况下是全量，一般不会走这里
            。。。
        }
    }
```
先遍历所有配置项进行更新，然后处理删除配置项。通过propertyUpdater.addOrChangeProperty更新配置型，代码：

```
void addOrChangeProperty(final String name, final Object newValue, final Configuration config) {
        // We do not want to abort the operation due to failed validation on one property
        try {
            //config中不包含配置项就增加
            if (!config.containsKey(name)) {
                logger.debug("adding property key [{}], value [{}]", name, newValue);

                config.addProperty(name, newValue);
            } else {
                //包含的话就比较
                Object oldValue = config.getProperty(name);

                if (newValue != null) {
                    Object newValueArray;
                    //数组处理
                    if (oldValue instanceof CopyOnWriteArrayList && AbstractConfiguration.getDefaultListDelimiter() != '\0'){
                        newValueArray =
                                        new CopyOnWriteArrayList();

                      Iterable<String> stringiterator = Splitter.on(AbstractConfiguration.getDefaultListDelimiter()).omitEmptyStrings().trimResults().split((String)newValue);
                      for(String s :stringiterator){
                            ((CopyOnWriteArrayList) newValueArray).add(s);
                        }
                      } else {
                          newValueArray = newValue;
                      }
                    //不相同修改配置
                    if (!newValueArray.equals(oldValue)) {
                        logger.debug("updating property key [{}], value [{}]", name, newValue);

                        config.setProperty(name, newValue);
                    }

                } else if (oldValue != null) {
                    logger.debug("nulling out property key [{}]", name);

                    config.setProperty(name, null);
                }
            }
        } catch (ValidationException e) {
            logger.warn("Validation failed for property " + name, e);
        }
    }
```
更新配置项的过程是拿出老的配置值跟新的配置值进行比较，不相同才修改，还处理了下数组配置项。这里的config是`DynamicURLConfiguration`，setProperty方法中触发事件：
```
    @Override
    public void setProperty(String key, Object value) throws ValidationException
    {
        if (value == null) {
            throw new NullPointerException("Value for property " + key + " is null");
        }
        fireEvent(EVENT_SET_PROPERTY, key, value, true);
        setPropertyImpl(key, value);
        fireEvent(EVENT_SET_PROPERTY, key, value, false);
    }
```
setPropertyImpl方法修改`DynamicURLConfiguration`内部保存的配置项，这里虽然改了`DynamicURLConfiguration`配置项值，但是真正用到的地方存在`DynamicProperty`中，触发fireEvent方法才会去修改`DynamicProperty`中的值：
```
    @Override
    protected void fireEvent(int type, String propName, Object propValue, boolean beforeUpdate) {
        if (listeners == null || listeners.size() == 0) {
            return;
        }
        ConfigurationEvent event = createEvent(type, propName, propValue, beforeUpdate);
        //通知每个监听器，这里listeners是一个CopyOnWriteArrayList类型，防止在遍历过程中更新数据产生问题
        for (ConfigurationListener l: listeners)
        {
            try {
                l.configurationChanged(event);
            } catch (ValidationException e) {
                if (beforeUpdate) {
                    throw e;
                } else {
                    logger.error("Unexpected exception", e);                    
                }
            } catch (Throwable e) {
                logger.error("Error firing configuration event", e);
            }
        }
    }
```
`DynamicURLConfiguration`添加到`ConcurrentCompositeConfiguration`中时，`ConcurrentCompositeConfiguration`会给每个Configuration增加一个`ConfigurationListener eventPropagater`:
```
public void addConfigurationAtIndex(AbstractConfiguration config, String name, int index)
    throws IndexOutOfBoundsException {
        if (!configList.contains(config)) {
            checkIndex(index);
            configList.add(index, config);
            if (name != null) {
                namedConfigurations.put(name, config);
            }
            //将自己的属性eventPropagater添加到config的监听列表中
            config.addConfigurationListener(eventPropagater);
            fireEvent(EVENT_CONFIGURATION_SOURCE_CHANGED, null, null, false);
        } else {
            logger.warn(config + " is not added as it already exits");
        }
    }
```
`DynamicURLConfiguration`中就有一个`ConfigurationListener`的监听器：

```
    //ConcurrentCompositeConfiguration监听器属性初始化
    private ConfigurationListener eventPropagater = new ConfigurationListener() {
        @Override
        public void configurationChanged(ConfigurationEvent event) {
            boolean beforeUpdate = event.isBeforeUpdate();
            if (propagateEventToParent) {
                int type = event.getType();
                String name = event.getPropertyName();
                Object value = event.getPropertyValue();
                Object finalValue;
                switch(type) {
                case HierarchicalConfiguration.EVENT_ADD_NODES:
                case EVENT_CLEAR:
                case EVENT_CONFIGURATION_SOURCE_CHANGED:
                    fireEvent(type, name, value, beforeUpdate);
                    break;

                case EVENT_ADD_PROPERTY:
                case EVENT_SET_PROPERTY:
                    //不管是beforeUpdate是true还是false，都会调用自己的fireEvent方法
                    if (beforeUpdate) {
                        // we want the validators to run even if the source is not
                        // the winning configuration
                        fireEvent(type, name, value, beforeUpdate);                                                
                    } else {
                        AbstractConfiguration sourceConfig = (AbstractConfiguration) event.getSource();
                        AbstractConfiguration winningConf = (AbstractConfiguration) getSource(name);
                        if (winningConf == null || getIndexOfConfiguration(sourceConfig) <= getIndexOfConfiguration(winningConf)) {
                            fireEvent(type, name, value, beforeUpdate);                        
                        }
                    }
                    break;
                case EVENT_CLEAR_PROPERTY:
                    finalValue = ConcurrentCompositeConfiguration.this.getProperty(name);
                    if (finalValue == null) {
                        fireEvent(type, name, value, beforeUpdate);                        
                    } else {
                        fireEvent(EVENT_SET_PROPERTY, name, finalValue, beforeUpdate);
                    }
                    break;
                default:
                    break;

                }
            }            
        }        
    };
```
可以看到，最后都调用`ConcurrentCompositeConfiguration`的fireEvent方法，fireEvent是从父类继承过来的，作用是调用所有监听器，那么`ConcurrentCompositeConfiguration`的监听器有哪些呢？我们调试过程中发现有两个：

- ExpandedConfigurationListenerAdapter 配置监听适配器
- ConfigurationBasedDeploymentContext.configListener 部署监听器，环境配置发生改变时，修改环境配置项

重要的是这个`ExpandedConfigurationListenerAdapter`，作用是将arhaius定义的监听器适配到Apache定义的监听器上，一个典型的适配器模式。`ExpandedConfigurationListenerAdapter`含有一个`PropertyListener expandedListener`属性，实现类是`DynamicProperty`的内部类`DynamicPropertyListener`。初始化的地方在`DynamicPropertyFactory`方法setDirect中：
```
    static void setDirect(DynamicPropertySupport support) {
        synchronized (ConfigurationManager.class) {
            config = support;
            DynamicProperty.registerWithDynamicPropertySupport(support);
            initializedWithDefaultConfig = false;
        }
    }
```
`DynamicProperty.registerWithDynamicPropertySupport`方法：

```
    static void registerWithDynamicPropertySupport(DynamicPropertySupport config) {
        initialize(config);
    }

    static synchronized void initialize(DynamicPropertySupport config) {
        dynamicPropertySupportImpl = config;
        //这里新建DynamicPropertyListener增加到config中
        config.addConfigurationListener(new DynamicPropertyListener());
        updateAllProperties();
    }
```
这里的config实现类是`ConfigurationBackedDynamicPropertySupportImpl`，同样是一个适配器模式，将apache的`DynamicPropertySupport`适配为archaius自己的`DynamicPropertySupport`，在`DynamicProperty`里面会用到。我们给出类图：

![](/images/archaius/listener.png)

根据前面的分析，fireEvent触发的事件最终会适配到`DynamicPropertyListener`的setProperty方法：
```
        //事件触发中调用setProperty
        @Override
        public void setProperty(Object source, String name, Object value, boolean beforeUpdate) {
            if (!beforeUpdate) {
                updateProperty(name, value);
            } else {
                validate(name, value);
            }
        }
        //更新配置
        private static boolean updateProperty(String propName, Object value) {
            //从ALL_PROPS中找出动态配置，更新value
            DynamicProperty prop = ALL_PROPS.get(propName);
            if (prop != null && prop.updateValue(value)) {
                //通知回调
                prop.notifyCallbacks();
                return true;
            }
            return false;
        }

        //更新stringValue值，然后刷新缓存
        boolean updateValue(Object newValue) {
            String nv = (newValue == null) ? null : newValue.toString();
            synchronized (lock) {
                if ((nv == null && stringValue == null)
                   || (nv != null && nv.equals(stringValue))) {
                    return false;
                }
                stringValue = nv;
                cachedStringValue.flush();
                booleanValue.flush();
                integerValue.flush();
                floatValue.flush();
                classValue.flush();
                doubleValue.flush();
                longValue.flush();
                changedTime = System.currentTimeMillis();
                return true;
            }
        }
```
看到这里，才是真正的将`DynamicProperty`值更新了，`DynamicLongProperty`在更新之后获取数据时，会使用新值。  

接下来我们来看回调，`DynamicProperty`的notifyCallbacks方法：
```
    private void notifyCallbacks() {
        //遍历所有的callbacks，执行run
        for (Runnable r : callbacks) {
            try {
                r.run();
            } catch (Exception e) {
                logger.error("Error in DynamicProperty callback", e);
            }
        }
    }
```
这里callbacks是`DynamicProperty`的属性，在给动态配置添加回调函数时，直接添加到`DynamicProperty.callbacks`属性中，来看`PropertyWrapper`类的addCallback犯法：

```
public void addCallback(Runnable callback) {
        if (callback != null) {
            //这里的prop就是DynamicProperty
            prop.addCallback(callback);
            callbackList.add(callback);
        }
    }
```
到这里，动态更新和回调就分析完了。  

有点复杂，我们再捋一遍：
1. `DynamicURLConfiguration`中的线程间隔一段时间从url中拉取配置
2. 拉取配置后调用`AbstractPollingScheduler`类的populateProperties方法更新配置
3. populateProperties方法中调用属性`DynamicPropertyUpdater propertyUpdater`的addOrChangeProperty方法
4. `DynamicPropertyUpdater`的方法addOrChangeProperty会进行新老配置型对比，调用`DynamicURLConfiguration`的setProperty方法修改变更项
5. `DynamicURLConfiguration`的setProperty方法先修改本身存放的配置项，然后触发事件`fireEvent`
6. `DynamicURLConfiguration`中的监听器是`ConcurrentCompositeConfiguration`的属性`ConfigurationListener eventPropagater`,`ConcurrentCompositeConfiguration`每次增加配置源是会给每个源添加eventPropagater监听器
7. `DynamicURLConfiguration`的fireEvent会通过eventPropagater触发`ConcurrentCompositeConfiguration`的fireEvent
8. `ConcurrentCompositeConfiguration`的监听器包含一个`ExpandedConfigurationListenerAdapter`的适配器，包含一个`PropertyListener`的属性，将事件转到`PropertyListener`中
9. `PropertyListener`的实现类是`DynamicProperty`的内部类`DynamicPropertyListener`，里面会调用`DynamicProperty`的静态方法`updateProperty`
10. `DynamicProperty`的`updateProperty`方法先从`ALL_PROPS`中获取配置`DynamicProperty`，更新配置项，然后通知所有的回调。
