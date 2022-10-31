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

### 1-2 Spring源码下载编译

#### 1、源码下载配置

1、下载源码：https://spring.io/

推荐下载**xxx.RELEASE版本**（5.2.0.RELEASE）

<img src="https://raw.githubusercontent.com/z-junwei/images/mian/img/Spring/202210311510674.png" alt="202210311435368" style="zoom:67%;" />

2、配置gradle阿里云镜像下载

打开项目文件中`build.gradle`文件，添加（2处）：

```gradle
repositories {
        maven { url 'http://maven.aliyun.com/nexus/content/groups/public/' }
        maven{ url 'http://maven.aliyun.com/nexus/content/repositories/jcenter'}
}
```

<img src="https://raw.githubusercontent.com/z-junwei/images/mian/img/Spring/202210311514165.png" alt="image-20221031151429137" style="zoom:67%;" />

<img src="https://raw.githubusercontent.com/z-junwei/images/mian/img/Spring/202210311514765.png" alt="image-20221031151443738" style="zoom:67%;" />

3、预编译

运行之前，最好先找到C盘 .gradle 文件下的 gradle.bat 下载依赖，确保编译成功。

<img src="https://raw.githubusercontent.com/z-junwei/images/mian/img/Spring/202210311521889.png" alt="image-20221031152104862" style="zoom:67%;" />

4、导入IDEA

<img src="https://raw.githubusercontent.com/z-junwei/images/mian/img/Spring/202210311521012.png" alt="image-20221031152131981" style="zoom:67%;" />

<img src="https://raw.githubusercontent.com/z-junwei/images/mian/img/Spring/202210311521671.png" alt="image-20221031152157642" style="zoom: 67%;" />

5、Upload  `spring-aspects` 模块

#### 2、新建模块测试

新建一个springdemo模块。

<img src="https://raw.githubusercontent.com/z-junwei/images/mian/img/Spring/202210311551692.png" alt="image-20221031155155665" style="zoom:67%;" />

1、接口

```java
package com.zjw.service;

public interface WelcomeService {
	String SayHello(String name);
}

```

2、实现类

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

3、xml

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

### 1-3 自研框架搭建

<img src="https://raw.githubusercontent.com/z-junwei/images/mian/img/Spring/202210311625192.png" alt="image-20221031162527154" style="zoom:67%;" />

1、新建simpleframework模块，选择使用maven创建

2、pom文件

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
		 xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
		 xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>

	<groupId>org.zjw</groupId>
	<artifactId>simpleframework</artifactId>
	<version>1.0-SNAPSHOT</version>
	<packaging>war</packaging>

	<properties>
		<maven.compiler.source>8</maven.compiler.source>
		<maven.compiler.target>8</maven.compiler.target>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
	</properties>

	<dependencies>
		<dependency>
			<groupId>javax.servlet.jsp</groupId>
			<artifactId>javax.servlet.jsp-api</artifactId>
			<version>2.3.3</version>
<!--			加上，不然有冲突-->
			<scope>provided</scope>
		</dependency>
		<dependency>
			<groupId>javax.servlet</groupId>
			<artifactId>javax.servlet-api</artifactId>
			<version>4.0.1</version>
			<scope>provided</scope>
		</dependency>
	</dependencies>

	<build>
		<finalName>simpleframework</finalName>
		<pluginManagement>
			<plugins>
				<plugin>
					<groupId>org.apache.maven.plugins</groupId>
					<artifactId>maven-compiler-plugin</artifactId>
					<version>3.8.1</version>
				</plugin>
<!--				引入tomcat插件-->
				<plugin>
					<groupId>org.apache.tomcat.maven</groupId>
					<artifactId>tomcat7-maven-plugin</artifactId>
					<version>2.2</version>
					<configuration>
						<path>/${project.artifactId}</path>
					</configuration>
				</plugin>
			</plugins>
		</pluginManagement>
	</build>

</project>
```

3、编写HelloServlet

```java
import javax.servlet.ServletException;
import javax.servlet.annotation.WebServlet;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;

@WebServlet("/hello")
public class HelloServlet extends HttpServlet {
	@Override
	protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
		String name = "My framework";
		req.setAttribute("name", name);
		req.getRequestDispatcher("/WEB-INF/jsp/hello.jsp").forward(req, resp);
	}
}
```

4、编写jsp

在main目录下新建WEB-INF/jsp/hello.jsp

```jsp
<%@ page pageEncoding="UTF-8"%>
<html>
    <head>
        <title>Hello</title>
    </head>
    <body>
        <h1>Hello!</h1>
        <h2>太流弊了，${name}</h2>
    </body>
</html>

```

5、运行

浏览器访问http://localhost:8080/simpleframework/hello

![image-20221031170955242](https://raw.githubusercontent.com/z-junwei/images/mian/img/Spring/202210311709293.png)

**jsp运行原理**

<img src="https://raw.githubusercontent.com/z-junwei/images/mian/img/Spring/202210311712376.png" alt="image-20221031171227298" style="zoom: 50%;" />
