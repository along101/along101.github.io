---
title: Feign源码解析
date: 2018-06-13 12:03:13
tags: [java,feign,rest]
categories: spring
---


## Feign介绍

[Feign](https://github.com/OpenFeign/feign/)是一款java的Restful客户端组件，Feign使得 Java HTTP 客户端编写更方便。Feign 灵感来源于[Retrofit](https://github.com/square/retrofit), [JAXRS-2.0](https://jax-rs-spec.java.net/nonav/2.0/apidocs/index.html)和[WebSocket](http://www.oracle.com/technetwork/articles/java/jsr356-1937161.html)。feign在github上有近3K个star，是一款相当优秀的开源组件，虽然相比Retrofit的近30K个star，逊色了太多，但是spring cloud集成了feign，使得feign在java生态中比Retrofit使用的更加广泛。

feign的基本原理是在接口方法上加注解，定义rest请求，构造出接口的动态代理对象，然后通过调用接口方法就可以发送http请求，并且自动解析http响应为方法返回值，极大的简化了客户端调用rest api的代码。官网的示例如下：

<!--more-->

```java
interface GitHub {
  @RequestLine("GET /repos/{owner}/{repo}/contributors")
  List<Contributor> contributors(@Param("owner") String owner, @Param("repo") String repo);
}

static class Contributor {
  String login;
  int contributions;
}

public static void main(String... args) {
  GitHub github = Feign.builder()
                       .decoder(new GsonDecoder())
                       .target(GitHub.class, "https://api.github.com");

  // Fetch and print a list of the contributors to this library.
  List<Contributor> contributors = github.contributors("OpenFeign", "feign");
  for (Contributor contributor : contributors) {
    System.out.println(contributor.login + " (" + contributor.contributions + ")");
  }
}
```
feign使用教程请参考官网[https://github.com/OpenFeign/feign/](https://github.com/OpenFeign/feign/)

本文主要是对feign源码进行分析，根据源码来理解feign的设计架构和内部实现技术。  

## `Feign.build` 构建接口动态代理

我们先来看看接口的动态代理是如何构建出来的，下图是主要接口和类的类图：

![](/images/feign/init1.png)

从上文中的示例可以看到，构建的接口动态代理对象是通过`Feign.builder()`生成`Feign.Builder`的构造者对象，然后设置相关的参数，再调用target方法构造的。`Feign.Builder`的参数包括：

```java

//拦截器，组装完RequestTemplate，发请求之前的拦截处理RequestTemplate
    private final List<RequestInterceptor> requestInterceptors = new ArrayList<RequestInterceptor>();
//日志级别
    private Logger.Level logLevel = Logger.Level.NONE;
//契约模型，默认为Contract.Default，用户创建MethodMetadata，用spring cloud就是扩展这个实现springMVC注解
    private Contract contract = new Contract.Default();
//客户端，默认为Client.Default，可以扩展ApacheHttpClient，OKHttpClient，RibbonClient等
    private Client client = new Client.Default(null, null);
//重试设置，默认不设置
    private Retryer retryer = new Retryer.Default();
//日志，可以接入Slf4j
    private Logger logger = new NoOpLogger();
//编码器，用于body的编码
    private Encoder encoder = new Encoder.Default();
//解码器，用户response的解码
    private Decoder decoder = new Decoder.Default();
//用@QueryMap注解的参数编码器
    private QueryMapEncoder queryMapEncoder = new QueryMapEncoder.Default();
//请求错误解码器
    private ErrorDecoder errorDecoder = new ErrorDecoder.Default();
//参数配置，主要是超时时间之类的
    private Options options = new Options();
//动态代理工厂
    private InvocationHandlerFactory invocationHandlerFactory = new InvocationHandlerFactory.Default();
//是否decode404
    private boolean decode404;
    private boolean closeAfterDecode = true;
```

这块是一个典型的构造者模式，`target`方法内部先调用`build`方法新建一个`ReflectFeign`对象，然后调用`ReflectFeign`的`newInstance`方法创建动态代理，代码如下：

```java
    //默认使用HardCodedTarget
    public <T> T target(Class<T> apiType, String url) {
      return target(new HardCodedTarget<T>(apiType, url));
    }

    public <T> T target(Target<T> target) {
      return build().newInstance(target);
    }

    public Feign build() {
      SynchronousMethodHandler.Factory synchronousMethodHandlerFactory =
          new SynchronousMethodHandler.Factory(client, retryer, requestInterceptors, logger,
                                               logLevel, decode404, closeAfterDecode);
      ParseHandlersByName handlersByName =
          new ParseHandlersByName(contract, options, encoder, decoder, queryMapEncoder,
                                  errorDecoder, synchronousMethodHandlerFactory);
      //handlersByName将所有参数进行封装，并提供解析接口方法的逻辑
      //invocationHandlerFactory是Builder的属性，默认值是InvocationHandlerFactory.Default,用创建java动态代理的InvocationHandler实现
      return new ReflectiveFeign(handlersByName, invocationHandlerFactory, queryMapEncoder);
    }
  }
```
`ReflectiveFeign`构造函数有三个参数：
- `ParseHandlersByName` 将builder所有参数进行封装，并提供解析接口方法的逻辑
- `InvocationHandlerFactory` java动态代理的`InvocationHandler`的工厂类，默认值是`InvocationHandlerFactory.Default`
- `QueryMapEncoder`  接口参数注解`@QueryMap`时，参数的编码器

`ReflectiveFeign.newInstance`方法创建接口动态代理对象：
```java

  public <T> T newInstance(Target<T> target) {
    //targetToHandlersByName是构造器传入的ParseHandlersByName对象，根据target对象生成MethodHandler映射
    Map<String, MethodHandler> nameToHandler = targetToHandlersByName.apply(target);
    Map<Method, MethodHandler> methodToHandler = new LinkedHashMap<Method, MethodHandler>();
    List<DefaultMethodHandler> defaultMethodHandlers = new LinkedList<DefaultMethodHandler>();
    //遍历接口所有方法，构建Method->MethodHandler的映射
    for (Method method : target.type().getMethods()) {
      if (method.getDeclaringClass() == Object.class) {
        continue;
      } else if(Util.isDefault(method)) {
        //接口default方法的Handler，这类方法直接调用
        DefaultMethodHandler handler = new DefaultMethodHandler(method);
        defaultMethodHandlers.add(handler);
        methodToHandler.put(method, handler);
      } else {
        methodToHandler.put(method, nameToHandler.get(Feign.configKey(target.type(), method)));
      }
    }
    //这里factory是构造其中传入的，创建InvocationHandler
    InvocationHandler handler = factory.create(target, methodToHandler);
    //java的动态代理
    T proxy = (T) Proxy.newProxyInstance(target.type().getClassLoader(), new Class<?>[]{target.type()}, handler);
    //将default方法直接绑定到动态代理上
    for(DefaultMethodHandler defaultMethodHandler : defaultMethodHandlers) {
      defaultMethodHandler.bindTo(proxy);
    }
    return proxy;
  }
```
这段代码主要的逻辑是：
1. 创建`MethodHandler`的映射，这里创建的是实现类`SynchronousMethodHandler`
2. 通过`InvocationHandlerFatory`创建`InvocationHandler`
3. 绑定接口的`default`方法，通过`DefaultMethodHandler`绑定

类图中已经画出，`SynchronousMethodHandler`和`DefaultMethodHandler`实现了`InvocationHandlerFactory.MethodHandler`接口，动态代理对象调用方法时，如果是`default`方法，会直接调用接口方法，因为这里将接口的`default`方法绑定到动态代理对象上了，其他方法根据方法签名找到`SynchronousMethodHandler`对象，调用其`invoke`方法。

## 创建MethodHandler方法处理器

`SynchronousMethodHandler`是feign组件的核心，接口方法调用转换为http请求和解析http响应都是通过`SynchronousMethodHandler`来执行的，相关类图如下：

![](/images/feign/init2.png)

创建`MethodHandler`实现类`SynchronousMethodHandler`的代码：
```java

    public Map<String, MethodHandler> apply(Target key) {
      //通过contract解析接口方法，生成MethodMetadata列表，默认的contract解析Feign自定义的http注解
      List<MethodMetadata> metadata = contract.parseAndValidatateMetadata(key.type());
      Map<String, MethodHandler> result = new LinkedHashMap<String, MethodHandler>();
      for (MethodMetadata md : metadata) {
        //BuildTemplateByResolvingArgs实现RequestTemplate.Factory，RequestTemplate的工厂
        BuildTemplateByResolvingArgs buildTemplate;
        //根据方法元数据，使用不同的RequestTemplate的工厂
        if (!md.formParams().isEmpty() && md.template().bodyTemplate() == null) {
          //如果有formParam，并且bodyTemplate不为空，请求体为x-www-form-urlencoded格式
          //将会解析form参数，填充到bodyTemplate中
          buildTemplate = new BuildFormEncodedTemplateFromArgs(md, encoder, queryMapEncoder);
        } else if (md.bodyIndex() != null) {
          //如果包含请求体，将会用encoder编码请求体对象
          buildTemplate = new BuildEncodedTemplateFromArgs(md, encoder, queryMapEncoder);
        } else {
          //默认的RequestTemplate的工厂，没有请求体，不需要编码器
          buildTemplate = new BuildTemplateByResolvingArgs(md, queryMapEncoder);
        }
        //使用工厂SynchronousMethodHandler.Factory创建SynchronousMethodHandler
        result.put(md.configKey(),
                   factory.create(key, md, buildTemplate, options, decoder, errorDecoder));
      }
      return result;
    }
```
这段代码的逻辑是：
1. 通过`Contract`解析接口方法，生成`MethodMetadata`，默认的`Contract`解析Feign自定义的http注解
2. 根据`MethodMetadata`方法元数据生成特定的`RequestTemplate`的工厂
3. 使用`SynchronousMethodHandler.Factory`工厂创建`SynchronousMethodHandler`
这里有两个工厂不要搞混淆了，`SynchronousMethodHandler`工厂和`RequestTemplate`工厂，`SynchronousMethodHandler`的属性包含`RequestTemplate`工厂

## `Contract`解析接口方法生成`MethodMetadata`
feign默认的解析器是`Contract.Default`继承了`Contract.BaseContract`，解析生成`MethodMetadata`方法入口：
```java

    @Override
    public List<MethodMetadata> parseAndValidatateMetadata(Class<?> targetType) {
      。。。
      Map<String, MethodMetadata> result = new LinkedHashMap<String, MethodMetadata>();
      for (Method method : targetType.getMethods()) {
        。。。
        MethodMetadata metadata = parseAndValidateMetadata(targetType, method);
        。。。
        result.put(metadata.configKey(), metadata);
      }
      return new ArrayList<MethodMetadata>(result.values());
    }

    protected MethodMetadata parseAndValidateMetadata(Class<?> targetType, Method method) {
      MethodMetadata data = new MethodMetadata();
      data.returnType(Types.resolve(targetType, targetType, method.getGenericReturnType()));
      data.configKey(Feign.configKey(targetType, method));

      if(targetType.getInterfaces().length == 1) {
        processAnnotationOnClass(data, targetType.getInterfaces()[0]);
      }
      //处理Class上的注解
      processAnnotationOnClass(data, targetType);

      for (Annotation methodAnnotation : method.getAnnotations()) {
        //处理方法注解
        processAnnotationOnMethod(data, methodAnnotation, method);
      }
      。。。
      Class<?>[] parameterTypes = method.getParameterTypes();
      Type[] genericParameterTypes = method.getGenericParameterTypes();
      //方法参数注解
      Annotation[][] parameterAnnotations = method.getParameterAnnotations();
      int count = parameterAnnotations.length;
      for (int i = 0; i < count; i++) {
        boolean isHttpAnnotation = false;
        if (parameterAnnotations[i] != null) {
          isHttpAnnotation = processAnnotationsOnParameter(data, parameterAnnotations[i], i);
        }
        if (parameterTypes[i] == URI.class) {
          //参数类型是URI，后面构造http请求时，使用该URI
          data.urlIndex(i);
        } else if (!isHttpAnnotation) {
          //如果没有被http注解，就是body参数
          。。。
          data.bodyIndex(i);
          data.bodyType(Types.resolve(targetType, targetType, genericParameterTypes[i]));
        }
      }

      if (data.headerMapIndex() != null) {
        //@HeaderMap注解的参数必须是Map，key类型必须是String
        checkMapString("HeaderMap", parameterTypes[data.headerMapIndex()], genericParameterTypes[data.headerMapIndex()]);
      }

      if (data.queryMapIndex() != null) {
        if (Map.class.isAssignableFrom(parameterTypes[data.queryMapIndex()])) {
          //@QueryMap注解的参数如果是Map，key类型必须是String
          checkMapKeys("QueryMap", genericParameterTypes[data.queryMapIndex()]);
        }
      }
      return data;
    }

    protected void processAnnotationOnClass(MethodMetadata data, Class<?> targetType) {
      if (targetType.isAnnotationPresent(Headers.class)) {
        //被Headers注解
        String[] headersOnType = targetType.getAnnotation(Headers.class).value();
        。。。
        //header解析成map，加到MethodMetadata中
        Map<String, Collection<String>> headers = toMap(headersOnType);
        headers.putAll(data.template().headers());
        data.template().headers(null); // to clear
        data.template().headers(headers);
      }
    }

    protected void processAnnotationOnMethod(MethodMetadata data, Annotation methodAnnotation,
                                             Method method) {
      Class<? extends Annotation> annotationType = methodAnnotation.annotationType();
      if (annotationType == RequestLine.class) {
        //@RequestLine注解
        String requestLine = RequestLine.class.cast(methodAnnotation).value();
        。。。
        if (requestLine.indexOf(' ') == -1) {
          。。。
          data.template().method(requestLine);
          return;
        }
        //http请求方法
        data.template().method(requestLine.substring(0, requestLine.indexOf(' ')));
        if (requestLine.indexOf(' ') == requestLine.lastIndexOf(' ')) {
          // no HTTP version is ok
          data.template().append(requestLine.substring(requestLine.indexOf(' ') + 1));
        } else {
          // skip HTTP version
          data.template().append(
              requestLine.substring(requestLine.indexOf(' ') + 1, requestLine.lastIndexOf(' ')));
        }
        //将'%2F'反转为'/'
        data.template().decodeSlash(RequestLine.class.cast(methodAnnotation).decodeSlash());
        //参数集合格式化方式，默认使用key=value0&key=value1
        data.template().collectionFormat(RequestLine.class.cast(methodAnnotation).collectionFormat());

      } else if (annotationType == Body.class) {
        //@Body注解
        String body = Body.class.cast(methodAnnotation).value();
        。。。
        if (body.indexOf('{') == -1) {
          //body中不存在{，直接传入body
          data.template().body(body);
        } else {
          //body中存在{，就是bodyTemplate方式
          data.template().bodyTemplate(body);
        }
      } else if (annotationType == Headers.class) {
        //@Header注解
        String[] headersOnMethod = Headers.class.cast(methodAnnotation).value();
        。。。
        data.template().headers(toMap(headersOnMethod));
      }
    }

    //处理参数上的注解
    protected boolean processAnnotationsOnParameter(MethodMetadata data, Annotation[] annotations, int paramIndex) {
      boolean isHttpAnnotation = false;
      for (Annotation annotation : annotations) {
        Class<? extends Annotation> annotationType = annotation.annotationType();
        if (annotationType == Param.class) {
          //@Param注解
          Param paramAnnotation = (Param) annotation;
          String name = paramAnnotation.value();
          。。。
          //增加到MethodMetadata中
          nameParam(data, name, paramIndex);
          //@Param注解的expander参数，定义参数的解释器，默认是ToStringExpander，调用参数的toString方法
          Class<? extends Param.Expander> expander = paramAnnotation.expander();
          if (expander != Param.ToStringExpander.class) {
            data.indexToExpanderClass().put(paramIndex, expander);
          }
          //参数是否已经urlEncoded，如果没有，会使用urlEncoded方式编码
          data.indexToEncoded().put(paramIndex, paramAnnotation.encoded());
          isHttpAnnotation = true;
          String varName = '{' + name + '}';
          if (!data.template().url().contains(varName) &&
              !searchMapValuesContainsSubstring(data.template().queries(), varName) &&
              !searchMapValuesContainsSubstring(data.template().headers(), varName)) {
            //如果参数不在path里面，不在query里面，不在header里面，就设置到formParam中
            data.formParams().add(name);
          }
        } else if (annotationType == QueryMap.class) {
          //@QueryMap注解，注解参数对象时，将该参数转换为http请求参数格式发送
          。。。
          data.queryMapIndex(paramIndex);
          data.queryMapEncoded(QueryMap.class.cast(annotation).encoded());
          isHttpAnnotation = true;
        } else if (annotationType == HeaderMap.class) {
          //@HeaderMap注解，注解一个Map类型的参数，放入http header中发送
          。。。
          data.headerMapIndex(paramIndex);
          isHttpAnnotation = true;
        }
      }
      return isHttpAnnotation;
    }
```
代码稍微有点多，但是逻辑很清晰，先处理类上的注解，再处理方法上注解，最后处理方法参数注解，把所有注解的情况都处理到就可以了。  

生成的`MethodMetadata`的结构如下：

```java
public final class MethodMetadata implements Serializable {
  //标识方法的key，接口名加方法签名：GitHub#contributors(String,String)
  private String configKey;
//方法返回值类型
  private transient Type returnType;
//uri参数的位置，方法中可以写个uri参数，发请求时直接使用这个参数
  private Integer urlIndex;
//body参数的位置，只能有一个未注解的参数为body，否则报错
  private Integer bodyIndex;
//headerMap参数的位置
  private Integer headerMapIndex;
//@QueryMap注解参数位置
  private Integer queryMapIndex;
//@QueryMap注解里面encode参数，是否已经urlEncode编码过了
  private boolean queryMapEncoded;
//body的类型
  private transient Type bodyType;
//RequestTemplate 原型
  private RequestTemplate template = new RequestTemplate();
//form请求参数
  private List<String> formParams = new ArrayList<String>();
//方法参数位置和名称的map
  private Map<Integer, Collection<String>> indexToName ;
//@Param中注解的expander方法，可以指定解析参数类
  private Map<Integer, Class<? extends Expander>> indexToExpanderClass ;
//参数是否被urlEncode编码过了，@Param中encoded方法
  private Map<Integer, Boolean> indexToEncoded ;
//自定义的Expander
  private transient Map<Integer, Expander> indexToExpander;
```

> `Contract也`是feign的一个扩展点，一个优秀组件的架构通常是具有很强的扩展性，feign的架构本身很简单，设计的扩展点也很简单方便，所以受到spring的青睐，将其集成到spring cloud中。spring cloud就是通过`Contract`的扩展，实现使用springMVC的注解接入feign。feign自己还实现了使用jaxrs注解接入feign。

## 初始化总结

上文已经完成了feign初始化结构为动态代理的整个过程，简单的捋一遍：
1. 初始化`Feign.Builder`传入参数，构造`ReflectiveFeign`
2. `ReflectiveFeign`通过内部类`ParseHandlersByName`的`Contract`属性，解析接口生成`MethodMetadata`
3. `ParseHandlersByName`根据`MethodMetadata`生成`RequestTemplate`工厂
4. `ParseHandlersByName`创建`SynchronousMethodHandler`，传入`MethodMetadata`、`RequestTemplate`工厂和`Feign.Builder`相关参数
5. `ReflectiveFeign`创建`FeignInvocationHandler`，传入参数`SynchronousMethodHandler`，绑定`DefaultMethodHandler`
6.` ReflectiveFeign`根据`FeignInvocationHandler`创建`Proxy`

关键的几个类是：
- `ReflectiveFeign` 初始化入口
- `FeignInvocationHandler` 实现动态代理的`InvocHandler`
- `SynchronousMethodHandler` 方法处理器，方法调用处理器
- `MethodMetadata` 方法元数据

## 接口调用
为方便理解，分析完feign源码后，我将feign执行过程分成三层，如下图：

![](/images/feign/execute.png)

三层分别为：
- 代理层 动态代理调用层
- 转换层 方法转http请求，解码http响应
- 网络层 http请求发送

java动态代理接口方法调用，会调用到`InvocaHandler`的invoke方法，feign里面实现类是`FeignInvocationHandler`，invoke代码如下：

```java
    private final Map<Method, MethodHandler> dispatch;

    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
      。。。
      return dispatch.get(method).invoke(args);
    }
```

根据方法找到`MethodHandler`，除接口的`default`方法外，找到的是`SynchronousMethodHandler`对象，然后调用`SynchronousMethodHandlerd.invoke`方法：

```java

  public Object invoke(Object[] argv) throws Throwable {
    //buildTemplateFromArgs是RequestTemplate工程对象，根据方法参数创建RequestTemplate
    RequestTemplate template = buildTemplateFromArgs.create(argv);
    //重试设置
    Retryer retryer = this.retryer.clone();
    while (true) {
      try {
        //执行和解码
        return executeAndDecode(template);
      } catch (RetryableException e) {
        retryer.continueOrPropagate(e);
        。。。
        continue;
      }
    }
  }

  Object executeAndDecode(RequestTemplate template) throws Throwable {
    //RequestTemplate转换为Request
    Request request = targetRequest(template)
    。。。
    Response response;
    long start = System.nanoTime();
    try {
      response = client.execute(request, options);
      response.toBuilder().request(request).build();
    } catch (IOException e) {
      。。。
      throw errorExecuting(request, e);
    }
    long elapsedTime = TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - start);

    boolean shouldClose = true;
    try {
      。。。
      if (Response.class == metadata.returnType()) {
        //如果接口方法返回的是Response类
        if (response.body() == null) {
          //body为空，直接返回
          return response;
        }
        if (response.body().length() == null ||
                response.body().length() > MAX_RESPONSE_BUFFER_SIZE) {
          //body不为空，且length>最大缓存值，返回response，但是不能关闭response
          shouldClose = false;
          return response;
        }
        // 读取body字节数组，返回response
        byte[] bodyData = Util.toByteArray(response.body().asInputStream());
        return response.toBuilder().body(bodyData).build();
      }
      if (response.status() >= 200 && response.status() < 300) {
        //响应成功
        if (void.class == metadata.returnType()) {
          //接口返回void
          return null;
        } else {
          //解码response，直接调用decoder解码
          Object result = decode(response);
          shouldClose = closeAfterDecode;
          return result;
        }
      } else if (decode404 && response.status() == 404 && void.class != metadata.returnType()) {
        //404解析
        Object result = decode(response);
        shouldClose = closeAfterDecode;
        return result;
      } else {
        //其他返回码，使用errorDecoder解析，抛出异常
        throw errorDecoder.decode(metadata.configKey(), response);
      }
    } catch (IOException e) {
      throw errorReading(request, response, e);
    } finally {
      //是否需要关闭response，根据Feign.Builder 参数设置是否要关闭流
      if (shouldClose) {
        ensureClosed(response.body());
      }
    }
  }

```
过程比较简单，生成`RquestTemplate` -> 转换为`Request` -> `client`发请求 -> `Decoder`解析`Response`

## `RquestTemplate`构建过程

先看看`RequestTemplate`的结构：
```java
private static final long serialVersionUID = 1L;
//请求参数 ?后面的name=value
  private final Map<String, Collection<String>> queries ;
//请求头
  private final Map<String, Collection<String>> headers ;
//请求方法 GET/POST等
  private String method;
//请求路径
  private StringBuilder url = new StringBuilder();
//字符集
  private transient Charset charset;
//请求体
  private byte[] body;
//@Body("%7B\"user_name\": \"{user_name}\", \"password\": \"{password}\"%7D")注解的模板
  private String bodyTemplate;
//是否decode削减，将"%2F"反转为"/"
  private boolean decodeSlash = true;
//集合格式化，分隔符
  private CollectionFormat collectionFormat = CollectionFormat.EXPLODED;
```

在`SynchronousMethodHandler.invoke`方法中生成`RequestTemplate`
```java
    //buildTemplateFromArgs是RequestTemplate.Factory实现类
    RequestTemplate template = buildTemplateFromArgs.create(argv);
```
`RequestTemplate.Factory`有三个实现类：
- `BuildTemplateByResolvingArgs` `RequestTemplate`工厂
- `BuildEncodedTemplateFromArgs` `BuildTemplateByResolvingArgs`的子类 重载`resolve`方法，解析form表单请求
- `BuildFormEncodedTemplateFromArgs` `BuildTemplateByResolvingArgs`的子类，重载`resolve`方法，解析body请求

`BuildTemplateByResolvingArgs`创建`RequestTemplate`的`create`方法：
```java
    //BuildTemplateByResolvingArgs实现RequestTemplate.Factory的方法
    public RequestTemplate create(Object[] argv) {
      RequestTemplate mutable = new RequestTemplate(metadata.template());
      if (metadata.urlIndex() != null) {
        //插入接口方法参数中的URI
        int urlIndex = metadata.urlIndex();
        mutable.insert(0, String.valueOf(argv[urlIndex]));
      }
      Map<String, Object> varBuilder = new LinkedHashMap<String, Object>();
      //方法参数位置和请求定义的参数名称的map
      for (Entry<Integer, Collection<String>> entry : metadata.indexToName().entrySet()) {
        //将方法参数值和定义的请求参数进行映射，varBuilder
        int i = entry.getKey();
        Object value = argv[entry.getKey()];
        if (value != null) { // Null values are skipped.
          if (indexToExpander.containsKey(i)) {
            value = expandElements(indexToExpander.get(i), value);
          }
          for (String name : entry.getValue()) {
            varBuilder.put(name, value);
          }
        }
      }
      //解析RequestTemplate
      RequestTemplate template = resolve(argv, mutable, varBuilder);
      //解析queryMap，这块代码有些奇怪，为什么单独把queryMap放在这里解析，而不是在resolve方法中，或者在RequestTemplate中
      if (metadata.queryMapIndex() != null) {
        // add query map parameters after initial resolve so that they take
        // precedence over any predefined values
        Object value = argv[metadata.queryMapIndex()];
        Map<String, Object> queryMap = toQueryMap(value);
        template = addQueryMapQueryParameters(queryMap, template);
      }
      //解析headerMap定义的参数
      if (metadata.headerMapIndex() != null) {
        template = addHeaderMapHeaders((Map<String, Object>) argv[metadata.headerMapIndex()], template);
      }

      return template;
    }

    //BuildTemplateByResolvingArgs
    protected RequestTemplate resolve(Object[] argv, RequestTemplate mutable,
                                      Map<String, Object> variables) {
      // 根据需要进行urlEncode参数
      Map<String, Boolean> variableToEncoded = new LinkedHashMap<String, Boolean>();
      for (Entry<Integer, Boolean> entry : metadata.indexToEncoded().entrySet()) {
        Collection<String> names = metadata.indexToName().get(entry.getKey());
        for (String name : names) {
          variableToEncoded.put(name, entry.getValue());
        }
      }
      //解析参数
      return mutable.resolve(variables, variableToEncoded);
    }

    //BuildEncodedTemplateFromArgs
    protected RequestTemplate resolve(Object[] argv, RequestTemplate mutable,
                                      Map<String, Object> variables) {
      Object body = argv[metadata.bodyIndex()];
      checkArgument(body != null, "Body parameter %s was null", metadata.bodyIndex());
      try {
        //编码并设置RequestTemplate的body
        encoder.encode(body, metadata.bodyType(), mutable);
      } catch (EncodeException e) {
        throw e;
      } catch (RuntimeException e) {
        throw new EncodeException(e.getMessage(), e);
      }
      return super.resolve(argv, mutable, variables);
    }

    //BuildFormEncodedTemplateFromArgs
    protected RequestTemplate resolve(Object[] argv, RequestTemplate mutable,
                                      Map<String, Object> variables) {
      //构造form参数，为HashMap
      Map<String, Object> formVariables = new LinkedHashMap<String, Object>();
      for (Entry<String, Object> entry : variables.entrySet()) {
        if (metadata.formParams().contains(entry.getKey())) {
          formVariables.put(entry.getKey(), entry.getValue());
        }
      }
      try {
        //编码并设置RequestTemplate的body，
        encoder.encode(formVariables, Encoder.MAP_STRING_WILDCARD, mutable);
      } catch (EncodeException e) {
        throw e;
      } catch (RuntimeException e) {
        throw new EncodeException(e.getMessage(), e);
      }
      //调用父类的resolve
      return super.resolve(argv, mutable, variables);
    }

```

`RequestTemplate`解析参数的方法：
```java
  //RequestTemplate解析参数的方法
  RequestTemplate resolve(Map<String, ?> unencoded, Map<String, Boolean> alreadyEncoded) {
    //替换请求Query中的参数
    replaceQueryValues(unencoded, alreadyEncoded);
    Map<String, String> encoded = new LinkedHashMap<String, String>();
    for (Entry<String, ?> entry : unencoded.entrySet()) {
      final String key = entry.getKey();
      final Object objectValue = entry.getValue();
      String encodedValue = encodeValueIfNotEncoded(key, objectValue, alreadyEncoded);
      encoded.put(key, encodedValue);
    }
    //解析Url，替换path中的参数
    String resolvedUrl = expand(url.toString(), encoded).replace("+", "%20");
    if (decodeSlash) {
      resolvedUrl = resolvedUrl.replace("%2F", "/");
    }
    url = new StringBuilder(resolvedUrl);
    //解析http请求的header
    Map<String, Collection<String>> resolvedHeaders = new LinkedHashMap<String, Collection<String>>();
    for (String field : headers.keySet()) {
      Collection<String> resolvedValues = new ArrayList<String>();
      for (String value : valuesOrEmpty(headers, field)) {
        String resolved = expand(value, unencoded);
        resolvedValues.add(resolved);
      }
      resolvedHeaders.put(field, resolvedValues);
    }
    headers.clear();
    headers.putAll(resolvedHeaders);
    //处理bodyTemplate
    if (bodyTemplate != null) {
      body(urlDecode(expand(bodyTemplate, encoded)));
    }
    return this;
  }

  //工具方法，将含有{varName}的字符串模板中的变量名用变量值替换
  public static String expand(String template, Map<String, ?> variables) {
    if (checkNotNull(template, "template").length() < 3) {
      return template;
    }
    checkNotNull(variables, "variables for %s", template);

    boolean inVar = false;
    StringBuilder var = new StringBuilder();
    StringBuilder builder = new StringBuilder();
    for (char c : template.toCharArray()) {
      switch (c) {
        case '{':
          if (inVar) {
            // '{{' is an escape: write the brace and don't interpret as a variable
            builder.append("{");
            inVar = false;
            break;
          }
          inVar = true;
          break;
        case '}':
          if (!inVar) { // then write the brace literally
            builder.append('}');
            break;
          }
          inVar = false;
          String key = var.toString();
          Object value = variables.get(var.toString());
          if (value != null) {
            builder.append(value);
          } else {
            builder.append('{').append(key).append('}');
          }
          var = new StringBuilder();
          break;
        default:
          if (inVar) {
            var.append(c);
          } else {
            builder.append(c);
          }
      }
    }
    return builder.toString();
  }

```

## `RquestTemplate`转换`Request`

先来看看`Request`的结构，完整的http请求信息的定义：

```java
  private final String method;
  private final String url;
  private final Map<String, Collection<String>> headers;
  private final byte[] body;
  private final Charset charset;
```

`SynchronousMethodHandler`的`targetRequest`方法将`RequestTemplate`转换为`Request`
```java
  Request targetRequest(RequestTemplate template) {
    //先应用所用拦截器，拦截器是在Feign.Builder中传入的，拦截器可以修改RequestTemplate信息
    for (RequestInterceptor interceptor : requestInterceptors) {
      interceptor.apply(template);
    }
    //调用Target的apply方法，默认Target是HardCodedTarget
    return target.apply(new RequestTemplate(template));
  }
```
这块先应用所有拦截器，然后target的`apply`方法。拦截器和target都是扩展点，拦截器可以在构造好`RequestTemplate`后和发请求前修改请求信息，target默认使用`HardCodedTarget`直接发请求，feign还提供了`LoadBalancingTarget`，适配Ribbon来发请求，实现客户端的负载均衡。  

创建过程：
```java
    //HardCodedTarget的apply方法
    public Request apply(RequestTemplate input) {
      if (input.url().indexOf("http") != 0) {
        input.insert(0, url());
      }
      //调用RequestTemplate的request方法
      return input.request();
    }

    //RequestTemplate的request方法
  public Request request() {
    //安全拷贝所有header
    Map<String, Collection<String>> safeCopy = new LinkedHashMap<String, Collection<String>>();
    safeCopy.putAll(headers);
    //调用Request的create静态方法
    return Request.create(
        method, url + queryLine(),
        Collections.unmodifiableMap(safeCopy),
        body, charset
    );
  }

  //Request的create方法
    public static Request create(String method, String url, Map<String, Collection<String>> headers,
                                 byte[] body, Charset charset) {
      //new 对象
      return new Request(method, url, headers, body, charset);
    }
```
从代码上可以看到，`RequestTemplate`基本上直接转为`Request`，没有做什么逻辑操作。对比下`LoadBalancingTarget`：

```
  public Request apply(RequestTemplate input) {
    //选取一个Server，lb是Ribbon的AbstractLoadBalancer类
    Server currentServer = lb.chooseServer(null);
    //生成url
    String url = format("%s://%s%s", scheme, currentServer.getHostPort(), path);
    input.insert(0, url);
    try {
      //生成Request
      return input.request();
    } finally {
      lb.getLoadBalancerStats().incrementNumRequests(currentServer);
    }
  }
```
可以看到，非常简单的几行代码，只要修改请求的url就能实现客户端负载均衡。

## http请求发送

`SynchronousMethodHandler`中构造好`Request`后，直接调用client的`execute`方法发送请求：

```java
      response = client.execute(request, options);
```

client是一个`Client`接口，默认实现类是`Client.Default`，使用java api中的`HttpURLConnection`发送http请求。feign还实现了：
- `ApacheHttpClient`
- `OkHttpClient`
- `RibbonClient`
使用`RibbonClient`跟使用`LoadBalancingTarget`作用都是实现客户端负载均衡，`RibbonClient`实现稍微复杂些。

## 接口调用过程总结
我们再将接口调用过程捋一遍：

1、接口的动态代理`Proxy`调用接口方法会执行的`FeignInvocationHandler`
2、`FeignInvocationHandler`通过方法签名在属性`Map<Method, MethodHandler> dispatch`中找到`SynchronousMethodHandler`，调用`invoke`方法
3、`SynchronousMethodHandler`的`invoke`方法根据传入的方法参数，通过自身属性工厂对象`RequestTemplate.Factory`创建`RequestTemplate`，工厂里面会用根据需要进行`Encode`
4、`SynchronousMethodHandler`遍历自身属性`RequestInterceptor`列表，对`RequestTemplate`进行改造
4、`SynchronousMethodHandler`调用自身`Target`属性的`apply`方法，将`RequestTemplate`转换为`Request`对象
5、`SynchronousMethodHandler`调用自身`Client`的`execute`方法，传入`Request`对象
6、`Client`将`Request`转换为`http`请求，发送后将http响应转换为`Response`对象
7、`SynchronousMethodHandler`调用`Decoder`的方法对`Response`对象解码后返回
8、返回的对象最后返回到`Proxy`

时序图如下：
![](/images/feign/execute-sequence.png)

## feign扩展点总结
前文分析源代码时，已经提到了feign的扩展点，最后我们再将feign的主要扩展点进行总结一下：

- `Contract` 契约
`Contract`的作用是解析接口方法，生成Rest定义。feign默认使用自己的定义的注解，还提供了
  - `JAXRSContract` javax.ws.rs注解接口实现
`SpringContract`是spring cloud提供SpringMVC注解实现方式。

- `InvocationHandler` 动态代理handler
通过`InvocationHandlerFactory`注入到`Feign.Builder`中，feign提供了Hystrix的扩展，实现Hystrix接入

- `Encoder` 请求body编码器
feign已经提供扩展包含：
  - 默认编码器，只能处理String和byte[]
  - json编码器`GsonEncoder`、`JacksonEncoder`
  - XML编码器`JAXBEncoder`

- `Decoder` http响应解码器
最基本的有：
  - json解码器 `GsonDecoder`、`JacksonDecoder`
  - XML解码器 `JAXBDecoder`
  - Stream流解码器 `StreamDecoder`

- `Target` 请求转换器
feign提供的实现有：
  - `HardCodedTarget` 默认`Target`，不做任何处理。
  - `LoadBalancingTarget` 使用Ribbon进行客户端路由

- `Client` 发送http请求的客户端
feign提供的`Client`实现有：
  - `Client.Default` 默认实现，使用java api的`HttpClientConnection`发送http请求
  - `ApacheHttpClient` 使用apache的Http客户端发送请求
  - `OkHttpClient` 使用OKHttp客户端发送请求
  - `RibbonClient` 使用Ribbon进行客户端路由

- `RequestInterceptor` 请求拦截器
调用客户端发请求前，修改`RequestTemplate`，比如为所有请求添加Header就可以用拦截器实现。

- `Retryer` 重试策略
默认的策略是`Retryer.Default`，包含`3`个参数：间隔、最大间隔和重试次数，第一次失败重试前会sleep输入的间隔时间的，后面每次重试sleep时间是前一次的`1.5`倍，超过最大时间或者最大重试次数就失败
