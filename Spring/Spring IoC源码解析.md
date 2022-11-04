## 友情提示

- 不要在意版本，核心思想不变
- 抓住骨架进行学习，覆盖所有细节难度太大
  - 解析配置
  - 定位与注册对象
  - 注入对象

## Xml与Annotation

先演示一下两种方式获取bean。

```java
package com.zjw.service;

public interface WelcomeService {
	String sayHello(String name);
}
```

```java
package com.zjw.service.impl;

import com.zjw.service.WelcomeService;
import org.springframework.stereotype.Service;

//@Service
public class WelcomeServiceImpl implements WelcomeService {
	@Override
	public String sayHello(String name) {
		System.out.println("hello, " + name);
		return "success";
	}
}
```

### 基于Xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	   xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd">

	<bean id="welcomeService" class="com.zjw.service.impl.WelcomeServiceImpl"></bean>

</beans>
```

```java
package com.zjw;

import com.zjw.service.WelcomeService;
import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.support.FileSystemXmlApplicationContext;

//@Configuration
//@ComponentScan("com.zjw")
public class Entrance {
	public static void main(String[] args) {
		System.out.println("hello world");
		String xmlPath = "D:\\study\\SpringSourceCode\\spring-framework-5.2.0.RELEASE\\springdemo\\src\\main\\resources\\spring\\spring-config.xml";
		ApplicationContext applicationContext = new FileSystemXmlApplicationContext(xmlPath);
		WelcomeService welcomeService = (WelcomeService) applicationContext.getBean("welcomeService");
		welcomeService.sayHello("spring");
	}
}
```

<img src="https://zjw-note-images.oss-cn-shanghai.aliyuncs.com/img/IoCAnalysis202211041639236.png" alt="image-20221104163904204" style="zoom:67%;" />

### 基于Annotation

把上面加了注解的类的注释解开。

```java
package com.zjw;

import com.zjw.service.WelcomeService;
import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.support.FileSystemXmlApplicationContext;

@Configuration
@ComponentScan("com.zjw")//扫描哪个包下，和自研类似
public class Entrance {
	public static void main(String[] args) {
		AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(Entrance.class);
		String[] beanDefinitionNames = applicationContext.getBeanDefinitionNames();
		for (String beanDefinitionName : beanDefinitionNames) {
			System.out.println(beanDefinitionName);
		}
		WelcomeService welcomeService = (WelcomeService) applicationContext.getBean("welcomeServiceImpl");
		welcomeService.sayHello("spring");
	}
}

```

<img src="https://zjw-note-images.oss-cn-shanghai.aliyuncs.com/img/IoCAnalysis202211041642538.png" alt="image-20221104164218507" style="zoom:67%;" />

在该包下加了注解的类都被加载到了bean容器中。



## Bean与BeanDefinition

**Bean是Spring的一等公民：**

- Bean的本质就是java对象，只是这个对象的声明周期由容器来管理
- 不需要为了创建Bean而在原来的java类上添加任何额外的限制
- 对java对象的控制方式体现在配置上

**根据配置，生成用来描述Bean的BeanDefinition，常用属性∶**

- 作用范围scope(@Scope)
- 懒加载lazy-init(@Lazy)：决定Bean实例是否延迟加载
- 首选primary(@Primary)：设置为true的bean会是优先的实现
- factorv-bean和factorv-method(@Confiauration和@Bean）

### 代码显示

```java
package com.zjw.entity;

public class User {
}
```

```java
package com.zjw.entity.factory;

import com.zjw.entity.User;

//静态工厂调用
public class StaticFactory {
	//静态的方法，返回User对象
	public static User getUser(){
		return new User();
	}
}
```

```java
package com.zjw.entity.factory;

import com.zjw.entity.User;
//实例工厂调用
public class UserFactory {
	//普通的方法，返回User对象
	//不能通过类名调用，需要通过对象调用
	public User getUser(){
		return new User();
	}
}
```

```java
package com.zjw.entity.factory;

import com.zjw.entity.User;
import org.springframework.beans.factory.FactoryBean;

public class UserFactoryBean implements FactoryBean<User> {
	@Override
	public User getObject() throws Exception {
		return new User();
	}

	@Override
	public Class<?> getObjectType() {
		return User.class;
	}
}
```

```java
public class Entrance {

	public static void main(String[] args) {

		//得到无参构造函数创建的对象:
		User user1a = (User) applicationContext.getBean("user1");
		User user1b = (User) applicationContext.getBean("user1");
		//得到静态工厂创建的对象：
		User user2a = (User) applicationContext.getBean("user2");
		User user2c = (User) applicationContext.getBean("user2");
		//得到实例工厂创建的对象：
		User user3a = (User) applicationContext.getBean("user3");
		User user3b = (User) applicationContext.getBean("user3");


		System.out.println("无参构造函数创建的对象:" + user1a);
		System.out.println("无参构造函数创建的对象:" + user1b);
		System.out.println("静态工厂创建的对象：" + user2a);
		System.out.println("静态工厂创建的对象：" + user2c);
		System.out.println("实例工厂创建的对象：" + user3a);
		System.out.println("实例工厂创建的对象：" + user3b);
	}
    
}
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	   xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd">

	<bean id="welcomeService" class="com.zjw.service.impl.WelcomeServiceImpl"/>
	<!-- 1.使用类的无参构造函数创建 -->
	<bean id="user1" class="com.zjw.entity.User" scope="singleton" lazy-init="true" primary="true"/>
	<!-- 2.使用静态工厂进行创建 -->
	<!-- class的值不是写User对象的全路径，而是写静态工厂的全路径 -->
	<!-- factory-method的值写要调用的方法 -->
	<bean id="user2" class="com.zjw.entity.factory.StaticFactory" factory-method="getUser" scope="singleton"/>
	<!-- 3.使用实例工厂进行创建 -->
	<!-- 需要先创建factoryBean对象，再通过factoryBean对象进行调用 -->
	<bean id="userFactory" class="com.zjw.entity.factory.UserFactory"/>
	<bean id="user3" factory-bean="userFactory" factory-method="getUser" scope="prototype" />

</beans>
```

![image-20221104171933387](https://zjw-note-images.oss-cn-shanghai.aliyuncs.com/img/IoCAnalysis202211041719374.png)

你会发现当scope="singleton"，对象相同；scope="prototype"对象不同。

### 容器初始化主要步骤

![image-20221104172550192](https://zjw-note-images.oss-cn-shanghai.aliyuncs.com/img/IoCAnalysis202211041725233.png)

- 解析配置
- 定位与注册对象