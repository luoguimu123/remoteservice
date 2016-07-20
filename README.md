# remoteservice

这是一个服务化工具类项目，其基于Spring，Aspect，Apache Commons，ZooKeeper等基础项目。项目意在提供服务化(发现，治理，调用,测试)一体化架构方案，暂时
只实现了部分基础功能。

框架提供了2种实现服务化调用的方案

1：面向注解的方案。
本框架提供如下4个主要核心注解
@Provider:表示服务的提供者，面向类的注解，被注解在服务提供方的类(具体实现类而非接口)之上。
@ProviderMethod：表示提供的具体服务，面向方法的注解，需要被注解在服务提供实现类中对外暴露服务的方法之上。其有2个属性，分别为group以及
service,分别对应Zookeepr中的/group/service/sub-xxxxxx这样的目录树中的前2级目录。
@Consumer:表示服务的消费者，面向类的注解，被注解在消费端持有的基本接口之上。
@ConsumerMethod：表示具体的消费服务方法，面向方法的注解，其属性以及意义与@ProviderMethod一样。

我们以具体的代码为例
假设服务提供端和服务消费端的基本服务接口如下：
public interface HelloService {

	String hello(String name);
	
	String hello2(String name);
	
}

服务提供端的对于该接口的一个具体实现如下：
@Provider
public class HelloServiceImpl implements HelloService {

	@Override
	@ProviderMethod(service="helloservice", group = "hello")
	public String hello(String name) {
		return "hello "+name;
	}

	@Override
	@ProviderMethod(service="helloservice1",group="hello")
	public String hello2(String name) {
		// TODO Auto-generated method stub
		return "hello, hello "+name;
	}

}
为了将这个类以远程服务调用的方式暴露出去，我们在Spring基于Java文件的零配置方式中的配置如下：

@Configuration
public class Config {

	@Bean
	public HelloService helloService() {
		return new HelloServiceImpl();
	}
	
	@Bean(initMethod="init", destroyMethod="destory")
	public ProviderZkUtil providerZkUtil() {
		ProviderZkUtil zkUtil = new ProviderZkUtil();
		zkUtil.setAddress("127.0.0.1:2181");
		zkUtil.setUrl("127.0.0.1:8080");
		return zkUtil;
	}
	
	@Bean
	public ProviderBootStrap providerBootStrap() {
		ProviderBootStrap strap = new ProviderBootStrap();
		strap.setProviderZkUtil(providerZkUtil());
		return strap;
	}
	
}
其中ProviderZkUtil是为服务提供方提供ZooKeeper操作的工具类，ProviderBootStrap则是服务提供方最核心的启动类。
而在客户端
@Consumer
public interface HelloService {

	@ConsumerMethod(group="hello", service="helloservice")
	String hello(String name);
	
	@ConsumerMethod(group="hello", service="helloservice1")
	String hello2(String name);
	
}

@Configuration
@EnableAspectJAutoProxy
public class Config {

	@Bean
	public HelloService helloService() {
		return new HelloService() {
			
			@Override
			@ConsumerMethod(group="hello", service="helloservice1")
			public String hello2(String name) {
				// TODO Auto-generated method stub
				return "2";
			}
			
			@Override
			@ConsumerMethod(group="hello", service="helloservice")
			public String hello(String name) {
				// TODO Auto-generated method stub
				return "1";
			}
		};
	}
	
	@Bean
	public TestService testService() {
		TestService testService = new TestService();
		testService.setHelloService(helloService());
		return testService;
	}
	
	@Bean(initMethod="init")
	public ConsumerZkUtil zkUtil() {
		ConsumerZkUtil zkUtil = new ConsumerZkUtil();
		zkUtil.setAddress("127.0.0.1:2181");
		return zkUtil;
	}
	
	@Bean
	public ConsumerAspect aspect() {
		ConsumerAspect aspect = new ConsumerAspect();
		aspect.setZkUtil(zkUtil());
		return aspect;
	}
	
}
其中配置类中可以实习化一个HelloService的匿名类，也可以直接将这个匿名类放在testService的属性注入中，其实现可以随意写。最终会被覆盖。
ConsumerAspect采用AOP的方式将所有对接口的调用实例化到远程服务的调用中去。

2：基于Java类的方案
此方案主要是通过采用工具类包装的方案来实现同样的功能。

服务提供端和消费端基本的接口和实现类同上，但是不需要其上多余的注解。

服务提供方的配置类如下：
@Configuration
public class FactoryConfig {

	@Bean
	public HelloService helloServiceImpl() {
		return new HelloServiceImpl();
	}

	@Bean(initMethod="init", destroyMethod="init")
	public ProviderZkUtil providerZkUtil() {
		ProviderZkUtil zkUtil = new ProviderZkUtil();
		zkUtil.setAddress("127.0.0.1:2181");
		zkUtil.setUrl("127.0.0.1:8080");
		return zkUtil;
	}
	
	@Bean
	public Object helloServiceFactory() {
		
		ProviderBeanFactory factory = new ProviderBeanFactory();
		factory.setTarget(helloServiceImpl());
		factory.setZkUtil(providerZkUtil());
		factory.setList(Arrays.asList("hello1:helloservice", "hello1:helloservice1"));
		return factory;
	}
	
	
}

ProviderBeanFactory实现了FactoryBean接口并生成一个伪动态代理将所有的服务注册到服务中心上，其中List参数是服务路径，其对应的是group:service

同理服务消费端的配置类如下：

@Configuration
@ComponentScan
public class FactoryConfig {
	
	@Bean(initMethod="init", destroyMethod="init")
	public ConsumerZkUtil zkUtil() {
		ConsumerZkUtil zkUtil = new ConsumerZkUtil();
		zkUtil.setAddress("127.0.0.1:2181");
		return zkUtil;
	}
	
	@Bean
	@Qualifier(value="helloServiceFactoryBean")
	public ConsumerBeanFactory helloServiceFactoryBean() {
		ConsumerBeanFactory factory =  new ConsumerBeanFactory();
		factory.setInterfaceName(HelloService.class.getName());
		factory.setTarget(new HelloService() {
			
			@Override
			public String hello2(String name) {
				// TODO Auto-generated method stub
				return null;
			}
			
			@Override
			public String hello(String name) {
				// TODO Auto-generated method stub
				return null;
			}
		});
		factory.setZkUtil(zkUtil());
		Map<String, String> map = new HashMap<>();
		map.put("hello", "hello1:helloservice");
		map.put("hello2", "hello1:helloservice1");
		factory.setMap(map);
		return factory;
	}
}
其中我们需要注意的时候，生成ConsumerBeanFactory的bean方法中，返回的任然是该类，但是在其他地方需要注入HelloService的时候，仍然可以通过
自己注解的方式获得该工厂Bean生成的代理Bean。需要用(Resource注解防止Autowire取不到Bean)


