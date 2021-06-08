---
layout:     post
title:      "SpringBoot(二) starter与servlet容器"
date:       2018-10-10
author:     "ZhouJ000"
header-img: "img/in-post/2018/post-bg-2018-headbg.jpg"
catalog: true
tags:
    - springBoot
--- 

<font id="last-updated">最后更新于：2018-10-10</font>

[SpringBoot(一) 启动与自动配置](https://zhouj000.github.io/2018/10/05/springboot-1/)  
[SpringBoot(二) starter与servlet容器](https://zhouj000.github.io/2018/10/10/springboot-2/)  
[SpringBoot(三) Environment](https://zhouj000.github.io/2019/10/08/springboot-3/)  
[SpringBoot(四) 集成apollo遇到的事儿](https://zhouj000.github.io/2019/10/16/springboot-4/)  
[SpringBoot(五) 健康检查(上)](https://zhouj000.github.io/2020/01/28/springboot-5/)  
[SpringBoot(六) 健康检查(下)](https://zhouj000.github.io/2020/02/19/springboot-6/)  




查看spring boot目录，在`spring-boot\spring-boot-project`下可以看出分类
```
pom.xml                              spring-boot-cli/           spring-boot-properties-migrator/
spring-boot/                         spring-boot-dependencies/  spring-boot-starters/
spring-boot-actuator/                spring-boot-devtools/      spring-boot-test/
spring-boot-actuator-autoconfigure/  spring-boot-docs/          spring-boot-test-autoconfigure/
spring-boot-autoconfigure/           spring-boot-parent/        spring-boot-tools/
```

# boot-starter

[Spring Boot application starters](https://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#using-boot-starter)

starter主要的作用是做了包依赖管理，查看spring-boot-starters文件夹下，举例spring-boot-starter-web，其实只有一个pom文件:
```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starters</artifactId>
		<version>${revision}</version>
	</parent>
	<artifactId>spring-boot-starter-web</artifactId>
	<name>Spring Boot Web Starter</name>
	<description>Starter for building web, including RESTful, applications using Spring
		MVC. Uses Tomcat as the default embedded container</description>
	<properties>
		<main.basedir>${basedir}/../../..</main.basedir>
	</properties>
	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-json</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-tomcat</artifactId>
		</dependency>
		<dependency>
			<groupId>org.hibernate.validator</groupId>
			<artifactId>hibernate-validator</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-web</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-webmvc</artifactId>
		</dependency>
	</dependencies>
</project>
```

## 实现自己的starter

@EnableAutoConfiguration里通过@Import导入了AutoConfigurationImportSelector，在selectImports方法中通过SpringFactoriesLoader读取META-INF/spring.factories的配置。**@Enable开头的Annotation可以借助@Import的支持，收集和注册特定场景相关的bean定义**。将其中EnableAutoConfiguration对应的配置项通过反射实例化为对应的标注了@Configuration的JavaConfig形式的IOC容器配置类，然后汇总为一个并加载到IOC容器。ConfigurationClassPostProcessor由于实现了BeanDefinitionRegistryPostProcessor，会在Spring容器初始化时被调用postProcessBeanDefinitionRegistry方法，其中会创建ConfigurationClassParser类解析configCandidates，实际是用ConditionEvaluator来判断@Conditional注解

由上一篇知道，自动配置通过META-INF/spring.factories配置与@Conditional注解来生成java配置，通过@ConfigurationProperties注解的配置文件加载默认配置

1 . 创建一个简单的maven项目  
2 . pom加入依赖与添加编码配置
```
<properties>
	<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
</properties>

<dependencies>
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-autoconfigure</artifactId>
		<version>2.0.5.RELEASE</version>
	</dependency>
</dependencies>
```
3 . 在resources下新建META-INF文件夹，里面创建spring.factories文件  
4 . 加入配置
```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.zj.hello-starter.HelloAutoConfiguration
```
5 . 创建HelloAutoConfiguration文件
```java
// 自动配置类
@Configuration
/*
 *  根据HelloProperties提供的参数，并通过@ConditionalOnClass判断Hello这个类
 *  在类路径中是否存在，且当容器中没有这个Bean的情况下自动配置这个Bean
 */
@EnableConfigurationProperties(HelloProperties.class)
@ConditionalOnClass(Hello.class)
@ConditionalOnProperty(prefix = "hello", value = "enabled", matchIfMissing = true)
public class HelloAutoConfiguration {
	@Autowired
	private HelloProperties helloProperties;
	
	@Bean
	@ConditionalOnMissingBean(Hello.class)
	public Hello hello() {
		Hello hello = new Hello();
		hello.setMsg(helloProperties.getMsg());
		return hello;
	}
}
```
6 . 创建HelloProperties配置
```java
// 属性配置
@ConfigurationProperties(prefix = "hello")
public class HelloProperties {
	// 默认值
	private static final String MSG = "world";

	private String msg = MSG;

	public String getMsg() {
		return msg;
	}
	public void setMsg(String msg) {
		this.msg = msg;
	}
}
```
7 . 创建Hello类
```java
// 判断依据类
public class Hello {
	private String msg;

	public String sayHello() {
		return "Hello " + msg;
	}

	public String getMsg() {
		return msg;
	}
	public void setMsg(String msg) {
		this.msg = msg;
	}
}
```
8 . 执行'mvn clean install'编译打包到本地mvn_lib  
9 . 在自己的应用中加入依赖
```
<dependency>
	<groupId>com.zj</groupId>
	<artifactId>hello-starter</artifactId>
	<version>1.0-SNAPSHOT</version>
</dependency>
```
10 . 可以在application.yml中添加配置`hello.msg: spring boot starter`  
11 . 写个测试类运行下
```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class Test6 {

    @Autowired
    public Hello hello;

    @Test
    public void test() {
        System.out.println(hello.sayHello());
    }
}

// 打印 Hello spring boot starter
```

# 自带servlet容器

从上面spring-boot-starter-web的pom文件中可以看到，引入了spring-boot-starter-tomcat依赖，所以启动web时就有了tomcat容器。除了tomcat，还可以通过exclusion子标签排除tomcat，加入spring-boot-starter-jetty来使用jetty容器

通过spring.factories文件可以发现了web.servlet包下的自动配置类
```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
...
org.springframework.boot.autoconfigure.web.servlet.DispatcherServletAutoConfiguration,\
org.springframework.boot.autoconfigure.web.servlet.ServletWebServerFactoryAutoConfiguration,\
org.springframework.boot.autoconfigure.web.servlet.error.ErrorMvcAutoConfiguration,\
org.springframework.boot.autoconfigure.web.servlet.HttpEncodingAutoConfiguration,\
org.springframework.boot.autoconfigure.web.servlet.MultipartAutoConfiguration,\
org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration,\
...
```
查看ServletWebServerFactoryAutoConfiguration类
```java
@Configuration
@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE)
@ConditionalOnClass(ServletRequest.class)
@ConditionalOnWebApplication(type = Type.SERVLET)
@EnableConfigurationProperties(ServerProperties.class)
@Import({ ServletWebServerFactoryAutoConfiguration.BeanPostProcessorsRegistrar.class,
		ServletWebServerFactoryConfiguration.EmbeddedTomcat.class,
		ServletWebServerFactoryConfiguration.EmbeddedJetty.class,
		ServletWebServerFactoryConfiguration.EmbeddedUndertow.class })
public class ServletWebServerFactoryAutoConfiguration { // ... }


@ConfigurationProperties(prefix = "server", ignoreUnknownFields = true)
public class ServerProperties {}
```
发现import了ServletWebServerFactoryConfiguration.EmbeddedTomcat，继续点进去查看，是一个内部类，很简单，创建一个tomcat工厂
```java
@Configuration
@ConditionalOnClass({ Servlet.class, Tomcat.class, UpgradeProtocol.class })
@ConditionalOnMissingBean(value = ServletWebServerFactory.class, search = SearchStrategy.CURRENT)
public static class EmbeddedTomcat {

	@Bean
	public TomcatServletWebServerFactory tomcatServletWebServerFactory() {
		return new TomcatServletWebServerFactory();
	}
}
```
![tomcatServletWebServerFactory](/img/in-post/2018/10/tomcatServletWebServerFactory.png)
另一方面，import也导入了ServletWebServerFactoryAutoConfiguration.BeanPostProcessorsRegistrar，它是在ConfigurationClassPostProcessor.postProcessBeanFactory方法中会调用processConfigBeanDefinitions方法，然后调用loadBeanDefinitionsForConfigurationClass方法到loadBeanDefinitionsFromRegistrars方法里，其中之一是通过BeanPostProcessorsRegistrar去注册几个bean
```java
public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata,
		BeanDefinitionRegistry registry) {
	if (this.beanFactory == null) {
		return;
	}
	// ----> 
	registerSyntheticBeanIfMissing(registry,
			"webServerFactoryCustomizerBeanPostProcessor",
			WebServerFactoryCustomizerBeanPostProcessor.class);
	registerSyntheticBeanIfMissing(registry,
			"errorPageRegistrarBeanPostProcessor",
			ErrorPageRegistrarBeanPostProcessor.class);
}
```
其中WebServerFactoryCustomizerBeanPostProcessor类实现了BeanPostProcessor接口，在实例化ServletWebServerFactoryConfiguration初始化时候，会调用applyBeanPostProcessorsBeforeInitialization方法，其中获取BeanPostProcessors执行其postProcessBeforeInitialization方法，会进行一些设值(通过ServerProperties)
```java
public Object postProcessBeforeInitialization(Object bean, String beanName)
		throws BeansException {
	if (bean instanceof WebServerFactory) {
		postProcessBeforeInitialization((WebServerFactory) bean);
	}
	return bean;
}
	
private void postProcessBeforeInitialization(WebServerFactory webServerFactory) {
	LambdaSafe
		.callbacks(WebServerFactoryCustomizer.class, getCustomizers(),
				webServerFactory)
		.withLogger(WebServerFactoryCustomizerBeanPostProcessor.class)
		// customize会为factory设值(TomcatWebServerFactoryCustomizer、TomcatServletWebServerFactoryCustomizer)
		.invoke((customizer) -> customizer.customize(webServerFactory));
}
```

## 启动

那么tomcat容器是在什么时候被创建且启动的呢？  
因为webApplicationType是SERVLET，所以创建了AnnotationConfigServletWebServerApplicationContext，在进行refresh方法中，有一个onRefresh方法，其注释是Initialize other special beans in specific context subclasses。正好AnnotationConfigServletWebServerApplicationContext继承的ServletWebServerApplicationContext实现了这个方法
```java
protected void onRefresh() {
	super.onRefresh();
	try {
		// ----> 创建webServer
		createWebServer();
	}
	catch (Throwable ex) {
		throw new ApplicationContextException("Unable to start web server", ex);
	}
}
```

1 . 创建webServer
```java
private void createWebServer() {
	WebServer webServer = this.webServer;
	ServletContext servletContext = getServletContext();
	if (webServer == null && servletContext == null) {
		// ----> 2. 获取WebServer工厂
		ServletWebServerFactory factory = getWebServerFactory();
		// ----> 3. 获取webServer
		this.webServer = factory.getWebServer(getSelfInitializer());
	}
	else if (servletContext != null) {
		try {
			getSelfInitializer().onStartup(servletContext);
		}
		catch (ServletException ex) {
			throw new ApplicationContextException("Cannot initialize servlet context",
					ex);
		}
	}
	// 初始化PropertySources
	initPropertySources();
}
```

2 . 获取WebServer工厂
```java
protected ServletWebServerFactory getWebServerFactory() {
	// Use bean names so that we don't consider the hierarchy
	// TomcatServletWebServerFactory正是继承了ServletWebServerFactory
	String[] beanNames = getBeanFactory()
			.getBeanNamesForType(ServletWebServerFactory.class);
	if (beanNames.length == 0) {
		throw new ApplicationContextException(
				"Unable to start ServletWebServerApplicationContext due to missing "
						+ "ServletWebServerFactory bean.");
	}
	if (beanNames.length > 1) {
		throw new ApplicationContextException(
				"Unable to start ServletWebServerApplicationContext due to multiple "
						+ "ServletWebServerFactory beans : "
						+ StringUtils.arrayToCommaDelimitedString(beanNames));
	}
	return getBeanFactory().getBean(beanNames[0], ServletWebServerFactory.class);
}
```

3 . 获取webServer
```java
public WebServer getWebServer(ServletContextInitializer... initializers) {
	// 创建tomcat
	Tomcat tomcat = new Tomcat();
	File baseDir = (this.baseDirectory != null ? this.baseDirectory
			: createTempDir("tomcat"));
	tomcat.setBaseDir(baseDir.getAbsolutePath());
	// org.apache.coyote.http11.Http11NioProtocol
	Connector connector = new Connector(this.protocol);
	// 设置connector
	tomcat.getService().addConnector(connector);
	// 设置端口等信息
	customizeConnector(connector);
	tomcat.setConnector(connector);
	tomcat.getHost().setAutoDeploy(false);
	configureEngine(tomcat.getEngine());
	for (Connector additionalConnector : this.additionalTomcatConnectors) {
		tomcat.getService().addConnector(additionalConnector);
	}
	// ----> 4. 准备上下文
	prepareContext(tomcat.getHost(), initializers);
	// ----> 5. 创建TomcatWebServer封装tomcat，启动
	return getTomcatWebServer(tomcat);
}


// StandardService
public void addConnector(Connector connector) {
	synchronized (connectorsLock) {
		connector.setService(this);
		Connector results[] = new Connector[connectors.length + 1];
		System.arraycopy(connectors, 0, results, 0, connectors.length);
		results[connectors.length] = connector;
		connectors = results;
		if (getState().isAvailable()) {
			try {
				connector.start();
			} catch (LifecycleException e) {
				log.error(sm.getString(
						"standardService.connector.startFailed",
						connector), e);
			}
		}
		// Report this property change to interested listeners
		support.firePropertyChange("connector", null, connector);
	}
}

protected void customizeConnector(Connector connector) {
	int port = (getPort() >= 0 ? getPort() : 0);
	connector.setPort(port);
	if (StringUtils.hasText(this.getServerHeader())) {
		connector.setAttribute("server", this.getServerHeader());
	}
	if (connector.getProtocolHandler() instanceof AbstractProtocol) {
		customizeProtocol((AbstractProtocol<?>) connector.getProtocolHandler());
	}
	if (getUriEncoding() != null) {
		connector.setURIEncoding(getUriEncoding().name());
	}
	// Don't bind to the socket prematurely if ApplicationContext is slow to start
	connector.setProperty("bindOnInit", "false");
	if (getSsl() != null && getSsl().isEnabled()) {
		customizeSsl(connector);
	}
	TomcatConnectorCustomizer compression = new CompressionConnectorCustomizer(
			getCompression());
	compression.customize(connector);
	for (TomcatConnectorCustomizer customizer : this.tomcatConnectorCustomizers) {
		customizer.customize(connector);
	}
}
```

4 . 准备上下文
```java
protected void prepareContext(Host host, ServletContextInitializer[] initializers) {
	File documentRoot = getValidDocumentRoot();
	// 创建TomcatEmbeddedContext
	TomcatEmbeddedContext context = new TomcatEmbeddedContext();
	if (documentRoot != null) {
		context.setResources(new LoaderHidingResourceRoot(context));
	}
	context.setName(getContextPath());
	context.setDisplayName(getDisplayName());
	context.setPath(getContextPath());
	File docBase = (documentRoot != null ? documentRoot
			: createTempDir("tomcat-docbase"));
	context.setDocBase(docBase.getAbsolutePath());
	context.addLifecycleListener(new FixContextListener());
	context.setParentClassLoader(
			this.resourceLoader != null ? this.resourceLoader.getClassLoader()
					: ClassUtils.getDefaultClassLoader());
	resetDefaultLocaleMapping(context);
	addLocaleMappings(context);
	context.setUseRelativeRedirects(false);
	configureTldSkipPatterns(context);
	WebappLoader loader = new WebappLoader(context.getParentClassLoader());
	loader.setLoaderClass(TomcatEmbeddedWebappClassLoader.class.getName());
	loader.setDelegate(true);
	context.setLoader(loader);
	if (isRegisterDefaultServlet()) {
		addDefaultServlet(context);
	}
	if (shouldRegisterJspServlet()) {
		addJspServlet(context);
		addJasperInitializer(context);
	}
	context.addLifecycleListener(new StaticResourceConfigurer(context));
	ServletContextInitializer[] initializersToUse = mergeInitializers(initializers);
	host.addChild(context);
	// 设置上下文，会创建一个TomcatStarter，并加入到context的initializers列表中
	configureContext(context, initializersToUse);
	// 留给子类扩展，空方法
	postProcessContext(context);
}

protected void configureContext(Context context,
		ServletContextInitializer[] initializers) {
	TomcatStarter starter = new TomcatStarter(initializers);
	if (context instanceof TomcatEmbeddedContext) {
		// Should be true
		((TomcatEmbeddedContext) context).setStarter(starter);
	}
	context.addServletContainerInitializer(starter, NO_CLASSES);
	for (LifecycleListener lifecycleListener : this.contextLifecycleListeners) {
		context.addLifecycleListener(lifecycleListener);
	}
	for (Valve valve : this.contextValves) {
		context.getPipeline().addValve(valve);
	}
	for (ErrorPage errorPage : getErrorPages()) {
		new TomcatErrorPage(errorPage).addToContext(context);
	}
	for (MimeMappings.Mapping mapping : getMimeMappings()) {
		context.addMimeMapping(mapping.getExtension(), mapping.getMimeType());
	}
	configureSession(context);
	for (TomcatContextCustomizer customizer : this.tomcatContextCustomizers) {
		customizer.customize(context);
	}
}
```

5 . 封装并启动
```java
public TomcatWebServer(Tomcat tomcat, boolean autoStart) {
	Assert.notNull(tomcat, "Tomcat Server must not be null");
	this.tomcat = tomcat;
	this.autoStart = autoStart;
	initialize();
}


private void initialize() throws WebServerException {
	TomcatWebServer.logger
			.info("Tomcat initialized with port(s): " + getPortsDescription(false));
	synchronized (this.monitor) {
		try {
			addInstanceIdToEngineName();

			Context context = findContext();
			context.addLifecycleListener((event) -> {
				if (context.equals(event.getSource())
						&& Lifecycle.START_EVENT.equals(event.getType())) {
					// Remove service connectors so that protocol binding doesn't
					// happen when the service is started.
					removeServiceConnectors();
				}
			});

			// Start the server to trigger initialization listeners
			// 启动tomcat
			this.tomcat.start();

			// We can re-throw failure exception directly in the main thread
			rethrowDeferredStartupExceptions();

			try {
				ContextBindings.bindClassLoader(context, context.getNamingToken(),
						getClass().getClassLoader());
			}
			catch (NamingException ex) {
				// Naming is not enabled. Continue
			}

			// Unlike Jetty, all Tomcat threads are daemon threads. We create a
			// blocking non-daemon to stop immediate shutdown
			// 创建一个阻塞的非守护线程阻止立即关闭
			startDaemonAwaitThread();
		}
		catch (Exception ex) {
			stopSilently();
			throw new WebServerException("Unable to start embedded Tomcat", ex);
		}
	}
}
```

### 疑问

在设置上下文时，新建了一个TomcatStarter，并放入上下文的initializers中，那是什么时候被调用的？这里的初始化又是做什么的？

在创建TomcatWebServer时，执行initialize方法，期间会启动tomcat`this.tomcat.start()`
```java
// Tomcat
public void start() throws LifecycleException {
	getServer();
	getConnector();
	// ---->
	server.start();
}

// LifecycleBase
public final synchronized void start() throws LifecycleException {
	if (LifecycleState.STARTING_PREP.equals(state) || LifecycleState.STARTING.equals(state) ||
			LifecycleState.STARTED.equals(state)) {

		if (log.isDebugEnabled()) {
			Exception e = new LifecycleException();
			log.debug(sm.getString("lifecycleBase.alreadyStarted", toString()), e);
		} else if (log.isInfoEnabled()) {
			log.info(sm.getString("lifecycleBase.alreadyStarted", toString()));
		}
		return;
	}

	if (state.equals(LifecycleState.NEW)) {
		init();
	} else if (state.equals(LifecycleState.FAILED)) {
		stop();
	} else if (!state.equals(LifecycleState.INITIALIZED) &&
			!state.equals(LifecycleState.STOPPED)) {
		invalidTransition(Lifecycle.BEFORE_START_EVENT);
	}

	try {
		setStateInternal(LifecycleState.STARTING_PREP, null, false);
		// ----> 启动组件，并且实现所需的要求，中间一步就是执行initializers里ServletContainerInitializer的onStartup方法
		startInternal();
		if (state.equals(LifecycleState.FAILED)) {
			// This is a 'controlled' failure. The component put itself into the
			// FAILED state so call stop() to complete the clean-up.
			stop();
		} else if (!state.equals(LifecycleState.STARTING)) {
			// Shouldn't be necessary but acts as a check that sub-classes are
			// doing what they are supposed to.
			invalidTransition(Lifecycle.AFTER_START_EVENT);
		} else {
			setStateInternal(LifecycleState.STARTED, null, false);
		}
	} catch (Throwable t) {
		// This is an 'uncontrolled' failure so put the component into the
		// FAILED state and throw an exception.
		ExceptionUtils.handleThrowable(t);
		setStateInternal(LifecycleState.FAILED, null, false);
		throw new LifecycleException(sm.getString("lifecycleBase.startFail", toString()), t);
	}
}
```

TomcatStarter正是继承ServletContainerInitializer，查看其onStartup方法
```java
public void onStartup(Set<Class<?>> classes, ServletContext servletContext)
		throws ServletException {
	try {
		for (ServletContextInitializer initializer : this.initializers) {
			initializer.onStartup(servletContext);
		}
	}
	catch (Exception ex) {
		this.startUpException = ex;
		// Prevent Tomcat from logging and re-throwing when we know we can
		// deal with it in the main thread, but log for information here.
		if (logger.isErrorEnabled()) {
			logger.error("Error starting Tomcat context. Exception: "
					+ ex.getClass().getName() + ". Message: " + ex.getMessage());
		}
	}
}
```
分别调用一些方法，具体的看不太懂
```java
// 1. AbstractServletWebServerFactory
// 工具方法，希望将指定的ServletContextInitializer参数与此实例中定义的参数组合的子类可以使用该方法
protected final ServletContextInitializer[] mergeInitializers(
		ServletContextInitializer... initializers) {
	List<ServletContextInitializer> mergedInitializers = new ArrayList<>();
	mergedInitializers.add((servletContext) -> this.initParameters
			.forEach(servletContext::setInitParameter));
	mergedInitializers.add(new SessionConfiguringInitializer(this.session));
	mergedInitializers.addAll(Arrays.asList(initializers));
	mergedInitializers.addAll(this.initializers);
	return mergedInitializers.toArray(new ServletContextInitializer[0]);
}


// 2. AbstractServletWebServerFactory
public void onStartup(ServletContext servletContext) throws ServletException {
	if (this.session.getTrackingModes() != null) {
		servletContext
				.setSessionTrackingModes(unwrap(this.session.getTrackingModes()));
	}
	// 设置cookie
	configureSessionCookie(servletContext.getSessionCookieConfig());
}


// 3. ServletWebServerApplicationContext
private void selfInitialize(ServletContext servletContext) throws ServletException {
	prepareWebApplicationContext(servletContext);
	ConfigurableListableBeanFactory beanFactory = getBeanFactory();
	ExistingWebApplicationScopes existingScopes = new ExistingWebApplicationScopes(
			beanFactory);
	// 注册一些web特定的scope("request", "session", "globalSession", "application")
	WebApplicationContextUtils.registerWebApplicationScopes(beanFactory,
			getServletContext());
	existingScopes.restore();
	WebApplicationContextUtils.registerEnvironmentBeans(beanFactory,
			getServletContext());
	// dispatcherServlet、characterEncodingFilter、hiddenHttpMethodFilter、httpPutFormContentFilter、requestContextFilter
	for (ServletContextInitializer beans : getServletContextInitializerBeans()) {
		beans.onStartup(servletContext);
	}
}

// RegistrationBean
public final void onStartup(ServletContext servletContext) throws ServletException {
	String description = getDescription();
	if (!isEnabled()) {
		logger.info(StringUtils.capitalize(description)
				+ " was not registered (disabled)");
		return;
	}
	// 动态添加filter，配置注册设置
	register(description, servletContext);
}
```

