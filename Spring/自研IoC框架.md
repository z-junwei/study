## 实现思路

**一个框架最基本的功能：**

- 解析配置
- 定位与注册对象
- 注入对象
- 提供通用的工具类

**IoC容器的实现：**

`创建注解 -> 提取标记对象 -> 实现容器 -> 依赖注入`

## 实现

### 创建注解

这里注解的作用和Spring的类似。

```java
@Target(ElementType.TYPE)//作用在类上
@Retention(RetentionPolicy.RUNTIME)//运行时通过反射获取注解信息
public @interface Controller {
}
```

```Java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface Service {
}
```

### 提取标记对象

实现思路：

- 指定范围，获取范围内的所有类
- 遍历所有类，获取被注解标记的类并加载进容器里

```java
package org.IoC.utils;

import lombok.extern.slf4j.Slf4j;
import sun.nio.cs.ext.IBM037;

import javax.accessibility.Accessible;
import java.io.File;
import java.io.FileFilter;
import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
import java.lang.reflect.InvocationTargetException;
import java.net.URL;
import java.util.HashSet;
import java.util.Set;

@Slf4j
public class ClassUtil {

	public static final String FILE_PROTOCOL = "file";

	/**
	 * 获取包下的所有类
	 *
	 * 需要完成的事情：
	 * 1、获取类的加载器：获取项目发布的实际路径
	 * 2、通过类加载器获取到加载的资源信息
	 * 3、依据不同的类型，采用不同的方式获取资源的集合
	 *
	 * @param packageName
	 * @return
	 */
	public static Set<Class<?>> getPackageClass(String packageName){
		//1、获取类的加载器：获取项目发布的实际路径
		ClassLoader classLoader = getClassLoader();
		//2、通过类加载器获取到加载的资源信息
		URL url = classLoader.getResource(packageName.replace(".","/"));
		if (url == null){
			log.warn("unable to retrieve anything from package, " + packageName);
			return null;
		}
		//3、依据不同的类型，采用不同的方式获取资源的集合，这里只要file资源
		Set<Class<?>> classSet = null;
		if (url.getProtocol().equalsIgnoreCase(FILE_PROTOCOL)){
			classSet = new HashSet<Class<?>>();
			File packageDirectory = new File(url.getPath());
			getClassFile(classSet, packageDirectory, packageName);
		}
		return classSet;
	}

	/**
	 * 递归获取
	 *
	 * @param classSet class资源的集合
	 * @param fileSource 文件或者目录
	 * @param packageName 包名
	 */
	private static void getClassFile(Set<Class<?>> classSet, File fileSource, String packageName) {
		//如果不是文件，直接return
		if (!fileSource.isDirectory()){
			return;
		}
		//如果是一个文件夹，则调用listFiles方法获取文件夹下的文件或文件夹
		File[] files = fileSource.listFiles(new FileFilter() {
			@Override
			public boolean accept(File file) {
				//子目录返回true继续
				if (file.isDirectory()){
					return true;
				}else {
					//获取文件的绝对路径
					String absoluteFilePath = file.getAbsolutePath();
					if (absoluteFilePath.endsWith(".class")){
						//若是class文件，则直接加载
						addToClassSet(absoluteFilePath);
					}
				}
				return false;
			}

			private void addToClassSet(String absoluteFilePath) {
				//1.从class文件的绝对路径里提取包含了package的类名
				//注意：路径最好不要带有中文
				//如：D:\study\SpringSourceCode\spring-framework-5.2.0.RELEASE\simpleframework\src\main\java\com\zjw\entity\User.java
				//弄成：com.zjw.entity.Student
				absoluteFilePath = absoluteFilePath.replace(File.separator, ".");
				String className = absoluteFilePath.substring(absoluteFilePath.indexOf(packageName));
				className = className.substring(0, className.lastIndexOf("."));

				//2.通过反射获取对应的class对象并加入到classSet
				Class targetClass = loadClass(className);
				classSet.add(targetClass);
			}
		});

		if (files != null){
			for (File file : files) {
				//递归获取
				getClassFile(classSet, file, packageName);
			}
		}
	}

	/**
	 * 反射获取类
	 * @param className
	 * @return
	 */
	public static Class<?> loadClass(String className){
		try {
			return Class.forName(className);
		} catch (ClassNotFoundException e) {
			log.error("load class error, ", e);
			throw new RuntimeException(e);
		}
	}

	/**
	 * 获取ClassLoader
	 * @return
	 */
	public static ClassLoader getClassLoader(){
		return Thread.currentThread().getContextClassLoader();
	}

	/**
	 * 实例化类
	 * @param clazz
	 * @param accessible:是否支持创建出私有class对象的实例
	 * @param <T>
	 * @return
	 */
	public static <T> T newInstance(Class<?> clazz, boolean accessible) {
		try {
			Constructor constructor = clazz.getDeclaredConstructor();
			constructor.setAccessible(accessible);
			return (T) constructor.newInstance();
		} catch (Exception e) {
			log.error("newInstance error");
			throw new RuntimeException(e);
		}
	}

	/**
	 * 设置类的属性值
	 * @param field 成员变量
	 * @param target 类实例
	 * @param value 成员变量的值
	 * @param accessible 是否允许设置私有属性
	 */
	public static void setField(Field field, Object target, Object value, boolean accessible){
		field.setAccessible(accessible);
		try {
			field.set(target, value);
		} catch (IllegalAccessException e) {
			log.error("setField error", e);
			throw new RuntimeException(e);
		}
	}


	public static void main(String[] args) {
		File packageDirectory = new File("D:\\study\\SpringSourceCode\\spring-framework-5.2.0.RELEASE\\simpleframework\\target\\classes\\com\\zjw\\entity");
		System.out.println(packageDirectory.isDirectory());
	}
}

```

编写测试类测试：

```java
package org.IoC.core.utils;

import org.junit.jupiter.api.Assertions;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;

import java.util.Set;

public class ClassUnitTest {
	@DisplayName("提取目标类方法：getClassPackageTest")
	@Test
	public void getClassPackageTest(){
		Set<Class<?>> classSet = ClassUtil.getPackageClass("com.zjw.entity");
		System.out.println(classSet);
		Assertions.assertEquals(12, classSet.size());
	}
}


```

测试结果：

![image-20221104151955799](https://zjw-note-images.oss-cn-shanghai.aliyuncs.com/img/MySelfIoC/202211041602520.png)

### 实现容器

#### 单例模式问题

自研容器推荐使用饿汉模式，方便简洁，但是要考虑安全性。

```java
//饿汉模式
public class StarvingSingleton {
	private static final StarvingSingleton starvingSingleton = new StarvingSingleton();
	private StarvingSingleton(){};
	public static StarvingSingleton getInstance(){
		return starvingSingleton;
	}
}
```

这种并不安全，可以利用反射机制来破除无参构造private的防御。

> **反射为啥能破坏单例？**
>
> 通过反射获得单例类的构造函数，由于该构造函数是private的，通过setAccessible(true)指示反射的对象在使用时应该取消 [Java](http://lib.csdn.net/base/javase) 语言访问检查，使得私有的构造函数能够被访问，这样使得单例模式失效。

```java
//反射破环
public class SingletonDemo {
	public static void main(String[] args) throws NoSuchMethodException, InvocationTargetException, InstantiationException, IllegalAccessException {
		System.out.println(StarvingSingleton.getInstance());
		Class clazz = StarvingSingleton.class;
		Constructor constructor = clazz.getDeclaredConstructor();
		constructor.setAccessible(true);
		System.out.println(constructor.newInstance());
	}
}
```

运行结果：

![image-20221102105920885](https://zjw-note-images.oss-cn-shanghai.aliyuncs.com/img/MySelfIoC/202211041602014.png)

**解决办法：**

采用枚举类。

> **枚举为啥不能被反射破环？**
>
> 参考：https://www.cnblogs.com/call-me-pengye/p/11214435.html

```java
public class StarvingSingletonEnum {

	private StarvingSingletonEnum(){};

	public static StarvingSingletonEnum getInstance(){
		return ContainerHolder.HOLDER.instance;
	}

	private enum ContainerHolder{
		HOLDER;
		private StarvingSingletonEnum instance;
		ContainerHolder(){
			instance = new StarvingSingletonEnum();
		}
	}	
}
```

测试：

```java
public class SingletonDemo {
	public static void main(String[] args) throws NoSuchMethodException, InvocationTargetException, InstantiationException, IllegalAccessException {
		System.out.println(StarvingSingletonEnum.getInstance());
		Class clazz = StarvingSingletonEnum.class;
		Constructor constructor = clazz.getDeclaredConstructor();
		constructor.setAccessible(true);
		StarvingSingletonEnum starvingSingletonEnum = (StarvingSingletonEnum) constructor.newInstance();
		System.out.println(starvingSingletonEnum.getInstance());
	}
}
```

![202211021348290](https://zjw-note-images.oss-cn-shanghai.aliyuncs.com/img/MySelfIoC/202211041602966.png)

直接测试枚举类：

```java
public static void main(String[] args) throws NoSuchMethodException, InvocationTargetException, InstantiationException, IllegalAccessException {
		Class clazz = ContainerHolder.class;
		Constructor constructor = clazz.getDeclaredConstructor();
//		Constructor constructor = clazz.getDeclaredConstructor(String.class, int.class);
		constructor.setAccessible(true);
		System.out.println(constructor.newInstance());
		System.out.println(StarvingSingletonEnum.getInstance());
}
```

![202211021349240](https://zjw-note-images.oss-cn-shanghai.aliyuncs.com/img/MySelfIoC/202211041601162.png)



#### 容器的组成部分

- 保存Class对象以及其实例的载体
- 容器的加载
- 容器的操作方式

##### 保存Class对象以及其实例的载体

```java
	/**
	 * 存放所有被配置标记的目标对象的Map
	 */
	private final Map<Class<?>, Object> beanMap = new ConcurrentHashMap<>();
```

##### 容器的加载

- 配置的管理与获取

```java
	/**
	 * 加载bean的注解列表
	 */
	private static final List<Class<? extends Annotation>> BEAN_ANNOTATION =
			Arrays.asList(
					Controller.class,
					Service.class
			);
```

- 获取指定范围内的Class对象

​		上面的`getPackageClass`方法。

- 依据配置提取Class对象，连同实例一起存入容器

```java
	/**
	 * 容器是否已经加载过bean
	 */
	private boolean loaded = false;

	/**
	 * 是否已经加载过bean
	 */
	public boolean isLoaded(){
		return loaded;
	}
	/**
	 * 扫描加载所有bean
	 * @param packageName
	 */
	public synchronized void loadBeans(String packageName){
		//判断bean容器是否被加载过
		if (loaded == true){
			log.warn("BeanContainer has been loaded......");
			return;
		}
		Set<Class<?>> classSet = ClassUtil.getPackageClass(packageName);
		if (ValidationUtil.isEmpty(classSet)){
			log.warn("get nothing from packageName " + packageName);
			return;
		}

		for (Class<?> clazz : classSet) {
			for (Class<? extends Annotation> annotation : BEAN_ANNOTATION) {
				//如果类上面标记了定义的注解
				if (clazz.isAnnotationPresent(annotation)){
					//将本类作为键，目标类的实例作为值，存入Map
					beanMap.put(clazz, ClassUtil.newInstance(clazz, true));
				}
			}
		}
		loaded = true;
	}
```

```java
package org.IoC.core.utils;

import java.util.Collection;
import java.util.Map;

public class ValidationUtil {
	public static boolean isEmpty(Collection<?> obj){
		return (obj == null || obj.isEmpty());
	}

	public static boolean isEmpty(String obj){
		return (obj == null || "".equals(obj));
	}

	public static boolean isEmpty(Object[] obj){
		return (obj == null || obj.length == 0);
	}

	public static boolean isEmpty(Map<?, ?> obj){
		return (obj == null || obj.isEmpty());
	}
}

```

**BeanContainer完整代码：**

```java
package org.IoC.core;

import org.IoC.core.annotation.Component;
import org.IoC.core.annotation.Controller;
import org.IoC.core.annotation.Repository;
import org.IoC.core.annotation.Service;
import org.IoC.utils.ClassUtil;
import org.IoC.utils.ValidationUtil;
import lombok.AccessLevel;
import lombok.NoArgsConstructor;
import lombok.extern.slf4j.Slf4j;

import java.lang.annotation.Annotation;
import java.util.Arrays;
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.concurrent.ConcurrentHashMap;

@Slf4j
@NoArgsConstructor(access = AccessLevel.PRIVATE)
public class BeanContainer {
	/**
	 * 存放所有被配置标记的目标对象的Map
	 */
	private final Map<Class<?>, Object> beanMap = new ConcurrentHashMap<>();

	/**
	 * 加载bean的注解列表
	 */
	private static final List<Class<? extends Annotation>> BEAN_ANNOTATION =
			Arrays.asList(
					Component.class,
					Controller.class,
					Repository.class,
					Service.class
			);

	/**
	 * 获取Bean的容器实例
	 */
	public static BeanContainer getInstance(){
		return ContainerHolder.HOLDER.instance;
	}


	/**
	 * 获取beans的数量
	 * @return
	 */
	public int getBeanSize() {
		return beanMap.size();
	}

	/**
	 * 枚举
	 */
	private enum ContainerHolder{
		HOLDER;
		private BeanContainer instance;
		ContainerHolder(){
			instance = new BeanContainer();
		}
	}

	/**
	 * 容器是否已经加载过bean
	 */
	private boolean loaded = false;

	/**
	 * 是否已经加载过bean
	 */
	public boolean isLoaded(){
		return loaded;
	}
	/**
	 * 扫描加载所有bean
	 * @param packageName
	 */
	public synchronized void loadBeans(String packageName){
		//判断bean容器是否被加载过
		if (loaded == true){
			log.warn("BeanContainer has been loaded......");
			return;
		}
		Set<Class<?>> classSet = ClassUtil.getPackageClass(packageName);
		if (ValidationUtil.isEmpty(classSet)){
			log.warn("get nothing from packageName " + packageName);
			return;
		}

		for (Class<?> clazz : classSet) {
			for (Class<? extends Annotation> annotation : BEAN_ANNOTATION) {
				//如果类上面标记了定义的注解
				if (clazz.isAnnotationPresent(annotation)){
					//将本类作为键，目标类的实例作为值，存入Map
					beanMap.put(clazz, ClassUtil.newInstance(clazz, true));
				}
			}
		}
		System.out.println(beanMap);
		loaded = true;
	}
    
    /**
	 * 设置类的属性值
	 * @param field 成员变量
	 * @param target 类实例
	 * @param value 成员变量的值
	 * @param accessible 是否允许设置私有属性
	 */
	public static void setField(Field field, Object target, Object value, boolean accessible){
		field.setAccessible(accessible);
		try {
			field.set(target, value);
		} catch (IllegalAccessException e) {
			log.error("setField error", e);
			throw new RuntimeException(e);
		}
	}
}

```

编写测试类测试BeanContainer：

测试之前要记得在com.zjw包下面的impl、controller下的方法分别加上自定义的注解@Service、@Controller，我这里总数为2

```java
public class BeanContainerTest {
	private static BeanContainer beanContainer;

	@BeforeAll
	static void init(){
		beanContainer = BeanContainer.getInstance();
	}

	@Test
	public void loadBeansTest(){
		Assertions.assertEquals(false, beanContainer.isLoaded());
		beanContainer.loadBeans("com.zjw");
		Assertions.assertEquals(2, beanContainer.getBeanSize()); // 我只加了两处
		Assertions.assertEquals(true, beanContainer.isLoaded());
	}
}
```

结果编译成功，说明加了自定义注解的类已经被成功放入到了Map中，打印一下beanMap看看。

![image-20221102160914939](https://zjw-note-images.oss-cn-shanghai.aliyuncs.com/img/MySelfIoC/202211041600688.png)



##### 容器的操作方式

涉及到容器的增删改查

- 增加、删除操作
- 根据Class获取对应实例
- 获取所有的Class和实例
- 通过注解来获取被注解标注的Class
- 通过超类获取对应的子类Class
- 获取容器载体保存Class的数量

下面的方法都写在BeanContainer类下

```java
/**
	 * 添加bean
	 * @param clazz
	 * @param bean
	 * @return
	 */
	public Object addBeans(Class<?> clazz, Object bean){
		return beanMap.put(clazz, bean);
	}

	/**
	 * 删除bean
	 * @param clazz
	 * @return
	 */
	public Object removeBeans(Class<?> clazz){
		return beanMap.remove(clazz);
	}

	/**
	 * 获取bean实例
	 * @param clazz
	 * @return
	 */
	public Object getBean(Class<?> clazz){
		return beanMap.get(clazz);
	}

	/**
	 * 获取容器管理的所有Class对象集合
	 * @return
	 */
	public Set<Class<?>> getClasses(){
		return beanMap.keySet();
	}

	/**
	 * 获取所有的bean集合
	 * @return
	 */
	public Set<Object> getBeans(){
		return new HashSet<>(beanMap.values());
	}

	/**
	 * 根据注解筛选出Bean的Class集合
	 * @param annotation
	 * @return
	 */
	public Set<Class<?>> getClassesByAnnotation(Class<? extends Annotation> annotation){
		//1.获取beanMap的所有class对象
		Set<Class<?>> keySet = getClasses();
		if (ValidationUtil.isEmpty(keySet)){
			log.warn("nothing in beanMap");
			return null;
		}
		//2.通过注解筛选被注解标记的class对象，并添加到classSet中
		Set<Class<?>> classSet = new HashSet<>();
		for (Class<?> clazz : keySet) {
			//类是否有相关的注解标记
			if (clazz.isAnnotationPresent(annotation)){
				classSet.add(clazz);
			}
		}
		return classSet.size() > 0 ? classSet : null;
	}

	/**
	 * 通过接口或者父类获取实现类或者子类的Class集合，不包括其本身
	 * @param interfaceOrClass
	 * @return
	 */
	public Set<Class<?>> getClassesBySuper(Class<?> interfaceOrClass){
		//1.获取beanMap的所有class对象
		Set<Class<?>> keySet = getClasses();
		if (ValidationUtil.isEmpty(keySet)){
			log.warn("nothing in beanMap");
			return null;
		}
		//2.判断keySet中的元素是否是传入的接口或者类的子类，是的话添加到classSet中
		Set<Class<?>> classSet = new HashSet<>();
		for (Class<?> clazz : keySet) {
			//判断keySet中的元素是否是传入的接口或者类的子类
			if (interfaceOrClass.isAssignableFrom(clazz) && !clazz.equals(interfaceOrClass)){
				classSet.add(clazz);
			}
		}
		return classSet.size() > 0 ? classSet : null;
	}
```

**测试某些方法：**

```java
//指定测试类的执行顺序
@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
public class BeanContainerTest {
	private static BeanContainer beanContainer;

	@BeforeAll
	static void init(){
		beanContainer = BeanContainer.getInstance();
	}

	@DisplayName("测试容器：loadBeansTest")
	@Test
	@Order(1) //最先执行,表示要先加载容器,获取bean，才能执行后面的getBean等方法。
	public void loadBeansTest(){
		Assertions.assertEquals(false, beanContainer.isLoaded());
		beanContainer.loadBeans("com.zjw");
		Assertions.assertEquals(2, beanContainer.getBeanSize());
		Assertions.assertEquals(true, beanContainer.isLoaded());
	}

	@DisplayName("根据类Class获取实例：getBeanTest")
	@Test
	@Order(2)
	public void getBeanTest(){
		//可以获取
		UserController userController = (UserController) beanContainer.getBean(UserController.class);
		Assertions.assertEquals(true, userController instanceof UserController);

		//不可以获取
		User user = (User) beanContainer.getBean(User.class);
		Assertions.assertEquals(null, user);
	}

	@DisplayName("根据注解类型获取对应的实例：getClassesByAnnotationTest")
	@Test
	@Order(3)
	public void getClassesByAnnotationTest(){
		Assertions.assertEquals(true, beanContainer.isLoaded());
		Assertions.assertEquals(1, beanContainer.getClassesByAnnotation(Controller.class).size());
	}

	@DisplayName("根据接口获取实现类：getClassesBySuperTest")
	@Test
	@Order(4)
	public void getClassesBySuperTest(){
		Assertions.assertEquals(true, beanContainer.isLoaded());
		Assertions.assertEquals(true, beanContainer.getClassesBySuper(UserService.class).contains(UserServiceImpl.class));
	}
}
```

![image-20221102172418850](https://zjw-note-images.oss-cn-shanghai.aliyuncs.com/img/MySelfIoC/202211041559274.png)

### 依赖注入

目前容器里面管理的Bean实例仍可能是不完备的

- 实例里面某些必须的成员变量还没有被创建出来

例如，需要添加类似@Autowired的注解将其注入。

<img src="https://zjw-note-images.oss-cn-shanghai.aliyuncs.com/img/MySelfIoC/202211041603678.png" alt="202211041600589" style="zoom:67%;" />

**实现思路：**

- 定义相关的注解标签
- 实现创建被注解标记的成员变量实例，并将其注入到成员变量里
- 依赖注入的使用

#### 定义相关的注解标签

```java
package org.IoC.inject.annotation;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * 目前支持成员变量注入
 */
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Autowired {
	//用来处理一个接口有多个是实现类的情况,可以给实现类定义一个value区分
	String value() default "";
}

```

#### 实现依赖注入

<img src="https://zjw-note-images.oss-cn-shanghai.aliyuncs.com/img/MySelfIoC/202211041604539.png" alt="202211041449933" style="zoom:67%;" />

**思路（以上图为例）：**

1. 首先要获取单例bean容器`BeanContainer`，目的是将@Controller标记的UserController加载到容器中。
2. 再通过容器获取UserController的Class对象，取出所有的属性（userService、name）。
3. 再筛选出被@Autowired标记的属性（userService）。
4. 获取成员变量的类型。获取这些成员变量的类型在容器里对应的实例（这里UserServiceImpl被@Service标记的）。
5. 通过反射将对应的成员变量实例注入到成员变量所在类的实例里。

```java
package org.IoC.inject;

import lombok.extern.slf4j.Slf4j;
import org.IoC.core.BeanContainer;
import org.IoC.inject.annotation.Autowired;
import org.IoC.utils.ClassUtil;
import org.IoC.utils.ValidationUtil;

import java.lang.reflect.Field;
import java.util.Arrays;
import java.util.Set;

@Slf4j
public class DependencyInjector {
	/**
	 * bean容器 单例
	 */
	private BeanContainer beanContainer;

	public DependencyInjector(){
		beanContainer = BeanContainer.getInstance();
	}

	/**
	 * 执行IoC
	 */
	public void doIoC(){
		if (ValidationUtil.isEmpty(beanContainer.getClasses())){
			log.warn("empty classSet in BeanContainer");
			return;
		}
		// 1.遍历Bean容器中所有的类对象
		for (Class<?> clazz : beanContainer.getClasses()) {
			// 2.遍历Class对象的所有成员变量
			Field[] fields = clazz.getDeclaredFields();
			//如果为空，则遍历下一个class
			if (ValidationUtil.isEmpty(fields)){
				continue;
			}
			System.out.println("获取到的成员变量：" + Arrays.toString(fields));
			for (Field field : fields) {
				// 3.找出被Autowired标记的成员变量
				if (field.isAnnotationPresent(Autowired.class)){
					System.out.println("筛选出被Autowire标记的成员变量：" + field);
					Autowired autowired = field.getAnnotation(Autowired.class);
					String autowiredValue = autowired.value();
					// 4.获取这些成员变量的类型
					Class<?> fieldClass = field.getType();
					// 5.获取这些成员变量的类型在容器里对应的实例
					Object fieldValue = getFieldInstance(fieldClass, autowiredValue);
					if (fieldValue == null){
						throw new RuntimeException("unable to inject relevant type, target fieldClass is: "
								+ fieldClass.getName() + autowiredValue);
					}else {
						// 6.通过反射将对应的成员变量实例注入到成员变量所在类的实例里
						Object targetBean = beanContainer.getBean(clazz);
						ClassUtil.setField(field, targetBean, fieldValue, true);
					}
				}
			}
		}
	}

	/**
	 * 根据Class在beanContainer里获取其实例或者实现类
	 * @param fieldClass
	 * @param autowiredValue
	 * @return
	 */
	private Object getFieldInstance(Class<?> fieldClass, String autowiredValue) {
		Object fieldValue = beanContainer.getBean(fieldClass);
		//直接可以获取到，比如@Contoller标记的
		if (fieldValue != null){
			return fieldValue;
		}else {
			//如果是接口，那么你要获取他的是实现类@Service
			Class<?> implementedClass = getImplementClass(fieldClass, autowiredValue);
			if (implementedClass != null){
				return beanContainer.getBean(implementedClass);
			}else {
				return null;
			}
		}
	}

	private Class<?> getImplementClass(Class<?> fieldClass, String autowiredValue) {
		Set<Class<?>> classSet = beanContainer.getClassesBySuper(fieldClass);
		if (!ValidationUtil.isEmpty(classSet)){
			//默认value为空
			if (ValidationUtil.isEmpty(autowiredValue)){
				//只有一个是实现类
				if (classSet.size() == 1){
					return classSet.iterator().next();
				}else {
					//如果多于两个实现类并且没有指定value，则抛出异常
					throw new RuntimeException("multiple implement class for " + fieldClass.getName());
				}
			}else {
				for (Class<?> clazz : classSet) {
                    //找到匹配的value
					if (autowiredValue.equals(clazz.getSimpleName())){
						return clazz;
					}
				}
			}
		}
		return null;
	}
}
```

**测试一下：**

**1、UserService只有一个实现类UserServiceImpl。**

```java
package org.IoC.inject;

import com.zjw.controller.UserController;
import org.IoC.core.BeanContainer;
import org.junit.jupiter.api.Assertions;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;

public class DependencyInjectorTest {

	@DisplayName("依赖注入doIoC")
	@Test
	public void doIoCTest(){
		BeanContainer beanContainer = BeanContainer.getInstance();
		beanContainer.loadBeans("com.zjw");
		Assertions.assertEquals(true, beanContainer.isLoaded());
		UserController userController = (UserController) beanContainer.getBean(UserController.class);
		Assertions.assertEquals(true, userController instanceof UserController);
		//UserController要加上@Getter注解，以获取注入的service实例
		Assertions.assertEquals(null, userController.getUserService());
		new DependencyInjector().doIoC();
		Assertions.assertNotEquals(null, userController.getUserService());
	}
}
```

![image-20221104151225991](https://zjw-note-images.oss-cn-shanghai.aliyuncs.com/img/MySelfIoC/202211041557805.png)

**2、UserService只有一个实现类UserServiceImpl、UserServiceImpl2。**

![image-20221104151327098](https://zjw-note-images.oss-cn-shanghai.aliyuncs.com/img/MySelfIoC/202211041557984.png)

给@Autowired指定value值

```java
@Autowired(value = "UserServiceImpl")
private UserService userService;
```

![image-20221104151527252](https://zjw-note-images.oss-cn-shanghai.aliyuncs.com/img/MySelfIoC/202211041557562.png)



## 总结

![1667547977897](https://zjw-note-images.oss-cn-shanghai.aliyuncs.com/img/MySelfIoC/202211041557293.jpeg)
