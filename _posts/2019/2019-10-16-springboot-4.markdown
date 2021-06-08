---
layout:     post
title:      "SpringBoot(四) 集成apollo遇到的事儿"
date:       2019-10-12
author:     "ZhouJ000"
header-img: "img/in-post/2019/post-bg-2019-headbg.jpg"
catalog: true
tags:
    - springBoot
--- 

[SpringBoot(一) 启动与自动配置](https://zhouj000.github.io/2018/10/05/springboot-1/)  
[SpringBoot(二) starter与servlet容器](https://zhouj000.github.io/2018/10/10/springboot-2/)  
[SpringBoot(三) Environment](https://zhouj000.github.io/2019/10/08/springboot-3/)  
[SpringBoot(四) 集成apollo遇到的事儿](https://zhouj000.github.io/2019/10/16/springboot-4/)  
[SpringBoot(五) 健康检查(上)](https://zhouj000.github.io/2020/01/28/springboot-5/)  
[SpringBoot(六) 健康检查(下)](https://zhouj000.github.io/2020/02/19/springboot-6/)  




# 问题

公司新写项目，使用公司封装的SpringBoot-Starter，其中配置中心使用的是**Apollo**。看了下Apollo配置中心发现可以配置环境和集群，查看了starter源码发现已经配置好了Apollo地址和环境，由于老项目使用的disconf在相同环境下是按机器有多个配置的，因此切换到Apollo相应就是使用**多套集群配置**

从封装代码中得知，是使用**继承ApplicationContextInitializer**来初始化的，在initialize方法中配置**System的Property**信息，得以将application.properties中配置的信息从**ConfigurableEnvironment**中获取并设置到System。封装的代码中会设置env、app.id、apollo.bootstrap.enabled等配置，但是并没有设置集群信息，这样会使用默认default集群

# 解决

从网上查询得知Apollo集群配置是使用**apollo.cluster**的，那么在application.properties中加入这个配置，启动一下果然没有读取到集群配置信息，那么按照封装的初始化思路，写入apollo.cluster属性：
```
public class InitApolloCluster implements ApplicationContextInitializer<ConfigurableApplicationContext>, Ordered {

    @Override
    public void initialize(ConfigurableApplicationContext applicationContext) {
        ConfigurableEnvironment environment = applicationContext.getEnvironment();
        String cluster = System.getProperty("apollo.cluster", environment.getProperty("apollo.cluster"));
        if(StringUtils.isNotBlank(cluster)){
            // 指定了cluster，不然使用default
            System.setProperty("apollo.cluster", cluster);
        }
    }

    @Override
    public int getOrder() {
		// 在封装的初始化类之后运行，其实没关系
        return Ordered.HIGHEST_PRECEDENCE + 2;
    }
}
```
然后在WEB-INFO下配置**spring.factories**(在spring.factories中要设置好apollo.cluster值)：
```
org.springframework.context.ApplicationContextInitializer=XXXX...
```
现在启动SpringBoot项目，果然使用了对应集群的Apollo配置


# 探究

在成功配置后，debug了一下，发现虽然进了初始化方法，但是前后**执行了2次**，并且第一次还没有从Environment取到配置，有点奇怪，从调用链可以看到是从SpringApplication的**applyInitializers**入口进去的：
```
protected void applyInitializers(ConfigurableApplicationContext context) {
	// 获取spring.factories配置的所有ApplicationContextInitializer实现类
	for (ApplicationContextInitializer initializer : getInitializers()) {
		Class<?> requiredType = GenericTypeResolver.resolveTypeArgument(
				initializer.getClass(), ApplicationContextInitializer.class);
		Assert.isInstanceOf(requiredType, context, "Unable to call initializer.");
		initializer.initialize(context);
	}
}


// 路径从SpringApplication的run方法到prepareContext方法
public ConfigurableApplicationContext run(String... args) {
	// ...
	ConfigurableEnvironment environment = prepareEnvironment(listeners,
					applicationArguments);
	// ...
	// 准备上下文
	prepareContext(context, environment, listeners, applicationArguments,
					printedBanner);
	// ...
}

private void prepareContext(ConfigurableApplicationContext context,
			ConfigurableEnvironment environment, SpringApplicationRunListeners listeners,
			ApplicationArguments applicationArguments, Banner printedBanner) {
	context.setEnvironment(environment);
	postProcessApplicationContext(context);
	// 调用所有的初始化类
	applyInitializers(context);
	listeners.contextPrepared(context);
	// ...
}
```
到这里发现就是正常的启动步骤，但是继续往下看，发现怎么这个run方法的调用并不是我们的启动类，而是从**新build的SpringApplication**去执行的

从[上一篇SpringBoot Environment](https://zhouj000.github.io/2019/10/08/springboot-3/)中写到，SpringApplication启动后准备环境，并且会调用`listeners.environmentPrepared(environment)`向监听器发送事件，其中**SpringApplicationRunListen类型的监听器**(借鉴了Spring的ApplicationListener)**是SpringApplication的run()方法执行的过程中贯穿始终的事件监听器**。在这个debug链路中可以看到，最终invokeListener方法执行了**BootstrapApplicationListener**的onApplicationEvent方法，而这个BootstrapApplicationListener类是springframework.**cloud**的：
```
public void onApplicationEvent(ApplicationEnvironmentPreparedEvent event) {
	ConfigurableEnvironment environment = event.getEnvironment();
	if (!environment.getProperty("spring.cloud.bootstrap.enabled", Boolean.class,
			true)) {
		return;
	}
	// don't listen to events in a bootstrap context
	if (environment.getPropertySources().contains(BOOTSTRAP_PROPERTY_SOURCE_NAME)) {
		return;
	}
	// 创建bootstrapServiceContext
	ConfigurableApplicationContext context = bootstrapServiceContext(environment,
			event.getSpringApplication());
	apply(context, event.getSpringApplication(), environment);
}
```
创建过程中会调用SpringApplicationBuilder的run方法：
```
private ConfigurableApplicationContext bootstrapServiceContext(
		ConfigurableEnvironment environment, final SpringApplication application) {
	StandardEnvironment bootstrapEnvironment = new StandardEnvironment();
	MutablePropertySources bootstrapProperties = bootstrapEnvironment
			.getPropertySources();
	for (PropertySource<?> source : bootstrapProperties) {
		bootstrapProperties.remove(source.getName());
	}
	String configName = environment
			.resolvePlaceholders("${spring.cloud.bootstrap.name:bootstrap}");
	String configLocation = environment
			.resolvePlaceholders("${spring.cloud.bootstrap.location:}");
	Map<String, Object> bootstrapMap = new HashMap<>();
	bootstrapMap.put("spring.config.name", configName);
	if (StringUtils.hasText(configLocation)) {
		bootstrapMap.put("spring.config.location", configLocation);
	}
	bootstrapProperties.addFirst(
			new MapPropertySource(BOOTSTRAP_PROPERTY_SOURCE_NAME, bootstrapMap));
	for (PropertySource<?> source : environment.getPropertySources()) {
		bootstrapProperties.addLast(source);
	}
	ClassLoader classLoader = Thread.currentThread().getContextClassLoader();
	// Use names and ensure unique to protect against duplicates
	List<String> names = SpringFactoriesLoader
			.loadFactoryNames(BootstrapConfiguration.class, classLoader);
	for (String name : StringUtils.commaDelimitedListToStringArray(
			environment.getProperty("spring.cloud.bootstrap.sources", ""))) {
		names.add(name);
	}
	// TODO: is it possible or sensible to share a ResourceLoader?
	// 这里创建了SpringApplicationBuilder
	SpringApplicationBuilder builder = new SpringApplicationBuilder()
			.profiles(environment.getActiveProfiles()).bannerMode(Mode.OFF)
			.environment(bootstrapEnvironment)
			.properties("spring.application.name:" + configName)
			.registerShutdownHook(false).logStartupInfo(false).web(false);
	List<Class<?>> sources = new ArrayList<>();
	for (String name : names) {
		Class<?> cls = ClassUtils.resolveClassName(name, null);
		try {
			cls.getDeclaredAnnotations();
		}
		catch (Exception e) {
			continue;
		}
		sources.add(cls);
	}
	builder.sources(sources.toArray(new Class[sources.size()]));
	AnnotationAwareOrderComparator.sort(sources);
	// 这里又调用了builder的run方法
	final ConfigurableApplicationContext context = builder.run();
	// Make the bootstrap context a parent of the app context
	addAncestorInitializer(application, context);
	// It only has properties in it now that we don't want in the parent so remove
	// it (and it will be added back later)
	bootstrapProperties.remove(BOOTSTRAP_PROPERTY_SOURCE_NAME);
	mergeDefaultProperties(environment.getPropertySources(), bootstrapProperties);
	return context;
}
```
从SpringApplicationBuilder的run方法可以看到，**重新创建了SpringApplication**并调用其run方法
```
public ConfigurableApplicationContext run(String... args) {
	if (this.running.get()) {
		// If already created we just return the existing context
		return this.context;
	}
	configureAsChildIfNecessary(args);
	if (this.running.compareAndSet(false, true)) {
		synchronized (this.running) {
			// If not already running copy the sources over and then run.
			this.context = build().run(args);
		}
	}
	return this.context;
}

public SpringApplication build() {
	return build(new String[0]);
}
public SpringApplication build(String... args) {
	configureAsChildIfNecessary(args);
	// 这里的application是在SpringApplicationBuilder创建时的构造函数中创建的
	// new SpringApplication(sources)
	this.application.setSources(this.sources);
	return this.application;
}
```
那么结果就出来了，**第一次执行的是由Spring Cloud创建的SpringApplication**，这个时候由于是新的SpringApplication，因此从Environment中是没有取到我们配置的apollo.cluster属性的。然后第二次，在prepareEnvironment执行后的prepareContext中又做了一遍相同的事情，而这次是我们一开始启动的那个SpringApplication，因此会取到apollo.cluster属性并成功放入System

答案很明显了，是spring-cloud-context包的原因导致的，然后我尝试从pom将封装的starter中exclusions这个引用，然后发现启动报错了，查看错误原因发现是创建类失败，点进去发现是封装的配置刷新类中依赖了Spring Cloud的**RefreshScope类**，导致了ClassNofFound

逻辑是在刷新类的postProcessAfterInitialization方法中，当查询到**ConfigurationProperties与RefreshScope**这2个注解修饰的配置类时，从apollo.ConfigService.getAppConfig()中添加**改变监听器addChangeListener**，在发生改变时**刷新**这个配置bean：
```
// 写在postProcessAfterInitialization中，即实现了BeanPostProcessor
ConfigService.getAppConfig().addChangeListener(new ConfigChangeListener() {
	@Override
	public void onChange(ConfigChangeEvent changeEvent) {
		Object oldSettings = springFactory.getBean(beanName);
		logger.info("before refresh {}", oldSettings.toString());
		refreshScope.refresh(beanName);
		Object newSettings = springFactory.getBean(beanName);
		logger.info("after refresh {}", newSettings.toString());
	}
});
```
我在网上搜了一下，貌似Apollo的配置修改更新都是这么写的，区别是onChange方法使用@ApolloConfigChangeListener注解触发也行。那么看来如果要实现热加载是需要依赖spring-cloud-context包的。然后点进去看一下refresh方法：
```
@ManagedOperation(description = "Dispose of the current instance of bean name provided and force a refresh on next method execution.")
public boolean refresh(String name) {
	// 不以scopedTarget.开头，加上
	if (!name.startsWith(SCOPED_TARGET_PREFIX)) {
		// User wants to refresh the bean with this name but that isn't the one in the
		// cache...
		name = SCOPED_TARGET_PREFIX + name;
	}
	// Ensure lifecycle is finished if bean was disposable
	// 卸载类
	if (super.destroy(name)) {
		this.context.publishEvent(new RefreshScopeRefreshedEvent(name));
		return true;
	}
	return false;
}
```
那么看起来这个刷新就是**卸载类后重新加载**，得以读取到最新的配置，实现了"热加载"。从一开始的判断上可以得知，卸载的这个类可能是个代理类，那么@RefreshScope这个注解应该会**生成代理类**，所以要在需要热加载的配置类上加上注解(需要注意，@RefreshScope作用的类，不能是final类，否则启动时会报错)



扩展：  
1、Apollo可以通过namespace(命名空间)关联引入重复的配置，这个还是比较方便的  
2、Apollo的配置由：application(应用) + environment(环境) + cluster(集群) + namespace(命名空间，配置使用@EnableApolloConfig({"xxx"}))决定

[@RefreshScope那些事](https://www.jianshu.com/p/188013dd3d02)  
[Spring作用域 (Scope：Request,Session,Thread,Refresh) 的代理机制源码解析](https://blog.csdn.net/qq_27529917/article/details/82781067)  
[spring cloud的RefreshScope注解进行热部署](https://blog.csdn.net/weixin_40318210/article/details/87954179)  
