#Spring Authorization Manager

为Api服务端添加简单的Token鉴权功能，基于Spring MVC

###功能简述

1. 对每个请求进行身份验证，如果身份验证不通过则返回错误`401`
2. 通过鉴权信息获得当前登录的用户

###使用方法

这个库的所有依赖库，我相信大部分库已经存在于你的项目中了：

```
<dependencies>
  <dependency>
  <groupId>com.scienjus</groupId>
  <artifactId>spring-authorization-manager</artifactId>
  <version>1.0-SNAPSHOT</version>
  </dependency>
  <!--Spring MVC依赖-->
  <dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-webmvc</artifactId>
    <version>4.2.2.RELEASE</version>
  </dependency>
  <dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context</artifactId>
    <version>4.2.2.RELEASE</version>
  </dependency>
  <dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-core</artifactId>
    <version>2.6.3</version>
  </dependency>
  <dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <version>2.6.3</version>
  </dependency>
  <dependency>
    <groupId>javax.servlet</groupId>
    <artifactId>javax.servlet-api</artifactId>
    <version>3.1.0</version>
  </dependency>
  <!--Redis依赖，只有在使用RedisTokenManager时才需要-->
  <dependency>
    <groupId>org.springframework.data</groupId>
    <artifactId>spring-data-redis</artifactId>
    <version>1.6.0.RELEASE</version>
  </dependency>
  <dependency>
    <groupId>redis.clients</groupId>
    <artifactId>jedis</artifactId>
    <version>2.7.3</version>
  </dependency>
  <!--数据库依赖，只有在使用DBTokenManager时才需要-->
  <dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>5.1.37</version>
  </dependency>
  <dependency>
    <groupId>commons-dbcp</groupId>
    <artifactId>commons-dbcp</artifactId>
    <version>1.4</version>
  </dependency>
  <dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-jdbc</artifactId>
    <version>4.2.2.RELEASE</version>
  </dependency>
</dependencies>
```

**使用Redis存储Token**

将Jedis客户端注入到`RedisTokenManager`：

```
<!--Redis客户端-->
<bean id="redis" class="org.springframework.data.redis.connection.jedis.JedisConnectionFactory" p:hostName="127.0.0.1" p:port="6379" p:password="redis"/>

<!--管理验证信息的bean-->
<bean id="tokenManager" class="com.scienjus.authorization.manager.impl.RedisTokenManager">
       <!--Token失效时间-->
       <property name="tokenExpireSeconds" value="3600" />
       <!--Redis客户端-->
       <property name="redis" ref="redis" />
</bean>
```

**使用Database存储Token**

只需要将`RedisTokenManager`替换成`DBTokenManager`，并将数据源注入进去：

```
<!--数据源-->
<bean id="dataSource" class="org.apache.commons.dbcp.BasicDataSource" destroy-method="close">
       <property name="driverClassName" value="com.mysql.jdbc.Driver"/>
       <property name="url" value="jdbc:mysql://127.0.0.1:3306/demo"/>
       <property name="username" value="root"/>
       <property name="password" value="root"/>
</bean>

<!--管理验证信息的bean-->
<bean id="tokenManager" class="com.scienjus.authorization.manager.impl.DBTokenManager">
       <!--Token失效时间-->
       <property name="tokenExpireSeconds" value="3600" />
       <!--数据源-->
       <property name="dataSource" ref="dataSource" />
       <!--存储验证信息的表名-->
       <property name="tableName" value="users" />
       <!--存储Key的字段名-->
       <property name="keyColumnName" value="username" />
       <!--存储Token的字段名-->
       <property name="tokenColumnName" value="token" />
       <!--存储过期时间的字段名-->
       <property name="expireAtColumnName" value="expire_at" />
</bean>
```

**配置身份验证的拦截器**

将配置好的`TokenManager`注入到`AuthorizationInterceptor`中：

```
<mvc:interceptors>
       <!--身份验证的拦截器-->
       <bean id="authorizationInterceptor" class="com.scienjus.authorization.interceptor.AuthorizationInterceptor">
              <!--验证信息存储的Http头-->
              <property name="httpHeaderName" value="authorization" />
              <!--验证信息通用前缀，例如Bearer-->
              <property name="httpHeaderPrefix" value="" />
              <!--验证失败时的错误信息-->
              <property name="unauthorizedErrorMessage" value="令牌失效，请重新登录" />
              <!--管理验证信息的bean-->
              <property name="manager" ref="tokenManager" />
       </bean>
</mvc:interceptors>
```

接下来只需要对需要身份验证的方法加上`@Authorization`注解即可，例如：

```
@RestController
@RequestMapping("/home")
public class TokenController {

    @RequestMapping(method = RequestMethod.GET)
    @Authorization
    public ResponseEntity<String> home() {
        return new ResponseEntity<>("Hello World", HttpStatus.OK);
    }

}
```
**配置获得当前登录用户的解析器**

首先需要实现`UserModelRepository`接口的`getCurrentUser`方法，可以通过Key得到对应的用户对象，然后配置一个解析器，并将其注入到`CurrentUserMethodArgumentResolver`：

```
<mvc:annotation-driven>
       <mvc:argument-resolvers>
              <!--配置注入登录用户的解析器-->
              <bean id="currentUserMethodArgumentResolver" class="com.scienjus.authorization.resolvers.CurrentUserMethodArgumentResolver">
                     <!--需要解析的用户类-->
                     <property name="userModelClass" value="com.scienjus.domain.User" />
                     <!--查询用户的bean-->
                     <property name="userModelRepository" ref="userRepository" />
              </bean>
       </mvc:argument-resolvers>
</mvc:annotation-driven>

<!--通过Key获得对应用户的bean-->
<bean id="userRepository" class="com.scienjus.repository.UserRepository" />
```

然后只需要在方法的参数上添加一个用户对象，并加上`@CurrentUser`注解，例如：

```
@RestController
@RequestMapping("/home")
public class TokenController {

    @RequestMapping(method = RequestMethod.GET)
    @Authorization
    public ResponseEntity<String> home(@CurrentUser user) {
        return new ResponseEntity<>("Hello " + user.getUsername(), HttpStatus.OK);
    }

}
```

需要注意的是，拥有`@CurrentUser`参数的方法，可以没有`@Authorization`注解，此时如果请求未登录，该参数会为`null`。

但是如果想要使用`CurrentUserMethodArgumentResolver`则必须配置`AuthorizationInterceptor`。