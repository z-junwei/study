# Spring

## 一 环境准备

### 1-1 Spring模块梳理

<img src="https://raw.githubusercontent.com/z-junwei/images/mian/img/Spring/202210311514251.png" alt="image-20221031151405194" style="zoom:67%;" />

**spring-core**

- 包含框架基本的核心工具类，其它组件要都要使用到这个包里的类
- 定义并提供资源的访问方式

**spring-beans**

Spring主要面向Bean编程（BOP）

- Bean的定义（BeanFactory ）
- Bean的解析
- Bean的创建

**spring-context**

- 为Spring提供运行时环境，保存对象的状态为Spring提供运行时环境，保存对象的状态
- 扩展了BeanFactory （ApplicationContext）

**spring-aop**

- 提供了面向切面的编程实现

### 1-2 Spring源码的下载和编译

#### 1、源码下载配置

**1、下载源码：**https://spring.io/

推荐下载**xxx.RELEASE版本**（5.2.0.RELEASE）

<img src="https://raw.githubusercontent.com/z-junwei/images/mian/img/Spring/202210311510674.png" alt="202210311435368" style="zoom:67%;" />

**2、配置gradle阿里云镜像下载**

打开项目文件中`build.gradle`文件，添加（2处）：

```gradle
repositories {
        maven { url 'http://maven.aliyun.com/nexus/content/groups/public/' }
        maven{ url 'http://maven.aliyun.com/nexus/content/repositories/jcenter'}
}
```

<img src="https://raw.githubusercontent.com/z-junwei/images/mian/img/Spring/202210311514165.png" alt="image-20221031151429137" style="zoom:67%;" />

<img src="https://raw.githubusercontent.com/z-junwei/images/mian/img/Spring/202210311514765.png" alt="image-20221031151443738" style="zoom:67%;" />

**3、预编译**

运行之前，最好先找到C盘 .gradle 文件下的 gradle.bat 下载依赖，确保编译成功。

<img src="https://raw.githubusercontent.com/z-junwei/images/mian/img/Spring/202210311521889.png" alt="image-20221031152104862" style="zoom:67%;" />

**4、导入IDEA**

<img src="https://raw.githubusercontent.com/z-junwei/images/mian/img/Spring/202210311521012.png" alt="image-20221031152131981" style="zoom:67%;" />

<img src="https://raw.githubusercontent.com/z-junwei/images/mian/img/Spring/202210311521671.png" alt="image-20221031152157642" style="zoom: 67%;" />

**5、Upload  `spring-aspects` 模块**

#### 2、新建模块

新建一个springdemo模块。

<img src="https://raw.githubusercontent.com/z-junwei/images/mian/img/Spring/202210311551692.png" alt="image-20221031155155665" style="zoom:67%;" />

```java
package com.zjw.service;

public interface WelcomeService {
	String SayHello(String name);
}

```

```java
package com.zjw.service.impl;

import com.zjw.service.WelcomeService;

public class WelcomeServiceImpl implements WelcomeService {
	@Override
	public String SayHello(String name) {
		System.out.println("hello, " + name);
		return "success";
	}
}
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	   xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd">

	<bean id="welcomeService" class="com.zjw.service.impl.WelcomeServiceImpl"></bean>

</beans>
```

打开本项目的build.gradle文件，添加：

```gradle
dependencies {
//    配置spring-context依赖，解析xml文件，生产bean并返回
    compile(project(":spring-context"))
    
    testImplementation 'org.junit.jupiter:junit-jupiter-api:5.7.0'
    testRuntimeOnly 'org.junit.jupiter:junit-jupiter-engine:5.7.0'
}
```

```java
public class Entrance {
	public static void main(String[] args) {
		System.out.println("hello world");
		String xmlPath = "D:\\study\\spring源码\\spring-framework-5.2.0.RELEASE\\springdemo\\src\\main\\resources\\spring\\spring-config.xml";
		ApplicationContext applicationContext = new FileSystemXmlApplicationContext(xmlPath);
		WelcomeService welcomeService = (WelcomeService) applicationContext.getBean("welcomeService");
		welcomeService.sayHello("spring");
	}
}
```

成功打印。
