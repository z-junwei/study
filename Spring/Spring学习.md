# 笔记-剑指Java自研框架，决胜Spring源码

学习课程：

![image-20221101170533811](https://raw.githubusercontent.com/z-junwei/images/mian/img/Spring/202211011705840.png)

## 一 自研IoC框架

### 1 实现思路

**一个框架最基本的功能：**

- 解析配置
- 定位与注册对象
- 注入对象
- 提供通用的工具类

**Io容器的实现：**

`创建注解 -> 提取标记对象 -> 实现容器 -> 依赖注入`

### 2 实现

#### 1 创建注解

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

```Java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface Repository {
}
```

```Java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface Component {
}

```

#### 2 提取标记对象

实现思路：

- 指定范围，获取范围内的所有类
- 遍历所有类，获取被注解标记的类并加载进容器里

```java
package com.zjw.myioc.core.utils;

import lombok.extern.slf4j.Slf4j;

import java.io.File;
import java.io.FileFilter;
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
				//如：D:\study\SpringSourceCode\spring-framework-5.2.0.RELEASE\simpleframework\src\main\java\com\zjw\entity\Student.java
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

	public static void main(String[] args) {
		File packageDirectory = new File("D:\\study\\SpringSourceCode\\spring-framework-5.2.0.RELEASE\\simpleframework\\target\\classes\\com\\zjw\\entity");
		System.out.println(packageDirectory.isDirectory());
	}
}

```

编写测试类测试：

```java
package com.zjw.myioc.core.utils;

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
		Assertions.assertEquals(3, classSet.size());
	}
}


```

测试结果：

![image-20221101170040640](https://raw.githubusercontent.com/z-junwei/images/mian/img/Spring/202211011700683.png)



