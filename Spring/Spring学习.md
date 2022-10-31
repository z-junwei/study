# Spring

## 一 环境准备

### 1-1 Spring模块梳理



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

1、下载源码：https://spring.io/

推荐下载**xxx.RELEASE版本**（5.2.0.RELEASE）

![202210311435368](https://raw.githubusercontent.com/z-junwei/images/mian/img/Spring/202210311510674.png)

2、配置gradle阿里云镜像下载

打开项目文件中`build.gradle`文件，添加（2处）：

```gradle
repositories {
        maven { url 'http://maven.aliyun.com/nexus/content/groups/public/' }
        maven{ url 'http://maven.aliyun.com/nexus/content/repositories/jcenter'}
}
```





