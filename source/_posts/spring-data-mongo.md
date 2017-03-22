---
title: spring-data-mongo
date: 2017-03-23 02:01:39
tags:
---
# 什么是SpringData

> Spring Data’s mission is to provide a familiar and consistent, Spring-based programming model for data access while still retaining the special traits of the underlying data store. 
意思是Spring 提供一致的数据访问操作，不管你用的什么持久化技术，关系型数据库，NoSQL等，Spring Data的编程接口都类似，提供相似的一致的，基于Spring的数据访问技术。

Spring Data Mongo 是基于Spring Data的扩展，提供了对文档型数据库MongoDB的支持。
本问将讲解Spring Data Mongo的配置
# 一，引入Spring Data Mongo 依赖
``` xml
<dependency>
    <groupId>org.springframework.data</groupId>
    <artifactId>spring-data-mongodb</artifactId>
    <version>1.9.5.RELEASE</version>
</dependency>
```
# 二，Mongo连接配置
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:mongo="http://www.springframework.org/schema/data/mongo"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/data/mongo
        http://www.springframework.org/schema/data/mongo/spring-mongo.xsd">
    <!-- MongoDB factory  -->
    <mongo:db-factory id="mongoDbFactory" host="10.0.0.2" port="27017" dbname="test" />

    <!-- MongoTemplate -->
    <mongo:template db-factory-ref="mongoDbFactory"/>
    <!-- 自动扫描该包下的repository，自动生成实现了类，根据接口的函数名生成相应的操作  -->
    <mongo:repositories base-package="me.zhengkun.springdatamogotest.repository"/>
</beans>
```
# 三，创建entity和repository
User实体
```java
package me.zhengkun.springdatamogotest.entity;

import org.springframework.data.annotation.Id;

/**
 * Created by unicorn on 2016/11/17.
 */
public class User {
    /**
     * 标识ID信息
     * 插入数据时如果没有提供，MongoDB会自动生成
     */
    @Id
    private String id;
    private String name;
    private String password;

    public User() {

    }

    public User(String name, String password) {
        this.name = name;
        this.password = password;
    }
    // 省略get set方法
}
```
UserRepository
只需要继承MongoRepository， 无需实现接口，Spring Data会自动根据函数名生成响应的操作指令，然后通过MongoTemplate执行
``` java
package me.zhengkun.springdatamogotest.repository;

import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.mongodb.repository.MongoRepository;
import org.springframework.data.mongodb.repository.Query;

import java.util.List;

import me.zhengkun.springdatamogotest.entity.User;

/**
 * 继承MongoRepository接口，其提供了常用的CRUD操作
 * 只需要定义接口，不需要实现该接口，Spring Data Mongo会自动生成实现该接口的代理类
 * 并且根据接口的方法名进行相应的操作
 *
 * MongoRepository泛型接口有两个参数，分别是实体类和主键类型
 * Created by unicorn on 2016/11/17.
 */
public interface UserRepository extends MongoRepository<User, String> {
    /**
     * 根据name查找
     * @param name name
     * @return user list
     */
    List<User> findByName(String name);

    /**
     * 同时根据name和password查找
     * @param name name
     * @param password password
     * @return user list
     */
    List<User> findByNameAndPassword(String name, String password);

    /**
     * 分页查找
     * @param name name
     * @param pageable 分页请求
     * @return 查找出的当前页的结果
     */
    Page<User> findByName(String name, Pageable pageable);

    /**
     * 自定义查找
     * 查找姓名为name1 或者 name2 的user
     * @param name1 name1
     * @param name2 name2
     * @return user list
     */
    @Query("{'$or':[{'name': ?0}, {'name': ?1}]}")
    List<User> customFind(String name1, String name2);
}
```
# 三，测试UserRepository
使用了Junit和Spring-Test，需要请引入相关的依赖
```xml
<dependency>
    <groupId>junit</groupId>
    <artifactId>junit</artifactId>
    <version>4.12</version>
    <scope>test</scope>
</dependency>
<dependency>
   <groupId>org.springframework</groupId>
   <artifactId>spring-test</artifactId>
   <version>4.2.8.RELEASE</version>
   <scope>test</scope>
</dependency>
```

UserRepositoryTest测试代码
```java
package me.zhengkun.springdatamongotest.repository;

import org.junit.After;
import org.junit.Before;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageRequest;
import org.springframework.test.context.ContextConfiguration;
import org.springframework.test.context.junit4.SpringJUnit4ClassRunner;

import java.util.List;

import javax.annotation.Resource;

import me.zhengkun.springdatamogotest.entity.User;
import me.zhengkun.springdatamogotest.repository.UserRepository;

/**
 * Created by unicorn on 2016/11/17.
 */
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration("classpath:applicationContext.xml")
public class UserRepositoryTest {
    @Resource
    private UserRepository userRepository;

    @Before
    public void before() {
        // 删除原有的数据
        userRepository.deleteAll();
        User user = new User("Jack", "YouJumpIJump");
        userRepository.save(user);

        User user1 = new User("Rose", "JackILoveU");
        userRepository.save(user1);

    }

    @After
    public void after() {
        // 测试完毕，删除数据
        userRepository.deleteAll();
    }

    @Test
    public void testFindByName() {
        List<User> users = userRepository.findByName("Jack");
        System.out.println(users);
    }

    @Test
    public void testFindByNameAndPassword() {
        List<User> users = userRepository.findByNameAndPassword("Rose", "JackILoveU");
        System.out.println(users);
    }

    @Test
    public void testCustomFind() {
        List<User> users = userRepository.customFind("Jack", "Rose");
        System.out.println(users);
    }

    @Test
    public void testFindByPage() {
        // 查找第0页，共1条
        Page<User> page = userRepository.findByName("Jack", new PageRequest(0, 1));
        System.out.println(page.getContent());
    }

}
```
# 四，总结
Spring Data为MongoDB的操作提供了一致的Repository接口，可以自动根据函数名生成查询方法，简化了操作，大大提高了开发效率。
示例代码地址https://github.com/hqs0417/examples/tree/master/spring-data-mongo-test