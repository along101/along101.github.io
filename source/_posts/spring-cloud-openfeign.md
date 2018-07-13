---
title: spring-cloud-openfeign 深度分析
date: 2018-06-15 12:03:13
tags: [spring-cloud,feign,rest]
categories: spring
---

# 简介
[feign](https://github.com/OpenFeign/feign)是一个声明试的HTTP客户端，spring-cloud-openfeign将feign集成到spring boot中，在接口上通过注解声明Rest协议，将http调用转换为接口方法的调用，使得客户端调用http服务更加简单。

当前spring cloud最新稳定版本是Edgware，feign在其集成的[spring-cloud-netflix](https://github.com/spring-cloud/spring-cloud-netflix) 1.4.0.RELEASE版本中。
> spring cloud下一个版本是Finchley，将会单独集成[spring-cloud-openfeign](https://github.com/spring-cloud/spring-cloud-openfeign)

<!--more-->

# demo
我们来看个简单的例子。源代码链接：[https://github.com/along101/spring-boot-test/tree/master/feign-test](https://github.com/along101/spring-boot-test/tree/master/feign-test)

## 服务端代码
使用spring boot编写一个简单的Rest服务
```java
@RestController
public class HelloController implements HelloService {
    @Override
    public String hello(@RequestParam("name") String name) {
        return "Hello " + name;
    }
}
```
接口代码：
```java
@RequestMapping("/test")
public interface HelloService {
    @RequestMapping(value = "/hello1", method = RequestMethod.GET)
    String hello(@RequestParam("name") String name);
}
```
代码很简单，通过springMVC注解在接口HelloService上声明Rest服务，HelloController被@RestController注解声明为一个Rest服务。

启动spring boot 就可通过浏览器访问http://localhost:8080/test/hello1?name=ppdai得到返回Hello ppdai。

## 客户端代码
客户端pom中需要加入spring-cloud-starter-feign的依赖
```xml
	<dependencies>
  ...
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-feign</artifactId>
		</dependency>
	</dependencies>

  	<dependencyManagement>
  		<dependencies>
  			<dependency>
  				<groupId>org.springframework.cloud</groupId>
  				<artifactId>spring-cloud-dependencies</artifactId>
  				<version>Camden.SR7</version>
  				<type>pom</type>
  				<scope>import</scope>
  			</dependency>
  		</dependencies>
  	</dependencyManagement>
```
在客户端中新建一个HelloClient接口继承服务端HelloService接口：

```java
//在spring boot配置文件中配置remote.hello.service.host=http://localhost:8080
@FeignClient(value = "HELLO-SERVICE", url = "${remote.hello.service.host}")
public interface HelloClient extends HelloService {

}
```
HelloClient接口上注解@FeignClient，声明为Feign的客户端，参数url指定服务端地址。
在spring boot启动类上增加注解@EnableFeignClients

```java
@SpringBootApplication
@EnableFeignClients
public class FeignClientApplication {
    public static void main(String[] args) {
        SpringApplication.run(FeignClientApplication.class, args);
    }
}
```
注意，HelloClient接口需要在启动类package或者子package之下。
编写测试类测试：
```java
@RunWith(SpringJUnit4ClassRunner.class)
@SpringBootTest(classes = FeignClientApplication.class)
public class HelloClientTest {
    @Autowired
    private HelloClient helloClient;

    @Test
    public void testClient() throws Exception {
        String result = helloClient.hello("ppdai");
        System.out.println(result);
    }
}
```
启动服务端后，运行该测试类，在控制台会打印出`Hello ppdai`

# 原理分析

看到客户端测试类中，我们只用了一行代码，就能完成对远程Rest服务的调用，相当的简单。为什么这么神奇，这几段代码是如何做到的呢？

## @EnableFeignClients 注解声明客户端接口
入口是启动类上的注解@EnableFeignClients，源代码：

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Import(FeignClientsRegistrar.class)
public @interface EnableFeignClients {
	//basePackages的别名
	String[] value() default {};
	//声明基础包，spring boot启动后，会扫描该包下被@FeignClient注解的接口
	String[] basePackages() default {};
	//声明基础包的类，通过该类声明基础包
	Class<?>[] basePackageClasses() default {};
	//默认配置类
	Class<?>[] defaultConfiguration() default {};
	//直接声明的客户端接口类
	Class<?>[] clients() default {};
}
```
@EnableFeignClients的参数声明客户端接口的位置和默认的配置类。

### @FeignClient注解，将接口声明为Feign客户端
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface FeignClient {

	@AliasFor("name")
	String value() default "";
	//名称，对应与eureka上注册的应用名
	@AliasFor("value")
	String name() default "";
	//生成spring bean的qualifier
	String qualifier() default "";
	//http服务的url
	String url() default "";
	boolean decode404() default false;
	//配置类，这里设置的配置类是Spring Configuration，将会在FeignContext中创建内部声明的Bean，用于不同的客户端进行隔离
	Class<?>[] configuration() default {};
	//声明hystrix调用失败后的方法
	Class<?> fallback() default void.class;
	Class<?> fallbackFactory() default void.class;
	String path() default "";
}
```

## FeignClientsRegistrar 注册客户端

@EnableFeignClients注解上被注解了@Import(FeignClientsRegistrar.class)，@Import注解的作用是将指定的类作为Bean注入到Spring Context中，我们再来看被引入的FeignClientsRegistrar

```java

class FeignClientsRegistrar implements ImportBeanDefinitionRegistrar,
		ResourceLoaderAware, BeanClassLoaderAware {
。。。

	@Override
	public void registerBeanDefinitions(AnnotationMetadata metadata,
			BeanDefinitionRegistry registry) {
		registerDefaultConfiguration(metadata, registry);
		registerFeignClients(metadata, registry);
	}
。。。
}
```
FeignClientsRegistrar类实现了3个接口:
- 接口ResourceLoaderAware用于注入ResourceLoader
- 接口BeanClassLoaderAware用于注入ClassLoader
- 接口ImportBeanDefinitionRegistrar用于动态向Spring Context中注册bean

ImportBeanDefinitionRegistrar接口方法registerBeanDefinitions有两个参数
- AnnotationMetadata 包含被@Import注解类的信息

> 这里 @Import注解在@EnableFeignClients上，@EnableFeignClients注解在spring boot启动类上，AnnotationMetadata拿到的是spring boot启动类的相关信息

- BeanDefinitionRegistry bean定义注册中心

## registerDefaultConfiguration方法，注册默认配置
registerDefaultConfiguration方法代码：

```java

	private void registerDefaultConfiguration(AnnotationMetadata metadata,
			BeanDefinitionRegistry registry) {
		//获取@EnableFeignClients注解参数
		Map<String, Object> defaultAttrs = metadata
				.getAnnotationAttributes(EnableFeignClients.class.getName(), true);
		//如果参数中包含defaultConfiguration
		if (defaultAttrs != null && defaultAttrs.containsKey("defaultConfiguration")) {
			String name;
			if (metadata.hasEnclosingClass()) {
				name = "default." + metadata.getEnclosingClassName();
			}
			else {
				name = "default." + metadata.getClassName();
			}
			//注册客户端的配置Bean
			registerClientConfiguration(registry, name,
					defaultAttrs.get("defaultConfiguration"));
		}
	}
```
取出@EnableFeignClients注解参数defaultConfiguration，注册到spring Context中。registerClientConfiguration方法代码如下：

```java

	private void registerClientConfiguration(BeanDefinitionRegistry registry, Object name,
			Object configuration) {
		// 创建一个BeanDefinitionBuilder，注册bean的类为FeignClientSpecification
		BeanDefinitionBuilder builder = BeanDefinitionBuilder
				.genericBeanDefinition(FeignClientSpecification.class);
		//增加构造函数参数
		builder.addConstructorArgValue(name);
		builder.addConstructorArgValue(configuration);
		//调用BeanDefinitionRegistry.registerBeanDefinition方法动态注册Bean
		registry.registerBeanDefinition(
				name + "." + FeignClientSpecification.class.getSimpleName(),
				builder.getBeanDefinition());
	}
```
这里使用spring 动态注册bean的方式，注册了一个FeignClientSpecification的bean。

##  FeignClientSpecification 客户端定义
一个简单的pojo，继承了NamedContextFactory.Specification，两个属性String name 和 Class<?>[] configuration，用于FeignContext命名空间独立配置，后面会用到。

```java
@Data
@AllArgsConstructor
@NoArgsConstructor
class FeignClientSpecification implements NamedContextFactory.Specification {

	private String name;

	private Class<?>[] configuration;

}
```

## registerFeignClients方法，注册feign客户端

```java

	public void registerFeignClients(AnnotationMetadata metadata,
			BeanDefinitionRegistry registry) {
		//生成一个scanner，扫描注定包下的类
		ClassPathScanningCandidateComponentProvider scanner = getScanner();
		scanner.setResourceLoader(this.resourceLoader);

		Set<String> basePackages;

		Map<String, Object> attrs = metadata
				.getAnnotationAttributes(EnableFeignClients.class.getName());
		//包含@FeignClient注解的过滤器
		AnnotationTypeFilter annotationTypeFilter = new AnnotationTypeFilter(
				FeignClient.class);
		final Class<?>[] clients = attrs == null ? null
				: (Class<?>[]) attrs.get("clients");
		if (clients == null || clients.length == 0) {
		//@EnableFeignClients没有声明clients，获取basePackages，设置过滤器
			scanner.addIncludeFilter(annotationTypeFilter);
			basePackages = getBasePackages(metadata);
		}
		else {
			//@EnableFeignClients声明了clients
			final Set<String> clientClasses = new HashSet<>();
			basePackages = new HashSet<>();
			//basePackages为声明的clients所在的包
			for (Class<?> clazz : clients) {
				basePackages.add(ClassUtils.getPackageName(clazz));
				clientClasses.add(clazz.getCanonicalName());
			}
			//增加过滤器，只包含声明的clients
			AbstractClassTestingTypeFilter filter = new AbstractClassTestingTypeFilter() {
				@Override
				protected boolean match(ClassMetadata metadata) {
					String cleaned = metadata.getClassName().replaceAll("\\$", ".");
					return clientClasses.contains(cleaned);
				}
			};
			scanner.addIncludeFilter(
					new AllTypeFilter(Arrays.asList(filter, annotationTypeFilter)));
		}
		//遍历basePackages
		for (String basePackage : basePackages) {
			//扫描包，根据过滤器找到候选的Bean
			Set<BeanDefinition> candidateComponents = scanner
					.findCandidateComponents(basePackage);
			// 遍历候选的bean
			for (BeanDefinition candidateComponent : candidateComponents) {
				if (candidateComponent instanceof AnnotatedBeanDefinition) {
					// 校验注解是否是注解在接口上
					AnnotatedBeanDefinition beanDefinition = (AnnotatedBeanDefinition) candidateComponent;
					AnnotationMetadata annotationMetadata = beanDefinition.getMetadata();
					Assert.isTrue(annotationMetadata.isInterface(),
							"@FeignClient can only be specified on an interface");
					// 获取注解属性
					Map<String, Object> attributes = annotationMetadata
							.getAnnotationAttributes(
									FeignClient.class.getCanonicalName());

					String name = getClientName(attributes);
					//注册客户端配置
					registerClientConfiguration(registry, name,
							attributes.get("configuration"));
					//注册客户端
					registerFeignClient(registry, annotationMetadata, attributes);
				}
			}
		}
	}
```
这个方法主要逻辑是扫描注解声明的客户端，调用registerFeignClient方法注册到registry中。这里是一个典型的spring动态注册bean的例子，可以参考这段代码在spring中轻松的实现类路径下class扫描，动态注册bean到spring中。想了解spring类的扫描机制，可以断点到ClassPathScanningCandidateComponentProvider.findCandidateComponents方法中，一步步调试。

## registerFeignClient方法，注册单个客户feign端

```java
	private void registerFeignClient(BeanDefinitionRegistry registry,
			AnnotationMetadata annotationMetadata, Map<String, Object> attributes) {
		String className = annotationMetadata.getClassName();
		//构建一个FeignClientFactoryBean的bean工厂定义
		BeanDefinitionBuilder definition = BeanDefinitionBuilder
				.genericBeanDefinition(FeignClientFactoryBean.class);
		validate(attributes);
		//根据@FeignClient注解的参数，设置属性
		definition.addPropertyValue("url", getUrl(attributes));
		definition.addPropertyValue("path", getPath(attributes));
		String name = getName(attributes);
		definition.addPropertyValue("name", name);
		definition.addPropertyValue("type", className);
		definition.addPropertyValue("decode404", attributes.get("decode404"));
		definition.addPropertyValue("fallback", attributes.get("fallback"));
		definition.addPropertyValue("fallbackFactory", attributes.get("fallbackFactory"));
		definition.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE);

		String alias = name + "FeignClient";
		AbstractBeanDefinition beanDefinition = definition.getBeanDefinition();
		beanDefinition.setPrimary(true);
		//设置qualifier
		String qualifier = getQualifier(attributes);
		if (StringUtils.hasText(qualifier)) {
			alias = qualifier;
		}
		//注册，这里为了简写，新建一个BeanDefinitionHolder，调用BeanDefinitionReaderUtils静态方法注册
		BeanDefinitionHolder holder = new BeanDefinitionHolder(beanDefinition, className,
				new String[] { alias });
		BeanDefinitionReaderUtils.registerBeanDefinition(holder, registry);
	}
```
registerFeignClient方法主要是将FeignClientFactoryBean工厂Bean注册到registry中，spring初始化后，会调用FeignClientFactoryBean的getObject方法创建bean注册到spring context中。

## FeignClientFactoryBean 创建feign客户端的工厂

```java
@Data
@EqualsAndHashCode(callSuper = false)
class FeignClientFactoryBean implements FactoryBean<Object>, InitializingBean,
		ApplicationContextAware {
	//feign客户端接口类
	private Class<?> type;
	private String name;  
	private String url;

	private String path;

	private boolean decode404;

	private ApplicationContext applicationContext;
	//hystrix集成，调用失败的执行方法
	private Class<?> fallback = void.class;
	//同上
	private Class<?> fallbackFactory = void.class;
。。。
}
```
FeignClientFactoryBean实现了FactoryBean接口，是一个工厂bean

### FeignClientFactoryBean.getObject方法

```java
	@Override
	public Object getObject() throws Exception {
		//FeignContext在FeignAutoConfiguration中自动注册，FeignContext用于客户端配置类独立注册，后面具体分析
		FeignContext context = applicationContext.getBean(FeignContext.class);
		//创建Feign.Builder
		Feign.Builder builder = feign(context);
		//如果@FeignClient注解没有设置url参数
		if (!StringUtils.hasText(this.url)) {
			String url;
			//url为@FeignClient注解的name参数
			if (!this.name.startsWith("http")) {
				url = "http://" + this.name;
			}
			else {
				url = this.name;
			}
			//加上path
			url += cleanPath();
			//返回loadBlance客户端，也就是ribbon+eureka的客户端
			return loadBalance(builder, context, new HardCodedTarget<>(this.type,
					this.name, url));
		}
		//@FeignClient设置了url参数，不做负载均衡
		if (StringUtils.hasText(this.url) && !this.url.startsWith("http")) {
			this.url = "http://" + this.url;
		}
		//加上path
		String url = this.url + cleanPath();
		//从FeignContext中获取client
		Client client = getOptional(context, Client.class);
		if (client != null) {
			if (client instanceof LoadBalancerFeignClient) {
				// 有url参数，不做负载均衡，但是客户端是ribbon，或者实际的客户端
				client = ((LoadBalancerFeignClient)client).getDelegate();
			}
			builder.client(client);
		}
		//从FeignContext中获取Targeter
		Targeter targeter = get(context, Targeter.class);
		//生成客户端代理
		return targeter.target(this, builder, context, new HardCodedTarget<>(
				this.type, this.name, url));
	}
```
这段代码有个比较重要的逻辑，如果在@FeignClient注解中设置了url参数，就不走Ribbon，直接url调用，否则通过Ribbon调用，实现客户端负载均衡。

可以看到，生成Feign客户端所需要的各种配置对象，都是通过FeignContex中获取的。

### FeignContext 隔离配置
在@FeignClient注解参数configuration，指定的类是Spring的Configuration Bean，里面方法上加@Bean注解实现Bean的注入，可以指定feign客户端的各种配置，包括Encoder/Decoder/Contract/Feign.Builder等。不同的客户端指定不同配置类，就需要对配置类进行隔离，FeignContext就是用于隔离配置的。

```java
public class FeignContext extends NamedContextFactory<FeignClientSpecification> {

	public FeignContext() {
		super(FeignClientsConfiguration.class, "feign", "feign.client.name");
	}
}
```
FeignContext继承NamedContextFactory，空参数构造函数指定FeignClientsConfiguration类为默认配置。
NamedContextFactory实现接口ApplicationContextAware，注入ApplicationContextAware作为parent：
```java

public abstract class NamedContextFactory<C extends NamedContextFactory.Specification>
		implements DisposableBean, ApplicationContextAware {
	//命名空间对应的Spring Context
	private Map<String, AnnotationConfigApplicationContext> contexts = new ConcurrentHashMap<>();
	//不同命名空间的定义
	private Map<String, C> configurations = new ConcurrentHashMap<>();
	//父ApplicationContext，通过ApplicationContextAware接口注入
	private ApplicationContext parent;
	//默认配置类
	private Class<?> defaultConfigType;
	private final String propertySourceName;
	private final String propertyName;
。。。
	//设置配置，在FeignAutoConfiguration中将Spring Context中的所有FeignClientSpecification设置进来，如果@EnableFeignClients有设置参数defaultConfiguration也会加进来，前面已经分析在registerDefaultConfiguration方法中注册的FeignClientSpecification Bean
	public void setConfigurations(List<C> configurations) {
		for (C client : configurations) {
			this.configurations.put(client.getName(), client);
		}
	}

	//获取指定命名空间的ApplicationContext，先从缓存中获取，没有就创建
	protected AnnotationConfigApplicationContext getContext(String name) {
		if (!this.contexts.containsKey(name)) {
			synchronized (this.contexts) {
				if (!this.contexts.containsKey(name)) {
					this.contexts.put(name, createContext(name));
				}
			}
		}
		return this.contexts.get(name);
	}

	//创建ApplicationContext
	protected AnnotationConfigApplicationContext createContext(String name) {
		//新建AnnotationConfigApplicationContext
		AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();
		//根据name在configurations找到所有的配置类，注册到context总
		if (this.configurations.containsKey(name)) {
			for (Class<?> configuration : this.configurations.get(name)
					.getConfiguration()) {
				context.register(configuration);
			}
		}
		//将default.开头的默认默认也注册到Context中
		for (Map.Entry<String, C> entry : this.configurations.entrySet()) {
			if (entry.getKey().startsWith("default.")) {
				for (Class<?> configuration : entry.getValue().getConfiguration()) {
					context.register(configuration);
				}
			}
		}
		//注册一些需要的bean
		context.register(PropertyPlaceholderAutoConfiguration.class,
				this.defaultConfigType);
		context.getEnvironment().getPropertySources().addFirst(new MapPropertySource(
				this.propertySourceName,
				Collections.<String, Object> singletonMap(this.propertyName, name)));
		if (this.parent != null) {
		// 设置parent
			context.setParent(this.parent);
		}
		//刷新，完成bean生成
		context.refresh();
		return context;
	}

	//从命名空间中获取指定类型的Bean
	public <T> T getInstance(String name, Class<T> type) {
		AnnotationConfigApplicationContext context = getContext(name);
		if (BeanFactoryUtils.beanNamesForTypeIncludingAncestors(context,
				type).length > 0) {
			return context.getBean(type);
		}
		return null;
	}

	//从命名空间中获取指定类型的Bean
	public <T> Map<String, T> getInstances(String name, Class<T> type) {
		AnnotationConfigApplicationContext context = getContext(name);
		if (BeanFactoryUtils.beanNamesForTypeIncludingAncestors(context,
				type).length > 0) {
			return BeanFactoryUtils.beansOfTypeIncludingAncestors(context, type);
		}
		return null;
	}

}
```
关键的方法是createContext，为每个命名空间独立创建ApplicationContext，设置parent为外部传入的Context，这样就可以共用外部的Context中的Bean，又有各种独立的配置Bean，熟悉springMVC的同学应该知道，springMVC中创建的WebApplicatonContext里面也有个parent，原理跟这个类似。

从FeignContext中获取Bean，需要传入命名空间，根据命名空间找到缓存中的ApplicationContext，先从自己注册的Bean中获取bean，没有获取到再从到parent中获取。

### 创建Feign.Builder

了解了FeignContext的原理，我们再来看feign最重要的构建类创建过程
```java
	protected Feign.Builder feign(FeignContext context) {
		。。。
		//从FeignContext中获取注册的Feign.Builder bean，设置Encoder/Decoder/Contract
		Feign.Builder builder = get(context, Feign.Builder.class)
				.logger(logger)
				.encoder(get(context, Encoder.class))
				.decoder(get(context, Decoder.class))
				.contract(get(context, Contract.class));
		。。。
		//设置feign其他参数，都从FeignContext中获取
		Retryer retryer = getOptional(context, Retryer.class);
		if (retryer != null) {
			builder.retryer(retryer);
		}
		ErrorDecoder errorDecoder = getOptional(context, ErrorDecoder.class);
		if (errorDecoder != null) {
			builder.errorDecoder(errorDecoder);
		}
		Request.Options options = getOptional(context, Request.Options.class);
		if (options != null) {
			builder.options(options);
		}
		Map<String, RequestInterceptor> requestInterceptors = context.getInstances(
				this.name, RequestInterceptor.class);
		if (requestInterceptors != null) {
			builder.requestInterceptors(requestInterceptors.values());
		}

		if (decode404) {
			builder.decode404();
		}

		return builder;
	}

```
这里设置了Feign.Builder所必须的参数Encoder/Decoder/Contract，其他参数都是可选的。这三个必须的参数从哪里来的呢？答案是在FeignContext的构造器中，传入了默认的配置FeignClientsConfiguration，这个配置类里面初始化了这三个参数。


### FeignClientsConfiguration 客户端默认配置
```java
@Configuration
public class FeignClientsConfiguration {
	//注入springMVC的HttpMessageConverters
	@Autowired
	private ObjectFactory<HttpMessageConverters> messageConverters;
	//注解参数处理器，处理SpringMVC注解，生成http元数据
	@Autowired(required = false)
	private List<AnnotatedParameterProcessor> parameterProcessors = new ArrayList<>();
。。。
	//Decoder bean，默认通过HttpMessageConverters进行处理
	@Bean
	@ConditionalOnMissingBean
	public Decoder feignDecoder() {
		return new ResponseEntityDecoder(new SpringDecoder(this.messageConverters));
	}
	//Encoder bean，默认通过HttpMessageConverters进行处理
	@Bean
	@ConditionalOnMissingBean
	public Encoder feignEncoder() {
		return new SpringEncoder(this.messageConverters);
	}
	//Contract bean，通过SpringMvcContract进行处理接口
	@Bean
	@ConditionalOnMissingBean
	public Contract feignContract(ConversionService feignConversionService) {
		return new SpringMvcContract(this.parameterProcessors, feignConversionService);
	}
	//hystrix自动注入
	@Configuration
	@ConditionalOnClass({ HystrixCommand.class, HystrixFeign.class })
	protected static class HystrixFeignConfiguration {
		//HystrixFeign的builder，全局关掉Hystrix配置feign.hystrix.enabled=false
		@Bean
		@Scope("prototype")
		@ConditionalOnMissingBean
		@ConditionalOnProperty(name = "feign.hystrix.enabled", matchIfMissing = true)
		public Feign.Builder feignHystrixBuilder() {
			return HystrixFeign.builder();
		}
	}
	//默认不重试
	@Bean
	@ConditionalOnMissingBean
	public Retryer feignRetryer() {
		return Retryer.NEVER_RETRY;
	}
	//默认的builder
	@Bean
	@Scope("prototype")
	@ConditionalOnMissingBean
	public Feign.Builder feignBuilder(Retryer retryer) {
		return Feign.builder().retryer(retryer);
	}
。。。
}
```
可以看到，feign需要的decoder/enoder通过适配器共用springMVC中的HttpMessageConverters引入。

feign有自己的注解体系，这里通过SpringMvcContract适配了springMVC的注解体系。

### SpringMvcContract 适配feign注解体系
SpringMvcContract继承了feign的类Contract.BaseContract，作用是解析接口方法上的注解和方法参数，生成MethodMetadata用于接口方法调用过程中组装http请求。

```java
public class SpringMvcContract extends Contract.BaseContract
		implements ResourceLoaderAware {
。。。

	//处理Class上的注解
	@Override
	protected void processAnnotationOnClass(MethodMetadata data, Class<?> clz) {
	。。。
	}
	//处理方法
	@Override
	public MethodMetadata parseAndValidateMetadata(Class<?> targetType, Method method) {
		。。。
	}
	//处理方法上的注解
	@Override
	protected void processAnnotationOnMethod(MethodMetadata data,
			Annotation methodAnnotation, Method method) {
        。。。
	}
	//处理参数上的注解
  	@Override
  	protected boolean processAnnotationsOnParameter(MethodMetadata data,
  			Annotation[] annotations, int paramIndex) {
  		。。。
  	}
}
```
几个覆盖方法分别是处理类上的注解，处理方法，处理方法上的注解，处理方法参数注解，最终生成完整的MethodMetadata。feign自己提供的Contract和扩展javax.ws.rx的Contract原理都是类似的。

### Targeter 生成接口动态代理
Feign.Builder生成后，就要用Target生成feign客户端的动态代理，这里FeignClientFactoryBean中使用Targeter，Targeter有两个实现类，分别是HystrixTargeter和DefaultTargeter，DefaultTargeter很简单，直接调用HardCodedTarget生成动态代理，HystrixTargeter源码如下：

```java
class HystrixTargeter implements Targeter {

	@Override
	public <T> T target(FeignClientFactoryBean factory, Feign.Builder feign, FeignContext context,
						Target.HardCodedTarget<T> target) {
		//如果不是HystrixFeign.Builder，直接调用target生成代理
		if (!(feign instanceof feign.hystrix.HystrixFeign.Builder)) {
			return feign.target(target);
		}
		//找到fallback或者fallbackFactory，设置到hystrix中
		feign.hystrix.HystrixFeign.Builder builder = (feign.hystrix.HystrixFeign.Builder) feign;
		Class<?> fallback = factory.getFallback();
		if (fallback != void.class) {
			return targetWithFallback(factory.getName(), context, target, builder, fallback);
		}
		Class<?> fallbackFactory = factory.getFallbackFactory();
		if (fallbackFactory != void.class) {
			return targetWithFallbackFactory(factory.getName(), context, target, builder, fallbackFactory);
		}

		return feign.target(target);
	}
  。。。
}
```
到这里，接口的动态代理就生成了，然后回到FeignClientFactoryBean工厂bean中，会将动态代理注入到SpringContext，在使用的地方，就可以通过@Autowire方式注入了。

## loadBalance方法，客户端负载均衡
如果@FeignClient注解中没有配置url参数，将会通过loadBalance方法生成Ribbon的动态代理：

```java
	protected <T> T loadBalance(Feign.Builder builder, FeignContext context,
			HardCodedTarget<T> target) {
		//这里获取到的Client是LoadBalancerFeignClient
		Client client = getOptional(context, Client.class);
		if (client != null) {
			builder.client(client);
			Targeter targeter = get(context, Targeter.class);
			return targeter.target(this, builder, context, target);
		}
。。。
	}
```
LoadBalancerFeignClient在FeignRibbonClientAutoConfiguration中自动配置的Bean

### LoadBalancerFeignClient 负载均衡客户端

```java
public class LoadBalancerFeignClient implements Client {
。。。
	@Override
	public Response execute(Request request, Request.Options options) throws IOException {
		try {
			//获取URI
			URI asUri = URI.create(request.url());
			//获取客户端的名称
			String clientName = asUri.getHost();
			URI uriWithoutHost = cleanUrl(request.url(), clientName);
			//创建RibbonRequest
			FeignLoadBalancer.RibbonRequest ribbonRequest = new FeignLoadBalancer.RibbonRequest(
					this.delegate, request, uriWithoutHost);
			//配置
			IClientConfig requestConfig = getClientConfig(options, clientName);
			//获取FeignLoadBalancer，发请求，转换Response
			return lbClient(clientName).executeWithLoadBalancer(ribbonRequest,
					requestConfig).toResponse();
		} catch (ClientException e) {
			。。。
		}
	}
```
代码逻辑也比较简单，就是是配到Ribbon客户端上调用。Ribbon的相关使用和原理就不在本文中描述。

# 总结

feign本身是一款优秀的开源组件，spring cloud feign又非常巧妙的将feign集成到spring boot中。
本文通过对spring cloud feign源代码的解读，详细的分析了feign集成到spring boot中的原理，使我们更加全面的了解到feign的使用。

spring cloud feign也是一个很好的学习spring boot的例子，从中我们可以学习到：

- spring boot注解声明注入bean
- spring类扫描机制
- spring接口动态注册bean
- spring命名空间隔离ApplicationContext
