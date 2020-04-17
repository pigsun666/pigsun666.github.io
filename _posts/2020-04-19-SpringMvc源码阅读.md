---
layout:     post
title:      SpingMvc源码
subtitle:   SpringMVC执行流程
date:       2020-4-19
author:     pigsun
header-img: img/post-bg-spring.jpeg
catalog: true
tags:
    - Spring
    - SpringMVC
---
# SpingMvc源码阅读
## 首先我们先了解一些专业名词
* DispatcherServlet springMVc的核心处理类 他继承了HttpServlet 所有请求都会进入这个类，然后通过不同的hanler处理请求。
* HandlerExecutionChain 请求执行器 
* HandlerMapping 通过不同的HandleMapping找不同注册方式的controller 因为实际上Controller有三种注册方法,2种处理方式，一种是viewControllerHandlerMapping 还有一种是BeanNameHandleMapping 
    * 通过@Controller标注当前类
    * 通过实现Controller接口(需要添加@Component注册到spring容器)
    * 通过实现HttpRequestHandler接口(需要添加@Component注册到spring容器)
* HandlerAdapter 适配器（ 根据请求使用特定的适配器）
* ViewResolvers 视图解析器


## 初始化springMvc
按照前文自己实现SpringMvc，servlet会去扫描所有加了Controller注解的类，我们来看下源码是不是这么做的。  


先看下继承结构   
    DispatcherServlet->FrameworkServlet->HttpServletBean->HttpServlet
父类都是抽象类 我们在HttpServletBean找到了init方法。
    public final void init() throws ServletException {
        if (this.logger.isDebugEnabled()) {
            this.logger.debug("Initializing servlet '" + this.getServletName() + "'");
        }

        PropertyValues pvs = new HttpServletBean.ServletConfigPropertyValues(this.getServletConfig(), this.requiredProperties);
        if (!pvs.isEmpty()) {
            try {
                BeanWrapper bw = PropertyAccessorFactory.forBeanPropertyAccess(this);
                ResourceLoader resourceLoader = new ServletContextResourceLoader(this.getServletContext());
                bw.registerCustomEditor(Resource.class, new ResourceEditor(resourceLoader, this.getEnvironment()));
                this.initBeanWrapper(bw);
                bw.setPropertyValues(pvs, true);
            } catch (BeansException var4) {
                if (this.logger.isErrorEnabled()) {
                    this.logger.error("Failed to set bean properties on servlet '" + this.getServletName() + "'", var4);
                }
    
                throw var4;
            }
        }
    
        this.initServletBean();
        if (this.logger.isDebugEnabled()) {
            this.logger.debug("Servlet '" + this.getServletName() + "' configured successfully");
        }
    
    }
我们就关注initServletBean这个方法，这个方法会调用到子类抽象类的initWebApplicationContext方法  
    
```
protected WebApplicationContext initWebApplicationContext() {
        WebApplicationContext rootContext = WebApplicationContextUtils.getWebApplicationContext(this.getServletContext());
        WebApplicationContext wac = null;
        if (this.webApplicationContext != null) {
            wac = this.webApplicationContext;
            if (wac instanceof ConfigurableWebApplicationContext) {
                ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext)wac;
                if (!cwac.isActive()) {
                    if (cwac.getParent() == null) {
                       //设置父容器
                        cwac.setParent(rootContext);
                    }

                   //这行代码完成了spring容器的初始化  
                   this.configureAndRefreshWebApplicationContext(cwac);
                }
            }
        }

        if (wac == null) {
            wac = this.findWebApplicationContext();
        }

        if (wac == null) {
            wac = this.createWebApplicationContext(rootContext);
        }

        if (!this.refreshEventReceived) {
            this.onRefresh(wac);
        }

        if (this.publishContext) {
            String attrName = this.getServletContextAttributeName();
            this.getServletContext().setAttribute(attrName, wac);
            if (this.logger.isDebugEnabled()) {
                this.logger.debug("Published WebApplicationContext of servlet '" + this.getServletName() + "' as ServletContext attribute with name [" + attrName + "]");
            }
        }

        return wac;
    }
```
里面的代码就不贴了，我们可以看到熟悉的wac.refresh() 这里SpringMvc和我们做法不一样，我们是直接扫描，他是交给spring来扫描。这里引入一个概念，spring子父容器。在早期的spring里面，SpringMvc和spring是两个容器，一般项目里都有两个配置文件，一个application.xml，还有一个是application-mvc.xml，为什么这么设计我也不知道，肯定是为了解决某些问题。
但是现在的spring还保留以前的一些代码逻辑，也有使用这些方法，但是实际上默认就是使用spring容器来管理。代码也不贴了，spring也有提供这么一个方法，可以开启子父容器setDetectHandlerMethodsInAncestorContexts。spring测试类HandlerMethodMappingTests有这方面的代码，有兴趣的同学可以仔细分析一下。

现在我们回到DispatcherServlet这个类initStrategies这个方法会初始化SpringMvc默认自带的工具集合，比如handleMapping，handleAdapter，ViewResolvers等等


```
protected void initStrategies(ApplicationContext context) {
        this.initMultipartResolver(context);
        this.initLocaleResolver(context);
        this.initThemeResolver(context);
        this.initHandlerMappings(context);
        this.initHandlerAdapters(context);
        this.initHandlerExceptionResolvers(context);
        this.initRequestToViewNameTranslator(context);
        this.initViewResolvers(context);
        this.initFlashMapManager(context);
    }
```

## 主入口doDispatch方法

### 1:根据Uri找到合适的handleMapping

所有被DispatcherServlet映射到的地址，都会进入该方法


```
//判断有没有包含文件
processedRequest = this.checkMultipart(request);
multipartRequestParsed = processedRequest != request;
//获取处理当前controller的handleMapping
mappedHandler = this.getHandler(processedRequest);
if (mappedHandler == null) {
    this.noHandlerFound(processedRequest, response);
    return;
}
```
我们关注一下mappedHandler=this.getHandler(processedRequest) 这个方法
按照前文我们判断当前请求是否由我们DispatcherServlet处理，我们是通过map简单的判断，SpringMvc肯定不会这样。他首先轮询内置的handleMapping,寻找处理当前请求的handleMapping。  
SpringMvc内置了2个HandleMapping，一个是常用的RequestMappingHandleMapping，另外一个则是BeanNameUrlHandleMapping,
第一个主要处理@controller配合@RequestMapping,
第二个主要是处理以beanName作为请求路径的方式。

如果没有找到能处理当前请求uri的handleMapping,会响应404。
如果找到了handleMapping会去寻找当前请求的适配器

### 2:寻找请求适配器HandleAdapter

```
HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());
```
里面也是轮询SpringMvc内置的请求适配器，请求适配器统一实现HandlerAdapter接口，接口提供了三个方法，supports是判断是否符合要求，返回Boolean值，handle方法就是具体执行方法了。

```
public interface HandlerAdapter {

	boolean supports(Object handler);

	@Nullable
	ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception;

	long getLastModified(HttpServletRequest request, Object handler);

}
```

这种设计是非常灵活的，开发者可以拓展springMvc，开发者只需要自定义一个类，实现当前接口，通过api添加进去，便可以拓展SpringMvc。我们学习源码不仅仅是为了更加了解框架的执行流程，更多的是应该学习这种编程思维，把流程抽象化的能力。

找到了合格的适配器后还会先看看是否快速重复访问，可选择返回缓存数据.

```
// Process last-modified header, if supported by the handler.
String method = request.getMethod();
boolean isGet = "GET".equals(method);
if (isGet || "HEAD".equals(method)) {
	long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
	if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
		return;
	}
}
```

然后会去调用一下所有的springMvc拦截器的preHandle方法，拦截器后文再提。

```
if (!mappedHandler.applyPreHandle(processedRequest, response)) {
		return;
}
```

### 3:请求适配器执行内部逻辑handle

如果上面操作都正常,则会用调用前面找到的请求适配器的handle方法。
我们当前分析RequestMappingHandleMapping这个适配器的执行流程，执行到invokeHandlerMethod方法，然后进行一部分设置，

```
invocableMethod.invokeAndHandle(webRequest, mavContainer, new Object[0]);
```
然后会调用到

```
Object returnValue = this.invokeForRequest(webRequest, mavContainer, providedArgs);
```
#### 1:参数推断和赋值和方法反射执行

高能来了

```java
public Object invokeForRequest(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer, Object... providedArgs) throws Exception {
        Object[] args = this.getMethodArgumentValues(request, mavContainer, providedArgs);
        if (this.logger.isTraceEnabled()) {
            this.logger.trace("Invoking '" + ClassUtils.getQualifiedMethodName(this.getMethod(), this.getBeanType()) + "' with arguments " + Arrays.toString(args));
        }

        Object returnValue = this.doInvoke(args);
        if (this.logger.isTraceEnabled()) {
            this.logger.trace("Method [" + ClassUtils.getQualifiedMethodName(this.getMethod(), this.getBeanType()) + "] returned [" + returnValue + "]");
        }

        return returnValue;
    }
```
这个代码会不会很熟悉，和我们自己手写springmvc很像，我们如果这样理解就很简单了，this.getMethodArgumentValues(request, mavContainer, providedArgs)这个方法就是处理请求参数，从request取参数值，然后创建出请求参数对象，

```
Object returnValue = this.doInvoke(args);
```
这个this实际上就是springMvc封装了method的对象，实际上就是反射调用，里面有当前controller的bean对象，实际意义就是method.invoke(this.bean,args)

然后返回值returnValue 返回出去   后面会处理。
我们看看getMethodArgumentValues方法 是怎么处理请求参数的。

```java
private Object[] getMethodArgumentValues(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer, Object... providedArgs) throws Exception {
        MethodParameter[] parameters = this.getMethodParameters();
        Object[] args = new Object[parameters.length];

        for(int i = 0; i < parameters.length; ++i) {
            MethodParameter parameter = parameters[i];
            parameter.initParameterNameDiscovery(this.parameterNameDiscoverer);
            args[i] = this.resolveProvidedArgument(parameter, providedArgs);
            if (args[i] == null) {
                if (this.argumentResolvers.supportsParameter(parameter)) {
                    try {
                        args[i] = this.argumentResolvers.resolveArgument(parameter, mavContainer, request, this.dataBinderFactory);
                    } catch (Exception var9) {
                        if (this.logger.isDebugEnabled()) {
                            this.logger.debug(this.getArgumentResolutionErrorMessage("Failed to resolve", i), var9);
                        }

                        throw var9;
                    }
                } else if (args[i] == null) {
                    throw new IllegalStateException("Could not resolve method parameter at index " + parameter.getParameterIndex() + " in " + parameter.getExecutable().toGenericString() + ": " + this.getArgumentResolutionErrorMessage("No suitable resolver for", i));
                }
            }
        }

        return args;
    }
```

是不是基本一模一样，但是springMvc比我们优秀很多，因为我们自己写的springMvc  参数类型判断都是通过if/else，这样肯定是缺少拓展能力，是属于硬编码的，不易于维护和拓展，看看springMvc是怎么处理的。

提供了一个请求参数处理的接口HandlerMethodArgumentResolver
这个接口提供2个方法

```java
public interface HandlerMethodArgumentResolver {
    //这里判断是否符合转换类型规则
    boolean supportsParameter(MethodParameter var1);

    //具体的转换逻辑
    @Nullable
    Object resolveArgument(MethodParameter var1, @Nullable ModelAndViewContainer var2, NativeWebRequest var3, @Nullable WebDataBinderFactory var4) throws Exception;
}
```

SringMVc内置了很多转换器![微信图片_20200414115951](C:\Users\36928\Desktop\微信图片_20200414115951.png)

会根据不同的类型，使用不同的转换器进行参数填充，程序员也可以自己拓展，非常灵活。

这里补充一个拓展的知识点，如果需要拓展springMvc ，需要编写一个类，通常是实现WebMvcConfigurer   然后实现具体方法，比如

```
default void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
    }
```

参数会传递一个list给你，如果我要新增的是fastJson消息转换器，通常是new 出来fastJson消息转换器，然后添加到list里面就可以了。但是这个List我们不能保证执行的顺序，因为传参过来的List和springMvc实际使用的List并不是同一个List，SpringMvc在内部有合并List的操作，所以如果你一定要修改执行的顺序，需要修改继承WebMvcConfigurationSupport，然后注入方法

```java
@Autowired//Spring mvc 环境初始化完之后 去加载这个方法
public void initArgumentResolvers(   RequestMappingHandlerAdapter requestMappingHandlerAdapter){
	//springMvc内部的List对象
    List<HandlerMethodArgumentResolver> argumentResolvers = new ArrayList<>(requestMappingHandlerAdapter.getArgumentResolvers());
    List<HandlerMethodArgumentResolver> customResolvers = requestMappingHandlerAdapter.getCustomArgumentResolvers();
    argumentResolvers.removeAll(customResolvers);
    argumentResolvers.addAll(0, customResolvers);
    requestMappingHandlerAdapter.setArgumentResolvers(argumentResolvers);
}
```
然后我们回到invokeAndHandle方法，执行完invokeForRequest方法之后会返回一个object对象，接下来就需要进行返回值解析，

#### 2:返回值解析构造ModelAndView对象

~~~~java
this.returnValueHandlers.handleReturnValue(returnValue, this.getReturnValueType(returnValue), mavContainer, webRequest);
~~~~

```java
//视图裁决
public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType, ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {
    //通过这个对象来进行判断接下来视图怎么做、
    HandlerMethodReturnValueHandler handler = this.selectHandler(returnValue, returnType);
    if (handler == null) {
        throw new IllegalArgumentException("Unknown return value type: " + returnType.getParameterType().getName());
    } else {
        handler.handleReturnValue(returnValue, returnType, mavContainer, webRequest);
    }
}
```

继续往selectHandler方法里面看

```java
private HandlerMethodReturnValueHandler selectHandler(@Nullable Object value, MethodParameter returnType) {
    //通过名字猜测是处理异步返回值的
    boolean isAsyncValue = this.isAsyncReturnValue(value, returnType);
    Iterator var4 = this.returnValueHandlers.iterator();

    HandlerMethodReturnValueHandler handler;
    do {
        do {
            if (!var4.hasNext()) {
                return null;
            }

            handler = (HandlerMethodReturnValueHandler)var4.next();
        } while(isAsyncValue && !(handler instanceof AsyncHandlerMethodReturnValueHandler));
    } while(!handler.supportsReturnType(returnType));

    return handler;
}
```

这里也是有返回值处理器   同样也是提供了对外拓展的方式。我们看看内置了多少种，



![image-20200414145046226](C:\Users\36928\AppData\Roaming\Typora\typora-user-images\image-20200414145046226.png)

方式和上面一样，一个是判断是否支持类型，另外一个是具体操作

```
public interface HandlerMethodReturnValueHandler {
    boolean supportsReturnType(MethodParameter var1);

    void handleReturnValue(@Nullable Object var1, MethodParameter var2, ModelAndViewContainer var3, NativeWebRequest var4) throws Exception;
}
```

执行完后 会构建一个mavContainer对象，里面会保存视图和模型各种信息，之后调用getModelAndView再去构建ModelAndView对象，会返回ModelAndView对象。当然当你返回值并不是ModelAndView或者加了@ResponseBody，又或者其他不知道的对象，那么ModelAndView也可以为空，回到doDIspatch方法，开始根据ModelAndView渲染视图。

### 4：渲染视图

```java
//如果有返回modelAndView，并且没有设置View  会设置默认的view
this.applyDefaultViewName(processedRequest, mv);
//执行所有拦截器的后置方法
mappedHandler.applyPostHandle(processedRequest, response, mv);
//处理返回值，渲染视图
this.processDispatchResult(processedRequest, response, mappedHandler, mv, (Exception)dispatchException);
```

着重关注processDispatchResult这个方法

```java
private void processDispatchResult(HttpServletRequest request, HttpServletResponse response, @Nullable HandlerExecutionChain mappedHandler, @Nullable ModelAndView mv, @Nullable Exception exception) throws Exception {
    boolean errorView = false;
    //首先判断是否有异常，如果有异常会响应异常视图
    if (exception != null) {
        if (exception instanceof ModelAndViewDefiningException) {
            this.logger.debug("ModelAndViewDefiningException encountered", exception);
            mv = ((ModelAndViewDefiningException)exception).getModelAndView();
        } else {
            Object handler = mappedHandler != null ? mappedHandler.getHandler() : null;
            mv = this.processHandlerException(request, response, handler, exception);
            errorView = mv != null;
        }
    }
	//如果返回值没有被清除，并且mv有值
    if (mv != null && !mv.wasCleared()) {
        //渲染视图
        this.render(mv, request, response);
        if (errorView) {
            WebUtils.clearErrorRequestAttributes(request);
        }
    } else if (this.logger.isDebugEnabled()) {
        this.logger.debug("Null ModelAndView returned to DispatcherServlet with name '" + this.getServletName() + "': assuming HandlerAdapter completed request handling");
    }

    if (!WebAsyncUtils.getAsyncManager(request).isConcurrentHandlingStarted()) {
        if (mappedHandler != null) {
            mappedHandler.triggerAfterCompletion(request, response, (Exception)null);
        }

    }
}
```

如果不需要渲染视图（如果是重定向 || 响应视图的话） 就会返回null

mavContainer.isRequestHandled() 判断是否需要重定向或响应

同时会把model里面的参数放到request.setAttribute（说明model的作用域是request作用域）

如果外部传入的modelAndView对象不为空，则会根据viewName进行视图渲染

```java
protected View resolveViewName(String viewName, @Nullable Map<String, Object> model, Locale locale, HttpServletRequest request) throws Exception {
    if (this.viewResolvers != null) {
        Iterator var5 = this.viewResolvers.iterator();

        while(var5.hasNext()) {
            ViewResolver viewResolver = (ViewResolver)var5.next();
            View view = viewResolver.resolveViewName(viewName, locale);
            if (view != null) {
                return view;
            }
        }
    }

    return null;
}
```

这里又是适配器设计模式，视图渲染器，

![image-20200414155137660](C:\Users\36928\AppData\Roaming\Typora\typora-user-images\image-20200414155137660.png)

这里只有1个视图渲染器，

```java
protected void renderMergedOutputModel(Map<String, Object> model, HttpServletRequest request, HttpServletResponse response) throws Exception {
    this.exposeModelAsRequestAttributes(model, request);
    this.exposeHelpers(request);
    String dispatcherPath = this.prepareForRendering(request, response);
    RequestDispatcher rd = this.getRequestDispatcher(request, dispatcherPath);
    if (rd == null) {
        throw new ServletException("Could not get RequestDispatcher for [" + this.getUrl() + "]: Check that the corresponding file exists within your web application archive!");
    } else {
        if (this.useInclude(request, response)) {
            response.setContentType(this.getContentType());
            if (this.logger.isDebugEnabled()) {
                this.logger.debug("Including resource [" + this.getUrl() + "] in InternalResourceView '" + this.getBeanName() + "'");
            }
			//转发
            rd.include(request, response);
        } else {
            if (this.logger.isDebugEnabled()) {
                this.logger.debug("Forwarding to resource [" + this.getUrl() + "] in InternalResourceView '" + this.getBeanName() + "'");
            }
			//重定向
            rd.forward(request, response);
        }

    }
}
```

include方法和forward方法已经是Servlet的接口，这里会完成视图转发或者重定向操作.