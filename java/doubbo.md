### springboot+dubbo （没有集成database）
1. pom加入dubbo和zookeeper的依赖
    ```
    <!-- dubbo -->
    <dependency>
        <groupId>com.alibaba.boot</groupId>
        <artifactId>dubbo-spring-boot-starter</artifactId>
        <version>0.2.0</version>
    </dependency>

    <dependency>
        <groupId>org.apache.zookeeper</groupId>
        <artifactId>zookeeper</artifactId>
        <version>3.4.12</version>
    </dependency>
    ```
2. 创建provider
创建xml配置dubbo注册者的相关信息
```
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd http://code.alibabatech.com/schema/dubbo
http://code.alibabatech.com/schema/dubbo/dubbo.xsd">

    <!-- 应用配置，不要与提供方相同 -->
    <dubbo:application name="springboot-dubbo-provider"/>

    <!-- 注册中心配置，使用zookeeper注册中心暴露服务地址 -->
    <dubbo:registry address="zookeeper://49.235.209.207:2181" timeout="60000" />

    <!--关闭服务消费方所有服务的启动检查。dubbo缺省会在启动时检查依赖的服务是否可用，不可用时会抛出异常，阻止Spring初始化完成。-->
    <dubbo:consumer check="false" />

    <!-- 使用注解方式暴露接口，会自动扫描package下所有包中dubbo相关的注解，这样就不用在xml中再针对每个服务接口配置dubbo:service interface-->
    <dubbo:annotation package="com.dubbo.demo.service"/>
    <!--<dubbo:service interface="com.practice.springboot.dubbo.provider.SayHelloImpl" ref="SayHelloImpl"/>-->
</beans>
```
3. consumer配置
```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd http://code.alibabatech.com/schema/dubbo
http://code.alibabatech.com/schema/dubbo/dubbo.xsd">

    <!-- 应用配置，不要与提供方相同 -->
    <dubbo:application name="springboot-dubbo-consumer"/>

    <!-- 注册中心配置，使用zookeeper注册中心暴露服务地址 -->
    <dubbo:registry address="zookeeper://49.235.209.207:2181" timeout="60000" />

    <!-- 使用注解方式创建远程服务代理-->
<!--    <dubbo:annotation package="com.dubbo.demo11.service.IProviderDemoService"/>-->
    <!--声明服务引用，与服务声明接口类型一致-->
    <dubbo:reference interface="com.dubbo.demo.service.IDemoService" id="IDemoService"/>
</beans>
```
### yml 配置文件
    ```
    spring:
      dubbo:
        application:            #应用配置，用于配置当前应用信息，不管该应用是提供者还是消费者。
          name: Provide
        registry:                 #注册中心配置，用于配置连接注册中心相关信息。
          address: zookeeper://127.0.0.1:2181
        protocol:     #协议配置，用于配置提供服务的协议信息，协议由提供方指定，消费方被动接受。
          name: dubbo
          port: 20880
        scan: com.dubboProvide.dubboProvide.service  #服务暴露与发现消费所在的package
    ```
