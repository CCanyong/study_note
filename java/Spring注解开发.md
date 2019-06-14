## Spring注解开发

### 一、IOC

#### 1）@Configuration

​	作用：声明注解类

#### 2）@ComponentScan

 	1. 作用：配置包扫描
 	2. 参数
      	1. value指定包名
      	2. excludeFilters指定不扫描哪些包(@Filter(type指定过滤类型)，
      	3. includeFilters指定扫描哪些包，需与useDefaultFilters = false, 关闭默认扫描所有包
      	4. 过滤规则：
          - FilterType.ANNOTATION：根据注解类型进行过滤，如@Controller，即过滤Controller.class
          - FilterType.ASSIGNABLE_TYPE：根据类型进行过滤，如Person.class
          - FilterType.ASPECTJ：使用ASPECTJ表达式
          - FilterType.REGEX：使用正则表达式
          - FilterType.CUSTOM：使用自定义规则，需实现接口TypeFilter，在里面做判断

```java
//声明注解类
@Configuration
@ComponentScan(value = "com.ccy.bean", 
		excludeFilters = {@Filter(type=FilterType.ANNOTATION, classes={Controller.class} )},
		/*includeFilters = {@Filter(type=FilterType.ANNOTATION,classes={Service.class})},*/
		includeFilters = {@Filter(type=FilterType.CUSTOM,classes=MyTypeFilter.class)},
		useDefaultFilters = false)
public class MainConfig {
	//在容器中注入一个bean，beanName默认为方法名，或直接在@Bean中配置
	@Bean("person")
	public Person person() {
		return new Person("ccy",20);
	}
}
```

```java
public class MyTypeFilter implements TypeFilter{

	/**
	 * 对包扫描到的类做筛选，返回true则放入容器，false则不放入
	 * 
	 * @param metadataReader 读取当前扫描类的信息
	 * @param metadataReaderFactory 所有扫描类的信息
	 */
	public boolean match(MetadataReader metadataReader, MetadataReaderFactory 															metadataReaderFactory) throws IOException {
		//获取当前类的注解信息
		AnnotationMetadata annotationMetadata = metadataReader.getAnnotationMetadata();
		ClassMetadata classMetadata = metadataReader.getClassMetadata();
		System.out.println(classMetadata.getClassName());
		return false;
	}
}
```

#### 3）@Scope

1. 调整作用域

   1. singleton：单例(默认); IOC容器启动时会调用方法创建对象放到容器中
   2. prototype：多例;	IOC容器启动时不会调用方法创建对象放到容器中, 每次从IOC容器中获取对象时调用该方法创建对象
   3. request：同一次请求创建一个实例
   4. session：同一个session创建一个实例

2. @Lazy 懒加载

   1. 必须是单实例bean，默认在容器启动时创建对象
   2.  懒加载-->第一次获取对象时创建并初始化

   ````
   @Lazy
   @Scope("singleton")
   @Bean("person")
   public Person person() {
   	return new Person("person",20);
   }
   ````

#### 4）@Conditional 

1. 按照一定条件进行判断，满足条件是才将bean注册到容器中 

2. 在@Conditional()中放入一个class数组，class实现了Condition接口

   ```java
   @Bean("window")
   @Conditional(value = {WindowsCondition.class})
   public Person person01() {
   	return new Person("微软",20);
   }
   @Bean("linux")
   @Conditional(value = {LinuxCondition.class})
   public Person person02() {
   	return new Person("ubuntu",20);
   }
   ```

   ```java
   public class LinuxCondition implements Condition {
   	/**
   	 * ConditionContext 判断条件能使用的上下文(环境)
   	 * AnnotatedTypeMetadata 注释信息
   	 */
   	public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
   		//获取环境信息
   		Environment environment = context.getEnvironment();
   		String property = environment.getProperty("os.name");
   		if(property.contains("Linux")) {
   			return true;
   		}
   		return false;
   	}
   }
   ```

   ```java
   public class WindowsCondition implements Condition {
   	/**
   	 * ConditionContext 判断条件能使用的上下文(环境)
   	 * AnnotatedTypeMetadata 注释信息
   	 */
   	public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
   		//获取环境信息
   		Environment environment = context.getEnvironment();
   		String property = environment.getProperty("os.name");
   		if(property.contains("Windows")) {
   			return true;
   		}
   		return false;
   	}
   }
   ```

 #### 5）@Import

1. 在IOC容器中注册bean的方法
    	1. 使用包扫描+注解：适用于自己编写的类使用
    	2. @Bean：适用于导入第三方包
    	3. 使用@@Import：
        1. @Import({Student.class,...})：直接导入(可导入数组)，beanName为全类名
        2. ImportSelector：实现类ImportSelector，方法返回一个需导入的全类名数组
        3. ImportBeanDefinitionRegistrar：手动注册bean
   	4. 通过实现FactoryBean接口来引入对象：beanName为工厂类全类名，类型则为getObject方法返回的类型
       1. 使用FactoryBean方法,beanName虽然为Factory的全类名，但bean类型为getObject()返回的类型
       2. 获取真正工厂bean对象，beanName前添加&

```java
@Import({Student.class,MyImportSelector.class,MyImportBeanDefinitionRegistrar.class,BookFactory.class})
public class MainConfig2 {
	
}
```

```java
public class MyImportBeanDefinitionRegistrar implements ImportBeanDefinitionRegistrar {
	/**
	 * AnnotationMetadata 当前类的注解信息
	 * BeanDefinitionRegistry 注册类
	 * 		通过registerBeanDefinition()方法手动注册
	 */
	public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
		//bean的定义信息(Bean的类型,Scope等)
		BeanDefinition beanDefinition = new RootBeanDefinition(Car.class);
		//注册bean，自定义beanName
		registry.registerBeanDefinition("car", beanDefinition);
	}

}
```

```java
public class MyImportSelector implements ImportSelector {
	/**
	 * AnnotationMetadata 当前标注@Import注解的类的所有信息
	 * @return 为需导入的全类名数组
	 */
	public String[] selectImports(AnnotationMetadata importingClassMetadata) {
		
		//可返回空数组，但不可返回null，会报空指针异常
		return new String[] {"com.ccy.bean.Teacher"};
	}
}
```



