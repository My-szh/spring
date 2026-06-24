/

# 									尚庭公寓项目知识点

## 7.2.2.1房间支付方式管理

#### 1、逻辑删除功能@TableLogic

由于数据库中所有表均采用逻辑删除策略，所以查询数据时均需要增加过滤条件is_deleted=0

逻辑删除功能只对Mybatis-Plus自动注入的sql起效，也就是说，对于手动在`Mapper.xml`文件配置的sql不会生效，需要单独考虑。

**实现逻辑删除两种方式**

（1）**@TableLogic注解** **(推荐)**

```java
@Data
public class BaseEntity {

    @Schema(description = "逻辑删除")
    @JsonIgnore
    @TableLogic
    @TableField("is_deleted")
    private Byte isDeleted;

}
```

（2）**application.yaml文件配置**

```yaml
mybatis-plus:
  global-config:
    db-config:
      logic-delete-field: flag # 全局逻辑删除的实体字段名(配置后可以忽略不配置步骤二)
      logic-delete-value: 1 # 逻辑已删除值(默认为 1)
      logic-not-delete-value: 0 # 逻辑未删除值(默认为 0)
```





#### 2、忽略特定字段@JsonIgnore（password等敏感字段）

通常情况下接口响应的Json对象中并不需要create_time、update_time、is_deleted等字段，这时只需在实体类中的相应字段添加@JsonIgnore注解，该字段就会在序列化时被忽略，不返回给前端。

（1）**@JsonIgnore**

（2）**@TableField(select=false)**

两者配合实现双重保护，确保敏感信息不会泄露

```java
	@Schema(description = "密码")
    @TableField(value = "password",select = false)  //查询数据库时不查询该字段
    @JsonIgnore  // 忽略json序列化，不将结果返回给前端
    private String password;
```





#### 3、自动填充配置@TableField`注解中的`fill`属性

保存或更新数据时，前端通常不会传入 isDeleted、createTime、updateTime 这三个字段，因此我们需要手动赋值。但是数据库中几乎每张表都有上述字段，所以手动去赋值就显得有些繁琐。为简化上述操作，我们可采取以下措施。

is_deleted字段：可将数据库中该字段的默认值设置为0。

create_time和update_time：可使用mybatis-plus的自动填充功能，所谓自动填充，就是通过统一配置，在插入或更新数据时，自动为某些字段赋值

```java
@Schema(description = "创建时间")
    @JsonIgnore
    @TableField(value = "create_time", fill = FieldFill.INSERT)
    private Date createTime;

    @Schema(description = "更新时间")
    @JsonIgnore
    @TableField(value = "update_time", fill = FieldFill.UPDATE)
    private Date updateTime;
```

在做完上述配置后，**当写入数据时，Mybatis-Plus会自动将实体对象的`create_time`字段填充为当前时间，当更新数据时，则会自动将实体对象的`update_time`字段填充为当前时间。**









## 7.2.2.3标签管理（*数据类型转换）

**详细说明见原文档**

#### 1、type字段类型转换。

前端、后端、数据库之间的字段类型转换。前端为String类型、后端代码为Enum枚举类型、数据库Integer类型。

**实现code(例如前端传入1)，到枚举类中的某个枚举常量（APARTMENT(1, "公寓"),）的相互转换**

1、自定义converterFactory 转换器并在 webmvcconfiguration  配置文件中注册。 

```java
@Component
public class StringToBaseEnumConverterFactory implements ConverterFactory<String, BaseEnum> {
    @Override
    public <T extends BaseEnum> Converter<String, T> getConverter(Class<T> targetType) {

        return new Converter<String, T>() {
            @Override
            public T convert(String code) {
                T[] enumConstants = targetType.getEnumConstants();
                for (T value : enumConstants) {
                    if (value.getCode().toString().equals(code)) {
                        return value;
                    }
                }

                throw new IllegalArgumentException();
            }
        };
    }
}
```

```java
@Configuration
public class WebMvcConfiguration implements WebMvcConfigurer {


    @Autowired
    private StringToBaseEnumConverterFactory stringToBaseEnumConverterFactory;

 

    @Override
    public void addFormatters(FormatterRegistry registry) {
        registry.addConverterFactory(this.stringToBaseEnumConverterFactory);
    }
}

```

只要是实现了BaseEnum接口的枚举类，都可以完成类型转换

```java
public interface BaseEnum {

    Integer getCode();

    String getName();
}
```

#### 2、 @EnumValue， **TypeHandler枚举类型转换**

Mybatis预置的`TypeHandler`可以处理常用的数据类型转换，例如`String`、`Integer`、`Date`等等，其中也包含枚举类型，但是枚举类型的默认转换规则是枚举对象实例（ItemType.APARTMENT）和实例名称（"APARTMENT"）相互映射。若想实现`code`属性到枚举对象实例的相互映射，需要自定义`TypeHandler`。

不过MybatisPlus提供了一个[通用的处理枚举类型的TypeHandler](https://baomidou.com/pages/8390a4/)。其使用十分简单，只需在`ItemType`枚举类的`code`属性上增加一个注解`@EnumValue`，Mybatis-Plus便可完成从`ItemType`对象到`code`属性之间的相互映射，具体配置如下。

#### 3、@JsonValue，**HTTPMessageConverter枚举类型转换**

`HttpMessageConverter`依赖于Json序列化框架（默认使用Jackson）。其对枚举类型的默认处理规则也是枚举对象实例（ItemType.APARTMENT）和实例名称（"APARTMENT"）相互映射。不过其提供了一个注解`@JsonValue`，同样只需在`ItemType`枚举类的`code`属性上增加一个注解`@JsonValue`，Jackson便可完成从`ItemType`对象到`code`属性之间的互相映射。具体配置如下，详细信息可参考Jackson[官方文档](https://fasterxml.github.io/jackson-annotations/javadoc/2.8/com/fasterxml/jackson/annotation/JsonValue.html)。





## 7.2.2.5基本属性管理(3查询)

#### 1、自定义查询业务及sql语句

sql语句联表查询（外连接left join）,过滤条件的设计，自定义返回对象格式 [自定义结果集resultMap(id  result  collection)]







## 7.2.2.8图片上传管理

#### 1、minio client 的配置

pom.xml中引入依赖

application.yml 中配置相关信息endpoint ,access-key,secret-key,bucket-name

将上述配置信息与一个属性类绑定，@configurationProperties

@EnableConfigurationProperties在主启动类或配置类中开启配置绑定

在配置类中创建并返回一个MinioClient对象，放入容器中供所有人使用。

#### 2、ServiceImpl中编写具体实现步骤

#### 3、全局异常处理器（由springMvc提供，要先引入web依赖）

编写一个全局异常处理器类处理所有的业务异常，@RestControllerAdvice声明全局异常处理器并返回处理结果

@ExceptionHandler(异常类.class)对不同异常类的具体处理方法上。







## 7.2.2.9公寓管理

#### 公寓管理与房间管理接口实现逻辑相同



#### 1、保存或更新公寓信息

用vo实体类接收前端数据，公寓中的额外信息是从其他表中查出的数据然后选择（除了新增的图片信息）。根据公寓id存在与否决定是保存或更新。if判断为更新则先删除原有的额外信息，再插入更改的信息（除新增图片外，通过公寓id加配套id从配套表中选择并保存）。

**保存或更新公寓信息，只需要修改中间表即可，通用service的saveBatch()方法可批量保存list对象。**



#### 2、根据条件分页查询公寓列表 

使用Mybatis-plus 内置的分页插件： 要先配置并注入容器。

```java
@Bean
public MybatisPlusInterceptor mybatisPlusInterceptor() {
    MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
    interceptor.addInnerInterceptor(new PaginationInnerInterceptor(DbType.MYSQL));
    return interceptor;
}
```

new一个Ipage对象存放分页查询出来的所有信息，返回给前端使用。

##### 超级大查询

1、多表的连接查询

2、动态SQL的使用

3、过滤条件的判断  ifnull的使用



#### 3、根据ID获取公寓详细信息（ 👉#{}与${}的区别 ）

需要使用Mapper，不推荐使用service因为可能发生循环依赖；自定义查询语句，从具体表中查出信息。

将基础信息和各个额外信息一一查出后再整合为ApartmentDetailVo对象返回。

sql语句中使用传递的参数时，mapper.xml中用#{}的形式引用，可以实现枚举类的自动识别code属性（ItemType.Apartment自动转为1）.但直接在console栏中无法使用#{},应使用${}。若在mapper.xml中使用${}则无法自动完成枚举类的sq转换。



#### 4、根据id删除公寓信息

**删除公寓信息前判断公寓下是否有房间为删除，有则向前端抛出异常。**

删除公寓基础信息和其他信息，修改中间表。





## 7.2.3.1看房预约管理

**知识点**：

`ViewAppointment`实体类中的`appointmentTime`字段为`Date`类型，`Date`类型的字段在序列化成JSON字符串时，需要考虑两个点，分别是**格式**和**时区**。本项目使用JSON序列化框架为Jackson，具体配置如下

- **格式**

  格式可按照字段单独配置，也可全局配置，下面分别介绍

  - **单独配置**

    在指定字段增加`@JsonFormat`注解，如下

    ```java
    @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss")
    private Date appointmentTime;
    ```

  - **全局配置**

    在`application.yml`中增加如下内容

    ```yml
    spring:
      jackson:
        date-format: yyyy-MM-dd HH:mm:ss
    ```

- **时区**

  时区同样可按照字段单独配置，也可全局配置，下面分别介绍

  - **单独配置**

    在指定字段增加`@JsonFormat`注解，如下

    ```java
    @JsonFormat(timezone = "GMT+8")
    private Date appointmentTime;
    ```

  - **全局配置**

    ```yml
    spring:
      jackson:
        time-zone: GMT+8
    ```

推荐格式按照字段单独配置，时区全局配置。





## 7.2.3.2（========）租约管理（定时任务）



#### 1、定时检查租约状态并更新

本节内容是通过定时任务定时检查租约是否到期。SpringBoot内置了定时任务，具体实现如下。

- **启用Spring Boot定时任务**

  在SpringBoot启动类上增加`@EnableScheduling`注解，如下

  ```java
  @SpringBootApplication
  @EnableScheduling   //开启springBoot内置的定时任务
  public class AdminWebApplication {
      public static void main(String[] args) {
          SpringApplication.run(AdminWebApplication.class, args);
      }
  }
  ```

- **编写定时逻辑**

  在**web-admin模块**下创建`com.atguigu.lease.web.admin.schedule.ScheduledTasks`类，内容如下

  

  可以**加入异常处理 **try-catch 提高代码健壮性

  ```java
  @Component
  public class ScheduledTasks {
  
      @Autowired
      private LeaseAgreementService leaseAgreementService;
  
      @Scheduled(cron = "0 0 0 * * *")
      public void checkLeaseStatus() {
  
          LambdaUpdateWrapper<LeaseAgreement> updateWrapper = new LambdaUpdateWrapper<>();
          Date now = new Date();
          updateWrapper.le(LeaseAgreement::getLeaseEndDate, now);  //租约已到期
          updateWrapper.eq(LeaseAgreement::getStatus, LeaseStatus.EXPIRED);  //将租约状态修改为已到期
          updateWrapper.in(LeaseAgreement::getStatus, LeaseStatus.SIGNED, LeaseStatus.WITHDRAWING);
          leaseAgreementService.update(updateWrapper);
          
      }
  }
  ```

  **知识点**:

  SpringBoot中的cron表达式语法如下

  ```
    ┌───────────── second (0-59)
    │ ┌───────────── minute (0 - 59)
    │ │ ┌───────────── hour (0 - 23)
    │ │ │ ┌───────────── day of the month (1 - 31)
    │ │ │ │ ┌───────────── month (1 - 12) (or JAN-DEC)
    │ │ │ │ │ ┌───────────── day of the week (0 - 7)
    │ │ │ │ │ │          (0 or 7 is Sunday, or MON-SUN)
    │ │ │ │ │ │
    * * * * * *
  ```



#### 2、定时任务补充

**Spring Boot 默认定时任务是单线程的**，如果一个任务卡住，会影响其他任务。

解决方案：配置定时任务线程池

```java
@Configuration
public class ScheduleConfig implements SchedulingConfigurer {
    
    @Override
    public void configureTasks(ScheduledTaskRegistrar taskRegistrar) {
        taskRegistrar.setScheduler(Executors.newScheduledThreadPool(5));
    }
}

```

分布式环境下，如果部署多个实例，定时任务会重复执行

解决方案：
使用 Redis 分布式锁
使用 XXL-JOB 等分布式任务调度平台
使用 ShedLock 库









## 7.2.5.2 后台用户信息管理



#### 1、保存或更新用户信息（密码加密处理,mybatis-plus的更新策略）

- **知识点**：

  - **密码处理**

    用户的密码通常不会直接以明文的形式保存到数据库中，而是会先经过处理，然后将处理之后得到的"密文"保存到数据库，这样能够降低数据库泄漏导致的用户账号安全问题。

    密码通常会使用一些单向函数进行处理，如下图所示

    <img src="D:/百度网盘Download/尚庭公寓项目/1.笔记/images/密码处理流程.drawio.png" style="zoom: 50%;" />

    常用于处理密码的单向函数（算法）有MD5、SHA-256等，**Apache Commons**提供了一个工具类`DigestUtils`，其中就包含上述算法的实现。

    > **Apache Commons**是Apache软件基金会下的一个项目，其致力于提供可重用的开源软件，其中包含了很多易于使用的现成工具。

    使用该工具类需引入`commons-codec`依赖，在**common模块**的pom.xml中增加如下内容

    ```xml
    <dependency>
        <groupId>commons-codec</groupId>
        <artifactId>commons-codec</artifactId>
    </dependency>
    ```

  - **Mybatis-Plus update strategy**  （推荐使用局部配置）

    使用Mybatis-Plus提供的更新方法时，若实体中的字段为`null`，默认情况下，最终生成的update语句中，不会包含该字段。若想改变默认行为，可做以下配置。

    - 全局配置

      在`application.yml`中配置如下参数

      ```xml
      mybatis-plus:
        global-config:
          db-config:
            update-strategy: <strategy>
      ```

      **注**：上述`<strategy>`可选值有：`ignore`、`not_null`、`not_empty`、`never`，默认值为`not_null`

      - `ignore`：忽略空值判断，不管字段是否为空，都会进行更新

      - `not_null`：进行非空判断，字段非空才会进行判断

      - `not_empty`：进行非空判断，并进行非空串（""）判断，主要针对字符串类型

      - `never`：从不进行更新，不管该字段为何值，都不更新

    - 局部配置

      在实体类中的具体字段通过`@TableField`注解进行配置，如下：

      ```java
      @Schema(description = "密码")
      @TableField(value = "password", updateStrategy = FieldStrategy.NOT_EMPTY)
      private String password;
      ```













## 7.2.6（========）登录管理（详细见文档）

jwt+ThreadLocal无状态登录

采用jwt登录，需定义jwtUtil工具类简化开发

```java
public class JwtUtil {
	//生成一个 HMAC-SHA 密钥
    private static SecretKey secretKey = Keys.hmacShaKeyFor("CY29Eb04RPNyQPxACH2jBNWFGn0ypMhc".getBytes());
    //创建token
    public static String createToken(Long userId, String username) {

        return Jwts.builder()		// 1. 创建JWT构建器
                .setSubject("LOGIN_USER")  	// 2. 设置主题（Subject），标识该令牌的用途或用户身份
                .setExpiration(new Date(System.currentTimeMillis() + 3600000 * 24 * 365L))   
                .claim("userId", userId)		//添加自定义声明（Claim）：键为 "userId"，值为变量 userId
                .claim("username", username)
                .signWith(secretKey, SignatureAlgorithm.HS256)  // 6. 使用密钥 secretKey 和 HMAC-SHA256 算法对JWT进行签名
                .compact();		// 7. 压缩并生成最终的JWT字符串（Base64Url编码的三段式）
    }
	//解析token
    public static Claims parseToken(String token) {

        if (token == null) {
            throw new LeaseException(ResultCodeEnum.ADMIN_LOGIN_AUTH);
        }

        try {
            Jws<Claims> claimsJws = Jwts.parserBuilder()   // 1. 创建JWT解析器的构建器
                    .setSigningKey(secretKey)   // 2. 设置验证签名所用的密钥（与生成时使用的密钥相同）
                    .build()
                    .parseClaimsJws(token);   // 4. 解析JWT字符串，验证签名和有效期，返回JWS对象
            return claimsJws.getBody();		 // 5. 从JWS对象中取出Claims（载荷体）
        } catch (ExpiredJwtException e) {
            throw new LeaseException(ResultCodeEnum.TOKEN_EXPIRED);
        } catch (JwtException e) {
            throw new LeaseException(ResultCodeEnum.TOKEN_INVALID);
        }
    }
}
```



#### 1、定义拦截器，获取并解析token，将登录用户信息存入本地线程

```java
@Component
public class AuthenticationInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {

        String token = request.getHeader("access-token");
        Claims claims = JwtUtil.parseToken(token);
        Long userId = claims.get("userId", Long.class);
        String username = claims.get("username", String.class);
        //存储登录用户信息到线程变量中
        LoginUserHolder.setLoginUser(new LoginUser(userId, username));

        return true;

    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
        //任务完成，清空ThreadLocal
        LoginUserHolder.clear();
    }
}
```

**注册拦截器**

```java
@Configuration
public class WebMvcConfiguration implements WebMvcConfigurer {


    @Autowired
    private StringToBaseEnumConverterFactory stringToBaseEnumConverterFactory;

    @Autowired
    private AuthenticationInterceptor authenticationInterceptor;

    @Override
    public void addFormatters(FormatterRegistry registry) {
        registry.addConverterFactory(this.stringToBaseEnumConverterFactory);
    }


    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(this.authenticationInterceptor).addPathPatterns("/admin/**").excludePathPatterns("/admin/login/**");
    }
}
```

**定义登录用户信息持有类**

```java
public class LoginUserHolder(){
    
    	public static ThreadLocal<LoginUser> threadLocal = new ThreadLocal<>();

        public static void setLoginUser(LoginUser loginUser) {
            threadLocal.set(loginUser);
        }

        public static LoginUser getLoginUser() {
            return threadLocal.get();
        }

        public static void clear() {
            threadLocal.remove();
        }
   
}
```















## 移动端

## 7.4.3.4房间信息



#### 1、根据条件分页查询房间列表

**知识点**：

- **xml文件`<`和`>`的转义**

  由于xml文件中的`<`和`>`是特殊符号，需要转义处理。

  | 原符号 | 转义符号 |
  | ------ | -------- |
  | `<`    | `&lt;`   |
  | `>`    | `&gt;`   |

- **Mybatis-Plus分页插件注意事项**

  使用Mybatis-Plus的分页插件进行分页查询时，如果结果需**要使用`<collection>`进行映射**，只能使用**[嵌套查询（Nested Select for Collection）](https://mybatis.org/mybatis-3/sqlmap-xml.html#nested-select-for-collection)**，而不能使用**[嵌套结果映射（Nested Results for Collection）](https://mybatis.org/mybatis-3/sqlmap-xml.html#nested-results-for-collection)**。











## 7.4.2.2 （=========）获取短信验证码

使用阿里云号码认正服务。

1、导入依赖

```xml
<!-- 阿里云号码认证服务  -->
        <dependency>
            <groupId>com.aliyun</groupId>
            <artifactId>dypnsapi20170525</artifactId>
            <version>2.0.0</version>
        </dependency>
```

2、配置yaml

```yaml
aliyun:
  sms:
    access-key-id: ${ACCESSKEY_ID}
    access-key-secret: QQMyEp3vzAt1XaxFUV8JQTukfUMWdN
    endpoint: dypnsapi.aliyuncs.com
```

3、properties属性类

```Java
@Data
@ConfigurationProperties(prefix = "aliyun.sms")
public class SmsProperties {

    private String accessKeyId;

    private String accessKeySecret;

    private String endpoint;
}
```

4、config配置类，创建client客户端，并注入容器。

```Java
@Configuration
@EnableConfigurationProperties(SmsProperties.class)
@ConditionalOnProperty(name = "aliyun.sms.endpoint")
public class SmsClientConfiguration {


    @Autowired
    private SmsProperties properties;

    @Bean
    public Client createClient() {
        //com.aliyun.credentials.Client client = new com.aliyun.credentials.Client();
        Config config = new Config();
        config.setEndpoint(properties.getEndpoint());
        config.setAccessKeyId(properties.getAccessKeyId());
        config.setAccessKeySecret(properties.getAccessKeySecret());
        try {
            return new Client(config);
        } catch (Exception e) {
            e.printStackTrace();
            throw new RuntimeException(e);
        }
    }
}
```

5、使用client发送验证码。







#### 7.4.4.1 保存浏览历史（异步执行）

- 保存浏览历史的动作应该在浏览房间详情时触发，所以在`RoomInfoServiceImpl`中的`getDetailById`方法的最后增加如下内容

  ```java
  browsingHistoryService.saveHistory(LoginUserContext.getLoginUser().getUserId(), id);
  ```

**保存浏览历史时，应先查询数据库浏览历史，若已存在，则更新浏览时间；否则保存新的浏览历史**。

**知识点**：

保存浏览历史的动作不应影响前端获取房间详情信息，故此处采取**异步操作**。Spring Boot提供了`@Async`注解来完成异步操作，具体使用方式为：

- 启用Spring Boot异步操作支持

  在 Spring Boot 主应用程序类上添加 `@EnableAsync` 注解

  ```java
  @SpringBootApplication
  @EnableAsync
  public class AppWebApplication {
      public static void main(String[] args) {
          SpringApplication.run(AppWebApplication.class);
      }
  }
  ```

- 在要进行异步处理的方法上添加 `@Async` 注解

**没有@Async**

```
用户请求 → Controller → Service.saveHistory() → 等待数据库操作完成 → 返回响应
                          ↑
                    主线程阻塞，必须等待这个方法执行完

```

**有@Async**

```
用户请求 → Controller → Service.saveHistory() → 立即返回响应
                          ↓
                    提交到线程池，由另一个线程执行
                          ↓
                    后台线程执行数据库操作（不阻塞主线程）

```

执行过程：
主线程调用 saveHistory() 时，Spring会立即返回
Spring将任务提交到线程池
线程池中的工作线程异步执行数据库插入/更新操作
用户可以立即收到房间详情的响应，体验更快

**底层实现机制**
	Spring使用 动态代理 + 线程池 实现异步：
	动态代理：Spring为包含 @Async 方法的Bean创建代理对象
	线程池：默认使用 SimpleAsyncTaskExecutor（每次创建新线程），也可以配置自定义线程池
	AOP拦截：当**调用 @Async 方法时，代理拦截调用并提交到线程池**









## 缓存优化问题。（重点）

**JMETER测试**

**接口响应时间380ms到3ms，QPS400到500+**   

**查询房间详情接口逻辑：**

**先查缓存，命中直接返回缓存数据**

**未命中尝试获取锁，获取锁失败等待重试，成功查询数据库，并存入缓存，释放锁。返回信息。** 

```java
public RoomDetailVo getDetailById(Long id) {
    String key=RedisConstant.APP_ROOM_PREFIX+id;
    RoomDetailVo roomDetailVo = (RoomDetailVo) redisTemplate.opsForValue().get(key);
    //缓存命中
    if(roomDetailVo!=null){
        return roomDetailVo;
    }

    //缓存未命中,重构缓存
    String lockKey="lock:admin"+id;
    roomDetailVo = new RoomDetailVo();
    try{
        //尝试获取锁
        boolean isLock = tryLock(lockKey);
        //获取锁失败,线程等待一段时间后重试
        if(!isLock){
            //Thread.sleep(50);
            return getDetailById(id);
        }
        //获取锁成功,查询数据库，重建缓存
        RoomInfo roomInfo = super.getById(id);
        Long apartmentId = roomInfo.getApartmentId();
        ApartmentInfo apartmentInfo = apartmentInfoMapper.selectById(apartmentId);

        //获取图片信息
        List<GraphVo> graphVoList = graphInfoMapper.selectListByItemTypeAndId(ItemType.ROOM, id);
        //获取属性信息
        List<AttrValueVo> attrValueVoList=roomAttrValueMapper.selectRoomAttrValueList(id);
        //获取配套信息
        List<FacilityInfo> facilityInfoList=roomFacilityMapper.selectRoomFacilityList(ItemType.ROOM,id);
        //获取标签信息
        List<LabelInfo> labelInfoList=roomLabelMapper.selectRoomLabelList(ItemType.ROOM,id);
        //获取支付方式列表
        List<PaymentType> paymentTypeList=roomPaymentTypeMapper.selectRoomPaymentTypeList(id);
        //获取可选租期列表
        List<LeaseTerm> leaseTermList=roomLeaseTermMapper.selectRoomLeaseTermList(id);

        BeanUtils.copyProperties(roomInfo, roomDetailVo);
        roomDetailVo.setApartmentInfo(apartmentInfo);
        roomDetailVo.setGraphVoList(graphVoList);
        roomDetailVo.setAttrValueVoList(attrValueVoList);
        roomDetailVo.setFacilityInfoList(facilityInfoList);
        roomDetailVo.setLabelInfoList(labelInfoList);
        roomDetailVo.setPaymentTypeList(paymentTypeList);
        roomDetailVo.setLeaseTermList(leaseTermList);
        redisTemplate.opsForValue().set(key,roomDetailVo);

    }catch (Exception e){
        e.printStackTrace();
        throw new RuntimeException(e);
    }finally {
        //释放锁
        unlock(lockKey);
    }
    return roomDetailVo;
}
```





**项目代码数：8500行**

**java代码：7000行**

**xml中SQL代码：1500行**

**配置文件100行**





**复杂业务逻辑处理示例**

**1、保存或更新房间/公寓详细信息（多对多属性管理）**

属性，配套，标签，可选支付方式，可选租期，图片。

保存：直接保存上述信息。**保存或更新公寓信息，只需要修改中间表即可，通用service的saveBatch()方法可批量保存list对象。**

```java
// 保存属性信息 room_attr_value
List<Long> attrValueIds = roomSubmitVo.getAttrValueIds();
if(!CollectionUtils.isEmpty(attrValueIds)){
    //RoomAttrValue 使用@Bulider注解，使用builder()方法创建对象，不能使用new
    ArrayList<RoomAttrValue> roomAttrValues = new ArrayList<>();
    for (Long attrValueId : attrValueIds) {
        RoomAttrValue roomAttrValue = RoomAttrValue.builder()
                .roomId(id)
                .attrValueId(attrValueId)
                .build();
        roomAttrValues.add(roomAttrValue);
    }
    roomAttrValueService.saveBatch(roomAttrValues);
}
```

更新：先删除原有信息和缓存，在保存

```java
//删除图片列表
LambdaQueryWrapper<GraphInfo> graphQueryWrapper = new LambdaQueryWrapper<>();
graphQueryWrapper.eq(GraphInfo::getItemId, id);
graphQueryWrapper.eq(GraphInfo::getItemType, ItemType.ROOM);
graphInfoService.remove(graphQueryWrapper);

//删除房间信息的同时删除缓存。
redisTemplate.delete(RedisConstant.APP_LOGIN_PREFIX + roomSubmitVo.getId());
```



2、根据条件分页查询房间/公寓列表

**MyBatis Plus分页插件 + 自定义SQL，完成多表关联分页查询**

```java
@Bean
public MybatisPlusInterceptor mybatisPlusInterceptor() {
    MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
    interceptor.addInnerInterceptor(new PaginationInnerInterceptor(DbType.MYSQL));
    return interceptor;
}
```

```java
IPage<RoomItemVo> page = new Page<>(current, size);
IPage<RoomItemVo> list = roomInfoService.pageItem(page, queryVo);
return Result.ok(list);
```

```java
IPage<RoomItemVo> result = roomInfoMapper.pageItem(page, queryVo);
```

```xml
<resultMap id="RoomItemVoMap" type="com.atguigu.lease.web.admin.vo.room.RoomItemVo" autoMapping="true">
        <id column="id" property="id"/>
        <association property="apartmentInfo" javaType="com.atguigu.lease.model.entity.ApartmentInfo"
                     autoMapping="true">
            <id column="apart_id" property="id"/>
            <result column="apart_is_release" property="isRelease"/>
        </association>
</resultMap>

<select id="pageItem" resultMap="RoomItemVoMap">
    select
        ri.id,
        ri.room_number,
        ri.rent,
        ri.apartment_id,
        ri.is_release,
        ai.id as apart_id,
        ai.is_release as apart_is_release,
        ai.name,
        ai.introduction,
        ai.district_id,
        ai.district_name,
        ai.city_id,
        ai.city_name,
        ai.province_id,
        ai.province_name,
        ai.address_detail,
        ai.latitude,
        ai.longitude,
        ai.phone,
        la.room_id is not null is_check_in,
        la.lease_end_date
    from room_info ri
    left join apartment_info ai on ri.apartment_id=ai.id and ai.is_deleted=0
    left join lease_agreement la on la.room_id=ri.id and la.is_deleted=0 and la.status in (2,5)
    <where>
        ri.is_deleted=0
        <if test="queryVo.provinceId != null">
            and ai.province_id=#{queryVo.provinceId}
        </if>
        <if test="queryVo.cityId != null">
            and ai.city_id=#{queryVo.cityId}
        </if>
        <if test="queryVo.districtId != null">
            and ai.district_id=#{queryVo.districtId}
        </if>
        <if test="queryVo.apartmentId != null">
            and ri.apartment_id=#{queryVo.apartmentId}
        </if>
    </where>
</select>
```









# 黑马点评



## 1、登录状态刷新

登录使用redis，未使用session.

登录状态刷新：在登录接口创建token时将其存入redis并设置有效期。通过定义拦截器，并在拦截器中获取redis中的token，获取成功，则刷新token 有效期。



## 2、缓存更新策略（如何解决数据库和缓存数据不一致）

**三种解决方案（选第一个）**

Cache Aside Pattern 人工编码方式：缓存调用者在更新完数据库后再去更新缓存，也称之为**双写方案**

Read/Write Through Pattern : 由系统本身完成，数据库与缓存的问题交由系统本身去处理

Write Behind Caching Pattern ：调用者只操作缓存，其他线程去异步处理数据库，实现最终一致

**操作缓存和数据库时有三个问题需要考虑**：

* 删除缓存还是更新缓存？
  * 更新缓存：每次更新数据库都更新缓存，无效写操作较多
  * **删除缓存：更新数据库时让缓存失效，查询时再更新缓存**
* 如何保证缓存与数据库的操作的同时成功或失败？
  * **单体系统，将缓存与数据库操作放在一个事务**
  * 分布式系统，利用TCC等分布式事务方案
* 先操作缓存还是先操作数据库？
  * 先删除缓存，再操作数据库
  * **先操作数据库，再删除缓存**





## 3、缓存穿透、击穿、雪崩。

1、**缓存穿透** ：缓存穿透是指客户端请求的数据在缓存中和数据库中都不存在，这样缓存永远不会生效，这些请求都会打到数据库。

常见的解决方案有两种：

* **缓存空对象**
  * 优点：实现简单，维护方便
  * 缺点：
    * 额外的内存消耗
    * 可能造成短期的不一致
* 布隆过滤
  * 优点：内存占用较少，没有多余key
  * 缺点：
    * 实现复杂
    * 存在误判可能

2、**缓存雪崩**是指在同一时段大量的缓存key同时失效或者Redis服务宕机，导致大量请求到达数据库，带来巨大压力。

解决方案：

* 给不同的Key的**TTL**添加随机值
* 利用**Redis集群**提高服务的可用性
* 给缓存业务添加降级限流策略
* 给业务添加多级缓存

3、**缓存击穿**（redis失效，数据库中有数据）问题也叫热点Key问题，就是一个被高并发访问并且缓存重建业务较复杂的key突然失效了，无数的请求访问会在瞬间给数据库带来巨大的冲击。

常见的解决方案有两种：

* **互斥锁(分布式锁)**
* 逻辑过期

两种方案详细解释和实现见文档。

**互斥锁(分布式锁)**

相较于原来从缓存中查询不到数据后直接查询数据库而言，现在的方案是 进行查询之后，如果从缓存没有查询到数据，则进行互斥锁的获取，获取互斥锁后，判断是否获得到了锁，如果没有获得到，则休眠，过一会再进行尝试，直到获取到锁为止，才能进行查询

如果获取到了锁的线程，再去进行查询，查询后将数据写入redis，再释放锁，返回数据，利用互斥锁就能保证只有一个线程去执行操作数据库的逻辑，防止缓存击穿

**逻辑过期**

当用户开始查询redis时，判断是否命中，如果没有命中则直接返回空数据，不查询数据库，而一旦命中后，将value取出，判断value中的过期时间是否满足，如果没有过期，则直接返回redis中的数据，如果过期，则在开启独立线程后直接返回之前的数据，独立线程去重构数据，重构完成后释放互斥锁。





利用互斥锁解决缓存击穿问题，**测试根据商铺id查询信息接口**，测试结果如下。

![](C:\Users\缪雨\Pictures\Screenshots\屏幕截图 2026-06-04 150000.png)

2秒发1000个请求，qps(http请求就=ops)(每秒处理的请求数量)大约500/s.

接口平均响应时间1ms.







## 4、优惠卷秒杀

**秒杀下单逻辑**

当用户开始进行下单，我们应当去查询优惠卷信息，查询到优惠卷信息，判断是否满足秒杀条件

比如时间是否充足，如果时间充足，则进一步判断库存是否足够，如果两者都满足，则扣减库存，创建订单，然后返回订单id，如果有一个条件不满足则直接结束。

**解决超卖问题-加锁**

**乐观锁**关键是在更新数据判断有没有其他线程对数据做了修改。

此处使用乐观锁，只要我扣减库存时的库存大于0即可。**(利用数据库事务特性保证不超卖)**

```java
boolean success = seckillVoucherService.update()
            .setSql("stock= stock -1")
            .eq("voucher_id", voucherId).update().gt("stock",0); //where id = ? and stock > 0
```

### 1、一人一单-加悲观锁（单体系统）（不好）

比如时间是否充足，如果时间充足，则进一步判断库存是否足够，然后再根据优惠卷id和用户id查询是否已经下过这个订单，如果下过这个订单，则不再下单，否则进行下单。

synchronized锁可以解决，但需考虑**锁粒度**

1、**intern() 这个方法是从常量池中拿到数据**，如果我们直接使用userId.toString() 他拿到的对象实际上是不同的对象，new出来的对象，我们使用锁必须保证锁必须是同一把，所以我们需要使用intern()方法

2、**当前方法被spring的事务控制**，如果你在方法内部加锁，可能会导致当前方法事务还没有提交，但是锁已经释放也会导致问题，所以我们选择将**当前方法整体包裹起来**，确保事务不会出现问题

3、**用this方式调用方法事务不生效**，事务想要生效，还得**利用代理来生效**，所以这个地方，我们需要获得原始的事务对象， 来操作事务

```
synchronized (userId.toString().intern()){
            //获取代理对象（事务）
            IVoucherOrderService proxy = (IVoucherOrderService) AopContext.currentProxy();
           return proxy.createVoucherOrder(voucherId);
}
```



### 2、一人一单-分布式系统（redission分布式锁）

```java
 RLock lock = redissonClient.getLock("lock:order:" + userId);
        //获取锁对象
        boolean isLock = lock.tryLock();
       
		//加锁失败
        if (!isLock) {
            return Result.fail("不允许重复下单");
        }
        try {
            //获取代理对象(事务)
            IVoucherOrderService proxy = (IVoucherOrderService) AopContext.currentProxy();
            return proxy.createVoucherOrder(voucherId);
        } finally {
            //释放锁
            lock.unlock();
        }
 }
```



### 3、（---------------）基于分布式锁实现秒杀优化

修改下单动作，现在我们去下单时，是通过lua表达式去原子执行判断逻辑，如果判断我出来不为0 ，则要么是库存不足，要么是重复下单，返回错误信息，如果是0，则把下单的逻辑保存到队列中去，然后异步执行

秒杀业务的优化思路是什么？

* 先利用Redis完成库存余量、一人一单判断，完成抢单业务

* 再将下单业务放入消息队列，利用独立线程异步下单

  

  

**未优化思路：**

正常判断库存是否充足（voucher.getStock()>0）,

充足：（手写分布式锁）redissonClient获取锁，失败返回不允许重复下单；成功调用代理对象**创建订单**，释放锁。

**创建订单方法：**先查询数据库判断是否下过单，没有则**扣减库存**（**乐观锁stock>0**）扣减失败订单不足，否则将订单存入数据库。





**秒杀优化思路解析**：

前提添加秒杀卷时同步存入redis

主线程提前生成订单号，再执行lua脚本，下单成功主线程直接返回订单号，由异步线程创建订单。失败返回原因（1或2）。

1、先利用lua脚本完成库存余量、一人一单判断（**redis层面一次判断基于秒杀卷**）， 根据lua脚本返回值  0：可以下单，1：库存不足，		2：不可重复下单，完成抢单业务。抢单成功生成一个订单加入消息队列（Redis Stream）异步执行。

2、处理订单异步线程VoucherOrderHandler(线程池中的任务)：从消息队列中获取并解析订单信息，**调用创建订单方法**（createVoucherOrder2），并确认消息已消费。（若出现异常，从待处理消息队列中重新获取订单，创建，确认消息；相当于重新执行一次异步订单处理逻辑）。

**创建订单方法：**（**数据库层面二次判断一人一单和库存余量**）先查询数据库判断是否下过单，没有则扣减库存扣减失败订单不足，否则将订单存入数据库。





## 5、分布式锁

分布式锁：满足分布式系统或集群模式下多进程可见并且互斥的锁。

分布式锁的核心思想就是让大家都使用同一把锁，只要大家使用的是同一把锁，那么我们就能锁住线程，不让线程进行，让程序串行执行，这就是分布式锁的核心思路

### **1、简单实现**

利用setnx方法（**redis2.6.12版本之前不具原子性，之后有，set nx ex**）进行加锁，同时增加过期时间，防止死锁，此方法可以保证**加锁和增加过期时间具有原子性**。

```java
 // 获取锁,加锁和增加过期时间一条命令完成具有原子性
    Boolean success = stringRedisTemplate.opsForValue()
            .setIfAbsent(KEY_PREFIX + name, threadId + "", timeoutSec, TimeUnit.SECONDS);
```

**通过delete释放锁不能保证原子性。**

```java
//通过del删除锁
    stringRedisTemplate.delete(KEY_PREFIX + name);
```

原代码逻辑判断库存充足后，**生成订单操作加锁**。

```java
 //创建锁对象(新增代码)
        SimpleRedisLock lock = new SimpleRedisLock("order:" + userId, stringRedisTemplate);
        //获取锁对象
        boolean isLock = lock.tryLock(1200);
		//加锁失败
        if (!isLock) {
            return Result.fail("不允许重复下单");
        }
        try {
            //获取代理对象(事务)
            IVoucherOrderService proxy = (IVoucherOrderService) AopContext.currentProxy();
            return proxy.createVoucherOrder(voucherId);
        } finally {
            //释放锁
            lock.unlock();
        }
```



### **2、分布式锁误删问题**

持有锁的线程在锁的内部出现了阻塞，导致他的锁自动释放，这时其他线程，线程2来尝试获得锁，就拿到了这把锁，然后线程2在持有锁执行过程中，线程1反应过来，继续执行，而线程1执行过程中，走到了删除锁逻辑，此时就会把本应该属于线程2的锁进行删除，这就是误删别人锁的情况说明

解决方案：**解决方案就是在每个线程释放锁的时候，去判断一下当前这把锁是否属于自己，如果不属于自己，则不进行锁的删除**，假设还是上边的情况，线程1卡顿，锁自动释放，线程2进入到锁的内部执行逻辑，此时线程1反应过来，然后删除锁，但是线程1，一看当前这把锁不是属于自己，于是不进行删除锁逻辑，当线程2走到删除锁逻辑时，如果没有卡过自动释放锁的时间点，则判断当前这把锁是属于自己的，于是删除这把锁。

**具体实现**

**在获取锁时存入线程标示（可以用UUID表示+加线程id）**
**在释放锁时先获取锁中的线程标示，判断是否与当前线程标示一致**

* 如果一致则释放锁
* 如果不一致则不释放锁

```java
private static final String ID_PREFIX = UUID.randomUUID().toString(true) + "-";
@Override
public boolean tryLock(long timeoutSec) {
   // 获取线程标示，作为value
   String threadId = ID_PREFIX + Thread.currentThread().getId();
   // 获取锁
   Boolean success = stringRedisTemplate.opsForValue()
                .setIfAbsent(KEY_PREFIX + name, threadId, timeoutSec, TimeUnit.SECONDS);
   return Boolean.TRUE.equals(success);
}
```

释放锁

```java
public void unlock() {
    // 获取线程标示
    String threadId = ID_PREFIX + Thread.currentThread().getId();
    // 获取锁中的标示
    String id = stringRedisTemplate.opsForValue().get(KEY_PREFIX + name);
    // 判断标示是否一致
    if(threadId.equals(id)) {
        // 释放锁
        stringRedisTemplate.delete(KEY_PREFIX + name);
    }
}
```

此时，还不算完全解决，仍有可能出现误删情况。**因为检查锁和释放锁并不满座原子性。**



### 3、基于set nx ex,lua脚本实现分布式锁

（1）**加锁**

利用setnx方法（**redis2.6.12版本之前不具原子性，之后有，set nx ex**）进行加锁，同时增加过期时间，防止死锁，此方法可以保证**加锁和增加过期时间具有原子性**。

```java
private static final String ID_PREFIX = UUID.randomUUID().toString(true) + "-";
@Override
public boolean tryLock(long timeoutSec) {
   // 获取线程标示
   String threadId = ID_PREFIX + Thread.currentThread().getId();
   // 获取锁
   Boolean success = stringRedisTemplate.opsForValue()
                .setIfAbsent(KEY_PREFIX + name, threadId, timeoutSec, TimeUnit.SECONDS);
   return Boolean.TRUE.equals(success);
}
```

（2）**释放锁**

**使用lua脚本。**

我们的RedisTemplate中，可以利用execute方法去执行lua脚本

```java
private static final DefaultRedisScript<Long> UNLOCK_SCRIPT;
    static {
        UNLOCK_SCRIPT = new DefaultRedisScript<>();
        UNLOCK_SCRIPT.setLocation(new ClassPathResource("unlock.lua"));
        UNLOCK_SCRIPT.setResultType(Long.class);
    }

public void unlock() {
    // 调用lua脚本
    stringRedisTemplate.execute(
            UNLOCK_SCRIPT,
            Collections.singletonList(KEY_PREFIX + name),
            ID_PREFIX + Thread.currentThread().getId());
}
经过以上代码改造后，我们就能够实现 拿锁比锁删锁的原子性动作了~
```



### 4、（=============）基于Redission实现分布式锁

基于setnx实现的分布式锁存在一下问题：

**重入问题**：同一个线程无法多次获取同一把锁

**不可重试**：获取锁只尝试一次就返回false，没有重试机制

**超时释放（无法自动续期）**：锁超时释放虽然可以避免死锁，但如果业务执行时间过长，也会导致锁释放，存在安全隐患

**主从一致性：** 如果Redis提供了主从集群，当我们向集群写数据时，主机需要异步的将数据同步给从机，而万一在同步过去之前，主机宕机了，就会出现死锁问题。

使用步骤：

（1）导入依赖

（2）配置Redission客户端

（3）使用Redission的分布式锁



**1、解决超时释放（无法自动续期）**

- **WatchDog 原理**：当锁获取成功后，Redisson 会启动一个**后台定时任务**（Netty 的 `Timeout`），**每隔锁过期时间的三分之一**（默认 10 秒）去检查业务是否还在执行。如果还在，就自动续期（重新设置过期时间到 30 秒）。业务释放锁或客户端宕机后，续期停止。



**2、解决不可重入**

利用 Redis **Hash（哈希）** 存储**线程ID和重入计数**，通过原子性的**Lua脚本（Lua Script）** 实现智能识别与计数。

- 若锁不存在 → 创建 Hash，次数=1，设过期时间。
- 若锁已存在且 field 是当前线程 → 次数+1，刷新过期时间。
- 否则 → 加锁失败。
  解锁时次数-1，减到 0 才真正删除锁。



**3、不可重试**

- **可重试性**：提供**带超时参数的 `tryLock` 方法**，内部利用 **发布-订阅模式（Pub/Sub）** 和 **信号量（Semaphore）** 实现高效的阻塞等待与唤醒。

获取锁失败时，Redisson 会**订阅**该锁的**释放频道**，当前**线程通过 `Semaphore` 阻塞等待**；持有锁的线程**释放锁后**，发布一条通知，唤醒所有等待线程重新尝试加锁。



## 6、Redis消息队列

基于List实现消息队列

基于PubSub实现消息队列

应该使用专门的消息队列--**RabbitMQ**



### （============）基于Stream实现消息队列

stream消息队列的**XREADGROUP命令**特点：

* 消息可回溯
* 可以**多消费者争抢消息**，加快消费速度（消费者组（Consumer Group）：将多个消费者划分到一个组中，监听同一个队列）
* 可以阻塞读取
* 没有**消息漏读的风险**
* 有**消息确认机制，保证消息至少被消费一次**



1、面试可能问到：

**消费者挂了会丢失吗？**

- 如果消费者挂了但 **没有 ACK**，消息会留在 Pending 列表（保存以消费未确认的消息）中，待消费者恢复后可以重新消费。



**秒杀异步下单：数据库最终一致性 & 回滚**

**如何保证数据库最终一致性？如果下单失败，怎么回滚？**

原有逻辑能保证最终一致性（消息一定能被消费）。

- 场景：数据库扣减失败（如网络异常、乐观锁失败）。此时 Redis 库存已经扣了，需要补偿。
- **回滚方式**：
  - 发送一条补偿消息（或直接调用 Redis Lua 脚本）将 Redis 库存加回去。



```java
private class VoucherOrderHandler implements Runnable {

    @Override
    public void run() {
        while (true) {
            try {
                // 1.获取消息队列中的订单信息 XREADGROUP GROUP g1 c1 COUNT 1 BLOCK 2000 STREAMS s1 >
                // ">"：从下一个未消费的消息开始
                List<MapRecord<String, Object, Object>> list = stringRedisTemplate.opsForStream().read(
                    Consumer.from("g1", "c1"),
                    StreamReadOptions.empty().count(1).block(Duration.ofSeconds(2)),
                    StreamOffset.create("stream.orders", ReadOffset.lastConsumed())
                );
                // 2.判断订单信息是否为空
                if (list == null || list.isEmpty()) {
                    // 如果为null，说明没有消息，继续下一次循环
                    continue;
                }
                // 解析数据
                MapRecord<String, Object, Object> record = list.get(0);
                Map<Object, Object> value = record.getValue();
                VoucherOrder voucherOrder = BeanUtil.fillBeanWithMap(value, new VoucherOrder(), true);
                // 3.创建订单
                createVoucherOrder(voucherOrder);
                // 4.确认消息 XACK
                stringRedisTemplate.opsForStream().acknowledge("s1", "g1", record.getId());
            } catch (Exception e) {
                log.error("处理订单异常", e);
                //处理异常消息
                handlePendingList();
            }
        }
    }

    private void handlePendingList() {
        while (true) {
            try {
                // 1.获取pending-list中的订单信息 XREADGROUP GROUP g1 c1 COUNT 1 BLOCK 2000 STREAMS s1 0
                //根据指定id从pending-list中获取已消费但未确认的消息，例如0，是从pending-list中的第一个消息开始
                List<MapRecord<String, Object, Object>> list = stringRedisTemplate.opsForStream().read(
                    Consumer.from("g1", "c1"),
                    StreamReadOptions.empty().count(1),
                    StreamOffset.create("stream.orders", ReadOffset.from("0"))
                );
                // 2.判断订单信息是否为空
                if (list == null || list.isEmpty()) {
                    // 如果为null，说明没有异常消息，结束循环
                    break;
                }
                // 解析数据
                MapRecord<String, Object, Object> record = list.get(0);
                Map<Object, Object> value = record.getValue();
                VoucherOrder voucherOrder = BeanUtil.fillBeanWithMap(value, new VoucherOrder(), true);
                // 3.创建订单
                createVoucherOrder(voucherOrder);
                // 4.确认消息 XACK
                stringRedisTemplate.opsForStream().acknowledge("s1", "g1", record.getId());
            } catch (Exception e) {
                log.error("处理pendding订单异常", e);
                try{
                    Thread.sleep(20);
                }catch(Exception e){
                    e.printStackTrace();
                }
            }
        }
    }
}
```







## 7、达人探店

### 1、点赞功能实现(set确保数据不重复)

添加一个isLike字段，标示是否被当前用户点赞，在点赞功能中先查询redis中是否有点赞记录。有则islike字段-1，并移除redis中的缓存；没有则islike字段+1，并写入redis缓存。



### 2、点赞排行榜（Zset）

逻辑同上，只是redis使用Zset,保存时将**当前时间戳作为分数**存入。

```java
 stringRedisTemplate.opsForZSet().add(key, userId.toString(), System.currentTimeMillis());
```



**查询top5的点赞用户**(Zset默认升序排序从小到大)

```java
String key = BLOG_LIKED_KEY + id;
    // 1.查询top5的点赞用户 zrange key 0 4
    Set<String> top5 = stringRedisTemplate.opsForZSet().range(key, 0, 4);
```

**stream流类型转换—一对一用Map**

```java
// 2.解析出其中的用户id
    List<Long> ids = top5.stream().map(Long::valueOf).collect(Collectors.toList());
```







## 8、好友关注

### 1、共同关注

**思路：**

把两人的关注的人分别放入到一个set集合中，然后再通过api去查看这两个set集合中的交集数据。

**关注接口改造：**

在用户关注了某位用户后，需要将数据放入到set集合中，方便后续进行共同关注，同时当取消关注时，也需要从set集合中进行删除

求共同好友关注接口

```java
@Override
public Result followCommons(Long id) {
    // 1.获取当前用户
    Long userId = UserHolder.getUser().getId();
    String key = "follows:" + userId;
    // 2.求交集
    String key2 = "follows:" + id;
    Set<String> intersect = stringRedisTemplate.opsForSet().intersect(key, key2);
    if (intersect == null || intersect.isEmpty()) {
        // 无交集
        return Result.ok(Collections.emptyList());
    }
    // 3.解析id集合
    List<Long> ids = intersect.stream().map(Long::valueOf).collect(Collectors.toList());
    // 4.查询用户
    List<UserDTO> users = userService.listByIds(ids)
            .stream()
            .map(user -> BeanUtil.copyProperties(user, UserDTO.class))
            .collect(Collectors.toList());
    return Result.ok(users);
}
```



### **2、feed流内容推送**(Zset)

feed流常见的两种模式：

**Timeline**：不做内容筛选，简单的按照内容发布时间排序，常用于好友或关注。例如朋友圈

智能排序：利用智能算法屏蔽掉违规的、用户不感兴趣的内容。推送用户感兴趣信息来吸引用户

此处采用timeline模式，有三种实现方案：

**拉模式**：也叫做读扩散，读关注的人的收件箱。

**推模式**：也叫做写扩散。写到粉丝的收件箱。

**推拉结合模式**：也叫做读写混合，兼具推和拉两种模式的优点。



（1）**推送到**粉丝收件箱：就是我们在保存完探店笔记后，获得到当前笔记的粉丝，然后把数据推送到粉丝的redis中去。

```java
// 3.查询笔记作者的所有粉丝 select * from tb_follow where follow_user_id = ?
    List<Follow> follows = followService.query().eq("follow_user_id", user.getId()).list();
    // 4.推送笔记id给所有粉丝
    for (Follow follow : follows) {
        // 4.1.获取粉丝id
        Long userId = follow.getUserId();
        // 4.2.推送
        String key = FEED_KEY + userId;
        stringRedisTemplate.opsForZSet().add(key, blog.getId().toString(), System.currentTimeMillis());
    }
```

(2）粉丝分页查询收件箱

因为收件箱内容会发生变化，所以不能使用传统分页，改**用feed流滚动分页。**

**优先查询显示最新发布的内容，zrevrange降序查询Zset(新发布的时间戳值大)**

具体操作如下：

**最小时间戳+查询个数作为偏移滚动分页**

1、每次查询完成后，我们要分析出查询出数据的最小时间戳，这个值会作为下一次查询的条件

2、我们需要找到与上一次查询相同的查询个数作为偏移量，下次查询时，跳过这些查询过的数据，拿到我们需要的数据









## 9、附近商户（GEO）

### 1、导入商铺数据到redis（GEOADD）

将商铺按类型分类，**typeId做为key**，**Geo数据结构值为RedisGeoCommands.GeoLocation<T>类**，将商铺id和经纬度信息存入该类。**以该类为value**，存入redis.

```java
@Test
void loadShopData() {
    // 1.查询店铺信息
    List<Shop> list = shopService.list();
    // 2.把店铺分组，按照typeId分组，typeId一致的放到一个集合
    Map<Long, List<Shop>> map = list.stream().collect(Collectors.groupingBy(Shop::getTypeId));
    // 3.分批完成写入Redis
    for (Map.Entry<Long, List<Shop>> entry : map.entrySet()) {
        // 3.1.获取类型id
        Long typeId = entry.getKey();
        String key = SHOP_GEO_KEY + typeId;
        // 3.2.获取同类型的店铺的集合
        List<Shop> value = entry.getValue();
        List<RedisGeoCommands.GeoLocation<String>> locations = new ArrayList<>(value.size());
        // 3.3.写入redis GEOADD key 经度 纬度 member
        for (Shop shop : value) {
            // stringRedisTemplate.opsForGeo().add(key, new Point(shop.getX(), shop.getY()), shop.getId().toString());
            locations.add(new RedisGeoCommands.GeoLocation<>(
                    shop.getId().toString(),
                    new Point(shop.getX(), shop.getY())
            ));
        }
        stringRedisTemplate.opsForGeo().add(key, locations);
    }
}
```



### 2、查询附近商户（GEOSEARCH-6.2后新功能）

**分页采用传统逻辑：**from=(current-1)\*size;     end=current\*size

**查询出前end页所有内容，再分页。**

```java
@Override
    public Result queryShopByType(Integer typeId, Integer current, Double x, Double y) {
        // 1.判断是否需要根据坐标查询
        if (x == null || y == null) {
            // 不需要坐标查询，按数据库查询
            Page<Shop> page = query()
                    .eq("type_id", typeId)
                    .page(new Page<>(current, SystemConstants.DEFAULT_PAGE_SIZE));
            // 返回数据
            return Result.ok(page.getRecords());
        }

        // 2.计算分页参数
        int from = (current - 1) * SystemConstants.DEFAULT_PAGE_SIZE;
        int end = current * SystemConstants.DEFAULT_PAGE_SIZE;

        // 3.查询redis、按照距离排序、分页。结果：shopId、distance
        String key = SHOP_GEO_KEY + typeId;
        GeoResults<RedisGeoCommands.GeoLocation<String>> results = stringRedisTemplate.opsForGeo() // GEOSEARCH key BYLONLAT x y BYRADIUS 10 WITHDISTANCE
                .search(
                        key,
                        GeoReference.fromCoordinate(x, y),
                        new Distance(5000),
                        RedisGeoCommands.GeoSearchCommandArgs.newGeoSearchArgs().includeDistance().limit(end)
                );
        // 4.解析出id
        if (results == null) {
            return Result.ok(Collections.emptyList());
        }
        List<GeoResult<RedisGeoCommands.GeoLocation<String>>> list = results.getContent();
        if (list.size() <= from) {
            // 没有下一页了，结束
            return Result.ok(Collections.emptyList());
        }
        // 4.1.截取 from ~ end的部分
        // 存储shopId
        List<Long> ids = new ArrayList<>(list.size());
        // 存储distance
        Map<String, Distance> distanceMap = new HashMap<>(list.size());
        list.stream().skip(from).forEach(result -> {
            // 4.2.获取店铺id
            String shopIdStr = result.getContent().getName();
            ids.add(Long.valueOf(shopIdStr));
            // 4.3.获取距离
            Distance distance = result.getDistance();
            distanceMap.put(shopIdStr, distance);
        });
        // 5.根据id查询Shop
        String idStr = StrUtil.join(",", ids);
        List<Shop> shops = query().in("id", ids).last("ORDER BY FIELD(id," + idStr + ")").list();
        for (Shop shop : shops) {
            shop.setDistance(distanceMap.get(shop.getId().toString()).getValue());
        }
        // 6.返回
        return Result.ok(shops);
    }
```











## 10、用户签到（BitMap）

**BitMap基于string类型数据结构实现的 。**

### 1、实现签到功能

SETBIT：向指定位置（offset）存入一个0或1

思路：我们可以把年和月作为bitMap的key，然后保存到一个bitMap中，每次签到就到对应的位上把数字从0变成1，只要对应是1，就表明说明这一天已经签到了，反之则没有签到。

```java
@Override
public Result sign() {
    // 1.获取当前登录用户
    Long userId = UserHolder.getUser().getId();
    // 2.获取日期
    LocalDateTime now = LocalDateTime.now();
    // 3.拼接key
    String keySuffix = now.format(DateTimeFormatter.ofPattern(":yyyyMM"));
    String key = USER_SIGN_KEY + userId + keySuffix;
    // 4.获取今天是本月的第几天
    int dayOfMonth = now.getDayOfMonth();
    // 5.写入Redis SETBIT key offset 1
    stringRedisTemplate.opsForValue().setBit(key, dayOfMonth - 1, true);
    return Result.ok();
}
```



### 2、签到统计

（1）只统计本月已签到的次数，使用**BITCOUNT** ：统计BitMap中值为1的bit位的数量。

（2）统计**当前用户截止当前时间在本月的连续签到天数**

思路：使用**BItField 获取本月到今天为止的所有签到数据（返回值是十进制），从后向前遍历每个bit位**：我们把签到结果和1进行与操作，每与一次，就把签到结果向右移动一位，依次内推，我们就能完成逐个遍历的效果了。

BITFIELD key GET <encoding> <offset>

- **encoding**：指定如何解析数据，即**截取多长**。
  - `u8`、`u16`...：无符号整数（Unsigned）。
  - `i8`、`i16`...：有符号整数（Signed）。
- **offset**：起始位偏移量（从 0 开始计数）。
  - 可以直接写数字，如 `10`（第 10 位）。

**BITFIELD sign:5:202203 GET u14 0：截取长度为14（0-13位）。返回值是一个整数数组，只有一个元素即截取段对应的十进制数。**

```java
@Override
public Result signCount() {
    // 1.获取当前登录用户
    Long userId = UserHolder.getUser().getId();
    // 2.获取日期
    LocalDateTime now = LocalDateTime.now();
    // 3.拼接key
    String keySuffix = now.format(DateTimeFormatter.ofPattern(":yyyyMM"));
    String key = USER_SIGN_KEY + userId + keySuffix;
    // 4.获取今天是本月的第几天
    int dayOfMonth = now.getDayOfMonth();
    // 5.获取本月截止今天为止的所有的签到记录，返回的是一个十进制的数字 BITFIELD sign:5:202203 GET u14 0
    List<Long> result = stringRedisTemplate.opsForValue().bitField(
            key,
            BitFieldSubCommands.create()
                    .get(BitFieldSubCommands.BitFieldType.unsigned(dayOfMonth)).valueAt(0)
    );
    if (result == null || result.isEmpty()) {
        // 没有任何签到结果
        return Result.ok(0);
    }
    //对应得十进制数
    Long num = result.get(0);
    if (num == null || num == 0) {
        return Result.ok(0);
    }
    // 6.循环遍历
    int count = 0;
    while (true) {
        // 6.1.让这个数字与1做与运算，得到数字的最后一个bit位  // 判断这个bit位是否为0
        if ((num & 1) == 0) {
            // 如果为0，说明未签到，结束
            break;
        }else {
            // 如果不为0，说明已签到，计数器+1
            count++;
        }
        // 把数字右移一位，抛弃最后一个bit位，继续下一个bit位
        num >>>= 1;
    }
    return Result.ok(count);
}
```











## 11、UV统计（HyperLogLog）

* UV：全称Unique Visitor，也叫独立访客量，是指通过互联网访问、浏览这个网页的自然人。1天内同一个用户多次访问该网站，只记录1次。
* PV：全称Page View，也叫页面访问量或点击量，用户每访问网站的一个页面，记录1次PV，用户多次打开页面，则记录多次PV。往往用来衡量网站的流量。

Redis中的HLL是基于string结构实现的























# SpringAiAlibaba

## 一、api调用大模型

调用大模型三件套（api-key,baseurl,模型名），项目构建步骤

1、建moudle

2、改pom

```yaml
<!-- DashScope ChatModel 支持（如果使用其他模型，请跳转 Spring AI 文档选择对应的 starter） -->
<dependency>
    <groupId>com.alibaba.cloud.ai</groupId>
    <artifactId>spring-ai-alibaba-starter-dashscope</artifactId>
</dependency>
<!--dashscope SDK安装-->
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>dashscope-sdk-java</artifactId>
    <version>2.22.0</version>
</dependency>
```

3、写yml

```yaml
# ====SpringAIAlibaba Config=============
spring.ai.dashscope.api-key=${aliQwen-api}
spring.ai.dashscope.base-url=https://dashscope.aliyuncs.com/compatible-mode/v1
spring.ai.dashscope.chat.options.model=qwen-plus
```

4、主启动

5、业务类

config配置类

```java
@Configuration
public class saaLLmConfig {
    /**
     * 方式1 ${}
     * 前提配置了yml中的api-key
     */
//    @Value("${spring.ai.dashscope.api-key}")
//    private String apiKey;
//
//    @Bean
//    public DashScopeApi getDashScopeApi() {
//        return DashScopeApi.builder().apiKey(apiKey).build();
//    }
    /**
     * 方式2 @Bean
     * 直接获取已配置的环境变量 System.getenv("DASHSCOPE_API_KEY")
     */
    @Bean
    public DashScopeApi getDashScopeApi() {
        return DashScopeApi.builder().apiKey(System.getenv("aliQwen-api")).build();
    }
}
```

controller类调用chatModel

```java
@RestController
public class chatModelController {

    @Resource
    private ChatModel chatModel;

    /**
     * 普通对话返回
     */
    @GetMapping("/doChat")
    public String doChat(@RequestParam(name="msg", defaultValue = "你是谁") String msg) {
        return chatModel.call(msg);
    }

    /**
     * 流式对话返回
     */
    @GetMapping("/doStreamChat")
    public Flux<String> doStreamChat(@RequestParam(name="msg", defaultValue = "你是谁") String msg) {
        return chatModel.stream(msg);
    }
}
```



## 二、ollama私有化部署和对接本地大模型

ollama调用本地大模型，**地址为localhost:11434**

1、改pom

```yaml
<!--ollama-->
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-starter-model-ollama</artifactId>
            <version>1.0.0</version>
        </dependency>
```

2、改yaml

```yaml
spring.ai.dashscope.api-key=${aliQwen-api}
spring.ai.ollama.base-url=http://localhost:11434
spring.ai.ollama.chat.model=qwen2.5:latest 
```

3、controller调用

```java
@RestController
public class OllamaController
{
    /*@Resource(name = "ollamaChatModel")
    private ChatModel chatModel;*/

    //方式2，精确调用防止冲突
    @Resource
    @Qualifier("ollamaChatModel")
    private ChatModel chatModel;

    /**
     * http://localhost:8002/ollama/chat?msg=你是谁
     * @param msg
     * @return
     */
    @GetMapping("/ollama/chat")
    public String chat(@RequestParam(name = "msg") String msg)
    {
        String result = chatModel.call(msg);
        System.out.println("---结果：" + result);
        return result;
    }

    @GetMapping("/ollama/streamchat")
    public Flux<String> streamchat(@RequestParam(name = "msg",defaultValue = "你是谁") String msg)
    {
        return chatModel.stream(msg);
    }
}

```





## 三、ChatModel 与ChatClient

**ChatModel**

对话模型(ChatModel)是底层接口，直接与具体大语言模型交互，
提供call()和stream()方法，适合简单大模型交互场景

**ChatClient**

ChatClient是高级封装，基于ChatModel构建，适合快速构建标准化复杂AI服务，支持同步和流式交互，集成多种高级功能。

**chatClient依赖chatModel,且不能自动注入，需要手动注入容器。**

config配置类

```java
@Configuration
public class saaLLmConfig {
    @Bean
    public DashScopeApi dashScopeApi() {
        return DashScopeApi.builder().apiKey(System.getenv("aliQwen-api")).build();
    }

    /**
     * chatclient以chatmodel为基础的升级
     * 不支持自动注入只支持手动注入
     */
    @Bean
    public ChatClient chantClient(ChatModel chatModel) {
        return  ChatClient.builder(chatModel).build();
    }

}
```

controller实现

```java
@RestController
public class chatClientController {

    @Resource
    private ChatClient chatClient;

	// ChatClient支持流式api
    @GetMapping("/chatClient")
    public String chat2(@RequestParam(name = "prompt",defaultValue= "2+4=") String prompt) {
        return chatClient.prompt().user(prompt).call().content();
    }
}
```

![image-20260606162330816](C:\Users\缪雨\AppData\Roaming\Typora\typora-user-images\image-20260606162330816.png)





## 四、Server-SentEvents(SSE)实现stream流式输出和多模型共存

**流式输出**：是一种逐步返回大模型生成结果的技术，生成一点返回一点，允许服务器将响应内容，分批次实时传输给客户端，而不是等待全部内容生成完毕后再一次性返回。

**Server-Sent Events (SSE) 服务器发送事件**是一种允许服务端可以持续推送数据片段（如逐词或逐句）到前端的 Web 技术。通过单向的HTTP长连接，使用一个长期存在的连接，让服务器可以主动将数据"推"给客户端，SSE是轻量级的单向通信协议，适合AI对话这类服务端主导的场景

SSE 的核心思想是：客户端发起一个请求，服务器保持这个连接打开并在有新数据时，通过这个连接将数据发送给客户端。这与传统的请求-响应模式（客户端请求一次，服务器响应一次，连接关闭）有本质区别。SSE下一代（Stream able Http）

**1、config类配置多个大模型**

```java
@Configuration
public class saaLLmConfig {

    //定义模型名称常量
    private final String DeepSeekModel = "deepseek-v3";
    private final String QwenModel = "qwen-plus";

    //生成模型注入容器
    @Bean(name = "deepSeekChatModel")
    public ChatModel deepSeekChatModel() {
        return DashScopeChatModel.builder()
                .dashScopeApi(DashScopeApi.builder().apiKey(System.getenv("aliQwen-api")).build())
                .defaultOptions(DashScopeChatOptions.builder().withModel(DeepSeekModel).build())
                .build();
    }

    @Bean(name = "qwenChatModel")
    public ChatModel qwenChatModel() {
        return DashScopeChatModel.builder()
                .dashScopeApi(DashScopeApi.builder().apiKey(System.getenv("aliQwen-api")).build())
                .defaultOptions(DashScopeChatOptions.builder().withModel(QwenModel).build())
                .build();
    }

    //chatclient
    @Bean(name = "deepseekChatClient")
    public ChatClient deepseekChatClient(@Qualifier(value = "deepSeekChatModel")  ChatModel  deepSeekChatModel) {
        return ChatClient.builder(deepSeekChatModel).build();
    }

    @Bean(name = "qwenChatClient")
    public ChatClient qwenChatClient(@Qualifier("qwenChatModel")  ChatModel qwenChatModel) {
        return ChatClient.builder(qwenChatModel).build();
    }
}
```

**2、controller调用**

```java
@RestController
public class StreamingOutputController {

    @Resource(name = "deepSeekChatModel")
    private ChatModel deepSeekChatModel;

    @Resource(name = "qwenChatModel")
    private ChatModel qwenChatModel;

    @Resource(name = "deepseekChatClient")
    private ChatClient deepSeekChatClient;

    @Resource(name = "qwenChatClient")
    private ChatClient qwenChatClient;

    //做流式输出的方法
    @GetMapping("/stream/streamOutput")
    public Flux<String> streamOutput(@RequestParam(name = "msg",defaultValue = "who are you") String msg) {
        return deepSeekChatModel.stream(msg);
    }

    @GetMapping("/stream/streamOutput2")
    public Flux<String> streamOutput2(@RequestParam(name = "msg",defaultValue = "who are you") String msg) {
        return qwenChatModel.stream(msg);
    }

    @GetMapping("/stream/streamOutput3")
    public Flux<String> streamOutput3(@RequestParam(name = "msg",defaultValue = "你是谁") String msg) {
        return deepSeekChatClient.prompt().user(msg).stream().content();
    }

    @GetMapping("/stream/streamOutput4")
    public Flux<String> streamOutput4(@RequestParam(name = "msg",defaultValue = "你是谁") String msg) {
        return qwenChatClient.prompt().user(msg).stream().content();
    }
}
```





## 五、Prompt提示词

**prompt提示词是引导ai模型生成特定输出的输入格式**，prompt设计会显著影响模型的响应。

prompt>message>string简单文本字符串提问

**prompt四大角色（4种message）**

- **System Message（系统消息）** - 告诉模型如何行为并为交互提供上下文
- **User Message（用户消息）** - 表示用户输入和与模型的交互，它们可以包含文本、图像、音频、文件和任何其他数量的多模态内容。
- **Assistant Message（助手消息）** - 模型生成的响应，包括文本内容、工具调用和元数据
- **Tool Response Message（工具响应消息）** - 表示工具调用的输出

![屏幕截图 2026-06-06 171518](C:\Users\缪雨\Pictures\Screenshots\屏幕截图 2026-06-06 171518.png)

```java
@GetMapping("/prompt/chat")
    public Flux<String> chat(@RequestParam(name = "question") String question){
        return deepseekChatClient.prompt()
                .system("你是一个小学生，不会回答小学范围外的知识")
                .user(question)
                .stream()
                .content();
    }

    @GetMapping("/prompt/chat2")
    public Flux<ChatResponse> chat2(@RequestParam(name = "question") String question){
        //系统消息
        SystemMessage  systemMessage = new SystemMessage("你是一个小学生，不会回答小学范围外的知识");
        //用户消息
        UserMessage userMessage = new UserMessage(question);
        //构建Prompt对象
        Prompt prompt1 = new Prompt(systemMessage,userMessage);
        //调用模型
        return qwenChatModel.stream(prompt1);
    }

```



## 六、提示词模板PromptTemplate

功能：

1、读取模板文件实现模板功能

2、多角色设定

3、人物设

```java
@RestController
public class PromptTemplateController
{
    @Resource(name = "deepseek")
    private ChatModel deepseekChatModel;
    @Resource(name = "qwen")
    private ChatModel qwenChatModel;

    @Resource(name = "deepseekChatClient")
    private ChatClient deepseekChatClient;
    @Resource(name = "qwenChatClient")
    private ChatClient qwenChatClient;


    @Value("classpath:/prompttemplate/atguigu-template.txt")
    private org.springframework.core.io.Resource userTemplate;

    /**
     * @Description: PromptTemplate基本使用，使用占位符设置模版 PromptTemplate
     */
    @GetMapping("/prompttemplate/chat")
    public Flux<String> chat(String topic, String output_format, String wordCount)
    {
        PromptTemplate promptTemplate = new PromptTemplate("" +
                "讲一个关于{topic}的故事" +
                "并以{output_format}格式输出，" +
                "字数在{wordCount}左右");

        // PromptTempate -> Prompt
        Prompt prompt = promptTemplate.create(Map.of(
                "topic", topic,
                "output_format",output_format,
                "wordCount",wordCount));

        return deepseekChatClient.prompt(prompt).stream().content();
    }

    /**
     * @Description: PromptTemplate读取模版文件实现模版功能
     */
    @GetMapping("/prompttemplate/chat2")
    public String chat2(String topic,String output_format)
    {
        PromptTemplate promptTemplate = new PromptTemplate(userTemplate);

        Prompt prompt = promptTemplate.create(Map.of("topic", topic, "output_format", output_format));

        return deepseekChatClient.prompt(prompt).call().content();
    }


    /**多角色设定
     * @Description:
     * 系统消息(SystemMessage)：设定AI的行为规则和功能边界(xxx助手/什么格式返回/字数控制多少)。
     * 用户消息(UserMessage)：用户的提问/主题
     */
    @GetMapping("/prompttemplate/chat3")
    public String chat3(String sysTopic, String userTopic)
    {
        // 1.SystemPromptTemplate
        SystemPromptTemplate systemPromptTemplate = new SystemPromptTemplate("你是{systemTopic}助手，只回答{systemTopic}其它无可奉告，以HTML格式的结果。");
        Message sysMessage = systemPromptTemplate.createMessage(Map.of("systemTopic", sysTopic));
        // 2.PromptTemplate
        PromptTemplate userPromptTemplate = new PromptTemplate("解释一下{userTopic}");
        Message userMessage = userPromptTemplate.createMessage(Map.of("userTopic", userTopic));
        // 3.组合【关键】 多个 Message -> Prompt
        Prompt prompt = new Prompt(List.of(sysMessage, userMessage));
        // 4.调用 LLM
        return deepseekChatClient.prompt(prompt).call().content();
    }


    /**
     * @Description: 人物角色设定，通过SystemMessage来实现人物设定，本案例用ChatModel实现
     * 设定AI为”医疗专家”时，仅回答医学相关问题
     * 设定AI为编程助手”时，专注于技术问题解答
     */
    @GetMapping("/prompttemplate/chat4")
    public String chat4(String question)
    {
        //1 系统消息
        SystemMessage systemMessage = new SystemMessage("你是一个Java编程助手，拒绝回答非技术问题。");
        //2 用户消息
        UserMessage userMessage = new UserMessage(question);
        //3 系统消息+用户消息=完整提示词
        //Prompt prompt = new Prompt(systemMessage, userMessage);
        Prompt prompt = new Prompt(List.of(systemMessage, userMessage));
        //4 调用LLM
        String result = deepseekChatModel.call(prompt).getResult().getOutput().getText();
        System.out.println(result);
        return result;
    }

    /**
     * @Description: 人物角色设定，通过SystemMessage来实现人物设定，本案例用ChatClient实现
     * 设定AI为”医疗专家”时，仅回答医学相关问题
     * 设定AI为编程助手”时，专注于技术问题解答
     */
    @GetMapping("/prompttemplate/chat5")
    public Flux<String> chat5(String question)
    {
        return deepseekChatClient.prompt()
                .system("你是一个Java编程助手，拒绝回答非技术问题。")
                .user(question)
                .stream()
                .content();
    }
}

```





## 七、格式化输出（Structured Output）

将LLM的返回结果转换为我们想要的结构化格式（如json、xml、html，java类)。

```java
@RestController
public class StructuredOutputController
{
    @Resource(name = "qwenChatClient")
    private ChatClient qwenChatClient;
    
    @GetMapping("/structuredoutput/chat2")
    public StudentRecord chat2(@RequestParam(name = "sname") String sname,
                               @RequestParam(name = "email") String email)
    {
        String stringTemplate = """
                学号1002,我叫{sname},大学专业是软件工程,邮箱{email}
                """;

        return qwenChatClient.prompt()
                .user(promptUserSpec -> promptUserSpec.text(stringTemplate)
                        .param("sname",sname)
                        .param("email",email))
                .call()
                .entity(StudentRecord.class);
    }
}

```

Spring AI Alibaba 支持两种方式控制结构化输出：

- **`outputSchema(String schema)`**: 提供 JSON schema 字符串。推荐使用 `BeanOutputConverter` 从 Java 类自动生成 schema，也可以手动提供自定义的 schema 字符串
- **`outputType(Class<?> type)`**: 提供 Java 类 - 使用 `BeanOutputConverter` 自动转换为 JSON schema（推荐方式，类型安全）

**推荐做法**：使用 `BeanOutputConverter` 生成 schema，既保证了类型安全，又实现了自动 schema 生成，代码更易维护。

1、**outputSchema` (String, 必需)**: 定义结构化输出格式的 JSON schema 字符串。可以通过 `BeanOutputConverter.getFormat()` 方法从 Java 类自动生成，也可以手动提供自定义的 schema 字符串。

**示例：**

使用 `BeanOutputConverter` 从 Java 类自动生成 JSON Schema：

```java
// 定义输出类型
public static class ContactInfo {
  private String name;
  private String email;
  private String phone;

  // Getters and Setters
  public String getName() { return name; }
  public void setName(String name) { this.name = name; }
  public String getEmail() { return email; }
  public void setEmail(String email) { this.email = email; }
  public String getPhone() { return phone; }
  public void setPhone(String phone) { this.phone = phone; }
}

// 使用 BeanOutputConverter 生成 outputSchema
BeanOutputConverter<ContactInfo> outputConverter = new BeanOutputConverter<>(ContactInfo.class);
String format = outputConverter.getFormat();

ReactAgent agent = ReactAgent.builder()
  .name("contact_extractor")
  .model(chatModel)
  .outputSchema(format)
  .build();

AssistantMessage result = agent.call(
  "从以下信息提取联系方式：张三，zhangsan@example.com，(555) 123-4567"
);

System.out.println(result.getText());
// 输出: {"name": "张三", "email": "zhangsan@example.com", "phone": "(555) 123-4567"}
```

2、`outputType` (`Class<?>`, 必需): 定义输出结构的 Java 类。该类应该是带有标准 getter 和 setter 的 POJO。

```java
// 直接使用 outputType，框架会自动处理 schema 转换
ReactAgent agent = ReactAgent.builder()
  .name("contact_extractor")
  .model(chatModel)
  .outputType(ContactInfo.class)  
  .saver(new MemorySaver())
  .build();
```

**总结**

outputType` vs `outputSchema：

- `outputType`: 更简洁，直接传入 Java 类，框架自动处理（**推荐**）
- `outputSchema`: 需要手动使用 `BeanOutputConverter` 生成 schema，或提供自定义字符串，提供更多控制





## 八、ChatMemory连续对话保存和持久化

**大模型本身不存储数据，需要将历史对话信息一次性传给大模型实现持久对话。**

```
*         模型实现对话记忆
* 要解决的两个问题：
*         1.持久化媒介（redis、数据库） RedisChatMemoryRepository
*         2.消息对话窗口，聊天记录上限问题  MessageWindowChatMemory（消息窗口聊天记忆）;
* 对话记忆实现
*         MessageChatMemoryAdvisor（顾问），每次交互中，会检索历史对话将其作为新的消息放入提示中。
```

1、用RedisChatMemoryRepository（存在reids中）实现持久化存储。

```yaml
<!--spring-ai-alibaba memory-redis-->
        <dependency>
            <groupId>com.alibaba.cloud.ai</groupId>
            <artifactId>spring-ai-alibaba-starter-memory-redis</artifactId>
        </dependency>
        <!--jedis-->
        <dependency>
            <groupId>redis.clients</groupId>
            <artifactId>jedis</artifactId>
        </dependency>
```

```java
@Configuration
public class RedisMemoryConfig {

    @Value("${spring.data.redis.host}")
    private String host;
    @Value("${spring.data.redis.port}")
    private Integer port;

    @Bean
    public RedisChatMemoryRepository redisChatMemoryRepository() {
        return RedisChatMemoryRepository.builder()
                .host(host)
                .port(port).build();
    }
}
```

2、配置实现对话记忆的chatclient

```java
@Configuration
public class saaLLmConfig {
	
    //chatmodel配置省略
    @Bean(name = "qwenChatClient")
    public ChatClient chatClient(@Qualifier("qwenchatModel") ChatModel chatModel,
                                 RedisChatMemoryRepository redisChatMemoryRepository) {
        //MessageWindowChatMemory规定了聊天记录上限问题
        MessageWindowChatMemory  messageWindowChatMemory = MessageWindowChatMemory.builder()
                        .chatMemoryRepository(redisChatMemoryRepository)
                        .maxMessages(10)
                        .build();

        //组装ChatClient，组装对话记忆顾问
        return ChatClient.builder(chatModel)
                .defaultAdvisors(MessageChatMemoryAdvisor.builder(messageWindowChatMemory).build())
                .build();
    }

}

```

3、使用chatclient

```java
@RestController
public class ChatMemoryController {

    @Resource(name = "qwenChatClient")
    private ChatClient qwenChatClient;

    @GetMapping("/chatmemory/chat")
    public String chat(String msg, String userId)
    {
        return qwenChatClient.prompt(msg)
                .advisors(advisorSpec -> advisorSpec.param(CONVERSATION_ID, userId))
                .call()
                .content();
    }

}
```





## 九、文生图和文生音

直接调用支持该类功能的大模型，无需编写配置类。

文生图

```java
@RestController
public class Text2ImageController {

    private final String IMAGE_MODEL="wanx2.1-t2i-turbo";

    @Resource
    private ImageModel imageModel;

    @GetMapping("/text2image")
    public String text2Image(@RequestParam(name = "prompt",defaultValue = "西瓜") String prompt) {
        return imageModel.call(new ImagePrompt(prompt,
                DashScopeImageOptions.builder().withModel(IMAGE_MODEL).build()))
                .getResult()
                .getOutput()
                .getUrl();
    }

}
```

文生音

DashScopeSpeechSynthesisOptions 设置模型和音色

```java
@RestController
public class Text2VoiceController {

    @Resource
    private SpeechSynthesisModel speechSynthesisModel;

    // voice model 文生音模型
    public static final String BAILIAN_VOICE_MODEL = "cosyvoice-v2";
    // voice timber 音色
    public static final String BAILIAN_VOICE_TIMBER = "longyingcui";//龙应催


    @GetMapping("/t2v/voice")
    public String voice(@RequestParam(name = "msg",defaultValue = "温馨提醒，支付宝到账100元请注意查收") String msg)
    {
        String filePath = "d:\\" + UUID.randomUUID() + ".mp3";

        //1 语音参数设置
        DashScopeSpeechSynthesisOptions options = DashScopeSpeechSynthesisOptions.builder()
                .model(BAILIAN_VOICE_MODEL)
                .voice(BAILIAN_VOICE_TIMBER)
                .build();

        //2 调用大模型语音生成对象
        SpeechSynthesisResponse response = speechSynthesisModel.call(new SpeechSynthesisPrompt(msg, options));

        //3 字节流语音转换
        ByteBuffer byteBuffer = response.getResult().getOutput().getAudio();

        //4 文件生成
        try (FileOutputStream fileOutputStream = new FileOutputStream(filePath))
        {
            fileOutputStream.write(byteBuffer.array());
        } catch (Exception e) {
            System.out.println(e.getMessage());
        }
        //5 生成路径OK
        return filePath;
    }
}
```





## 十、向量化和向量数据库

1、**Embedding(嵌入)**是指将文本、图像、视频等转换为成为向量（vectors ）的浮点数数组。嵌入数组的长度即向量的维度。

EmbeddingModel嵌入模型即嵌入过程中使用的模型。

2、**向量数据库**（Vector Store）：专门用于存储、管理、检索高维向量数据的数据库系统。其核心功能是通过高效的索引结构和相似性计算算法，支持大规模向量数据的快速查询与分析，向量数据库维度越高，查询精准度也越高，查询效果也越好。

**使用RedisStack作为向量存储**

![image-20260606203551708](C:\Users\缪雨\AppData\Roaming\Typora\typora-user-images\image-20260606203551708.png)

**RedisStack = 原生Redis + 搜索 + 图 + 时间序列 + JSON + 概率结构 + 可视化工具 + 开发框架支持**

```
<!-- 添加 Redis 向量数据库依赖 -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-vector-store-redis</artifactId>
</dependency>
```

```
spring.ai.dashscope.embedding.options.model=text-embedding-v3

spring.ai.vectorstore.redis.initialize-schema=true
spring.ai.vectorstore.redis.index-name=custom-index
spring.ai.vectorstore.redis.prefix=custom-prefix

```

```java
@RestController
@Slf4j
public class VectorStoreController {

    //文本向量化
    @Resource
    private EmbeddingModel embeddingModel;
    //向量化数据库存储（RedisStack作为向量存储）
    @Resource
    private VectorStore vectorStore;

    /**
     * 1. 文本向量化
     */
    @GetMapping("/embed")
    public EmbeddingResponse embed(String text) {

        EmbeddingResponse embeddingResponse = embeddingModel.call(new EmbeddingRequest(List.of(text)
                , DashScopeEmbeddingOptions.builder().withModel("text-embedding-v3").build()));

        System.out.println(Arrays.toString(embeddingResponse.getResult().getOutput()));
        return embeddingResponse;
    }

    /**
     * 2. 向量化数据库存储，会自动调用embeddingModel将文本转换为向量
     */
    @GetMapping("/store")
    public void add()
    {
        List<Document> documents = List.of(
                new Document("i study LLM"),
                new Document("i love java")
        );
        vectorStore.add(documents);
    }

    /**
     * 3. 向量化数据库查询
     */
    @GetMapping("/embed2vector/get")
    public List getAll(@RequestParam(name = "msg") String msg)
    {
        SearchRequest searchRequest = SearchRequest.builder()
                .query(msg)
                .topK(2)
                .build();

        List<Document> list = vectorStore.similaritySearch(searchRequest);
        System.out.println(list);
        return list;
    }

}
```

![image-20260606204848798](C:\Users\缪雨\AppData\Roaming\Typora\typora-user-images\image-20260606204848798.png)





## 十一、RAG检索增强生成（解决幻觉）

**RAG基础概念**：**检索**通过在查询时获取相关的外部知识来解决这些问题；使用特定上下文的信息来**增强** LLM 的回答。

**知识库**是用于检索的文档或结构化数据的存储库。

**RAG核心思想**：**将检索与生成集成**以产生基于事实的、上下文感知的答案。

#### 1、构建RAG系统的组件

**文档加载器和解析器**

从外部源（文件、数据库、云存储、在线平台等）摄取数据，返回标准化的文档对象。Spring AI Alibaba 提供了丰富的 [Document Reader](https://java2ai.com/integration/rag/document-readers) 和 [Document Parser](https://java2ai.com/integration/rag/document-parsers) 实现

**文本分割器**

将大型文档分解为更小的块，这些块可以单独检索并适合模型的上下文窗口。

**嵌入模型**

嵌入模型将文本转换为数字向量，使得具有相似含义的文本在向量空间中靠近在一起。

**向量存储**（**Milvus**、**Elasticsearch**、RedisStack）

用于存储和搜索嵌入的专用数据库。

**检索器**

检索器是一个接口，给定非结构化查询返回文档。

#### 2、RAG架构

（1）**两步RAG：**检索文档 → 增强上下文 → 生成答案

在**两步 RAG**中，检索步骤总是在生成步骤之前执行。这种架构简单且可预测，适合许多应用，其中检索相关文档是生成答案的明确前提。

Spring AI 提供了开箱即用的 `QuestionAnswerAdvisor` 和 **`RetrievalAugmentationAdvisor`，**简化两步 RAG 的实现。这些 Advisor **自动处理检索和上下文增强**

三种实现方式

![屏幕截图 2026-06-07 143419](C:\Users\缪雨\Pictures\Screenshots\屏幕截图 2026-06-07 143419.png)

**构建知识库示例：**

```java
// 1. 加载文档
Resource resource = new FileSystemResource("path/to/document.txt");
TextReader textReader = new TextReader(resource);
List<Document> documents = textReader.get();

// 2. 分割文档为块
TokenTextSplitter splitter = new TokenTextSplitter();
List<Document> chunks = splitter.apply(documents);

// 3. 将块添加到向量存储
vectorStore.add(chunks);

// 现在你可以使用向量存储进行检索
List<Document> results = vectorStore.similaritySearch("查询文本");
```

（2）**Agentic RAG**

**Agentic 检索增强生成（RAG）\**将检索增强生成的优势与基于 Agent 的推理相结合。Agent（由 LLM 驱动）不是在回答之前检索文档，而是逐步推理并决定在交互过程中\**何时**以及**如何**检索信息。

（3）**混合 RAG**

混合 RAG 结合了两步 RAG 和 Agentic RAG 的特点。它引入了中间步骤，如查询预处理、检索验证和生成后检查。

- 简单 FAQ → 两步 RAG

- 复杂研究任务 → Agentic RAG
- 需要质量保证 → 混合 RAG



#### 3、RAG系统构建示例

AI智能运维助手，通过提供的错误编码，给出异常解释辅助运维人员更好的定位问题和维护系统

SpringAI+阿里百炼嵌入模型text-embedding-v3+向量数据库RedisStack+DeepSeek来实现RAG功能。

**1、构建知识库**（RedisStack）

ops.txt:

00000 系统OK正确执行后的返回
A0001 用户端错误一级宏观错误码
A0100 用户注册错误二级宏观错误码
B1111 支付接口超时
C2222 Kafka消息解压严重

```java
@Configuration
public class InitVectorDatabaseConfig
{
    @Autowired
    private VectorStore vectorStore;

    @Value("classpath:ops.txt")
    private Resource sqlFile;

    @PostConstruct
    public void init()
    {
        // 1.读取文件
        TextReader textReader = new TextReader(sqlFile);
        textReader.setCharset(Charset.defaultCharset());
        
        // 2.文件转换成向量（分词）
        List<Document> list = new TokenTextSplitter().transform(textReader.read());
        // 3.写入向量数据库（Redis）,无法去重复版
        vectorStore.add(list);
}

```

**知识库数据可能重复，可以使用setnx命令去重。**

**2、advisor（RetrievalAugmentationAdvisor）调用实现两步RAG**

```java
@RestController
public class RagController
{
    @Resource(name = "qwenChatClient")
    private ChatClient chatClient;
    @Resource
    private VectorStore vectorStore;  //向量存储知识库
    
    @GetMapping("/rag4aiops")
    public Flux<String> rag(String msg)
    {
        String systemInfo = """
                你是一个运维工程师,按照给出的编码给出对应故障解释,否则回复找不到信息。
                """;

        RetrievalAugmentationAdvisor advisor = RetrievalAugmentationAdvisor.builder()
                .documentRetriever(
                        VectorStoreDocumentRetriever.builder()
                                .vectorStore(vectorStore)
                                .build()
                )
                .build();

        return chatClient.prompt()
                .system(systemInfo)
                .user(msg)
                .advisors(advisor) // RAG功能,向量数据库查询
                .stream()
                .content();
    }
} 
```







## 十一、Tool Calling工具调用

大模型支持tool calling 才行

**1、定义一个可以查询当前时间的工具类（@Tool声明）**

默认情况下工具调用的返回值会传给大模型进一步处理。

但可通过@Tool注解的returnDirect=true来启动直接返回

```java
public class DateTimeTools
{
    /**
     * 1.定义 function call（tool call）
     * 2. returnDirect
     *    true = tool直接返回不走大模型，直接给客户
     *    false = 拿到tool返回的结果，给大模型，最后由大模型回复
     */
    @Tool(description = "获取当前时间", returnDirect = false)
    public String getCurrentTime()
    {
        return LocalDateTime.now().toString();
    }
}

```

**2、ChatClient调用工具**

```java
@RestController
public class ToolCallingController{

    @Resource
    private ChatClient chatClient;

    @GetMapping("/toolcall/chat2")
    public Flux<String> chat2(@RequestParam(name = "msg",defaultValue = "你是谁现在几点") String msg)
    {
        return chatClient.prompt(msg)
                .tools(new DateTimeTools())
                .stream()
                .content();
    }
}

```





## 十二、MCP模型上下文协议

**MCP 是一个标准化协议**，使 AI 模型能够以结构化的方式与外部工具和资源交互。

#### 1、Java MCP 三层架构

![image-20260607161728142](C:\Users\缪雨\AppData\Roaming\Typora\typora-user-images\image-20260607161728142.png)

（Client/Server 层）顶层处理主要应用程序逻辑和协议操作：

- *McpClient* - 管理客户端操作和服务器连接

- *McpServer* - 处理服务器端协议操作和客户端请求

- 两个组件都利用下面的会话层进行通信管理

  

（Session 层）中间层管理通信模式并维护连接状态：

- *McpSession* - 核心会话管理接口
- *McpClientSession* - 客户端特定的会话实现
- *McpServerSession* - 服务器端特定的会话实现



（Transport 层）底层处理实际的消息传输和序列化：

- *McpTransport* - 管理 JSON-RPC 消息序列化和反序列化

- 支持多种传输实现（**STDIO、HTTP/SSE**、Streamable-HTTP 等）

- 为所有更高级别的通信提供基础

  

#### 2、Spring AI MCP 集成

Spring AI 通过以下 Spring Boot starters 提供 MCP 集成

**Server Starters**:

```yaml
<!--注意事项（重要）
            spring-ai-starter-mcp-server-webflux不能和<artifactId>spring-boot-starter-web</artifactId>依赖并存，
            否则会使用tomcat启动,而不是netty启动，从而导致mcpserver启动失败，但程序运行是正常的，mcp客户端连接不上。
        -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>
        <!--mcp-server-webflux-->
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-starter-mcp-server-webflux</artifactId>
        </dependency>

```

**Client Starters**

```yaml
<!-- 2.mcp-client 依赖 -->
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-starter-mcp-client</artifactId>
        </dependency>

```



#### 3、MCP Server实现

改pom

写yml

```
# ====mcp-server Config=============
spring.ai.mcp.server.type=async
spring.ai.mcp.server.name=customer-define-mcp-server
spring.ai.mcp.server.version=1.0.0
```

定义天气查询工具类

```java
@Service
public class WeatherService
{
    @Tool(description = "根据城市名称获取天气预报")
    public String getWeatherByCity(String city)
    {
        Map<String, String> map = Map.of(
                "北京", "11111降雨频繁，其中今天和后天雨势较强，部分地区有暴雨并伴强对流天气，需注意",
                "上海", "22222多云,15℃~27℃,南风3级，当前温度27℃。",
                "深圳", "333333多云40天，阴16天，雨30天，晴3天"
        );
        return map.getOrDefault(city, "抱歉：未查询到对应城市！");
    }
}

```

将工具方法暴露给外部 mcp client 调用

```java
@Configuration
public class McpServerConfig
{
    /**
     * 将工具方法暴露给外部 mcp client 调用
     * @param weatherService
     * @return
     */
    @Bean
    public ToolCallbackProvider weatherTools(WeatherService weatherService)
    {
        return MethodToolCallbackProvider.builder()
                .toolObjects(weatherService)
                .build();
    }
}

```

#### 4、MCP Client实现

```
# ====mcp-client Config=============
spring.ai.mcp.client.type=async
spring.ai.mcp.client.request-timeout=60s
spring.ai.mcp.client.toolcallback.enabled=true
spring.ai.mcp.client.sse.connections.mcp-server1.url=http://localhost:8014
```

创建ChatClient时利用MCP调用服务端查询天气工具类

```java
@Configuration
public class SaaLLMConfig
{
    @Bean
    public ChatClient chatClient(ChatModel chatModel, ToolCallbackProvider tools)
    {
        return ChatClient.builder(chatModel)
                .defaultToolCallbacks(tools.getToolCallbacks())  //mcp协议，配置见yml文件
                .build();
    }
}

```

调用chatclient

```java
@RestController
public class McpClientController
{
    @Resource
    private ChatClient chatClient;//使用mcp支持

    @Resource
    private ChatModel chatModel;//没有纳入tool支持，普通调用

    // http://localhost:8015/mcpclient/chat?msg=上海
    @GetMapping("/mcpclient/chat")
    public Flux<String> chat(@RequestParam(name = "msg",defaultValue = "北京") String msg)
    {
        System.out.println("使用了mcp");
        return chatClient.prompt(msg).stream().content();
    }

    @RequestMapping("/mcpclient/chat2")
    public Flux<String> chat2(@RequestParam(name = "msg",defaultValue = "北京") String msg)
    {
        System.out.println("未使用mcp");
        return chatModel.stream(msg);
    }
}

```



#### 5、调用互联网通用MCP服务（百度地图）

1、需要配置百度地图server

在resources类路径下增加mcp-server.json5文件即服务端配置文件。

```json
{
  "mcpServers":
  {
    "baidu-map":
    {
      "command": "cmd",
      "args": ["/c", "npx", "-y", "@baidumap/mcp-server-baidu-map"],
      "env":  {"BAIDU_MAP_API_KEY": "jfzXuCZe7niYUvdXlVCzgLwk2dmUxcfq"}
    }
  }
}
```

yaml中指定客户端对应的服务端配置为该文件

```yaml
# ====mcp-client Config=============
spring.ai.mcp.client.toolcallback.enabled=true
spring.ai.mcp.client.stdio.servers-configuration=classpath:/mcp-server.json
```

2、创建ChatClient 调用百度地图工具

```java
@Configuration
public class SaaLLMConfig
{
    @Bean
    public ChatClient chatClient(ChatModel chatModel, ToolCallbackProvider tools)
    {
        return ChatClient.builder(chatModel)
                //mcp协议，配置见yml文件，此处只赋能给ChatClient对象
                .defaultToolCallbacks(tools.getToolCallbacks())
                .build();
    }
}

```

3、chatclient 可自动调用百度地图工具

```java
@RestController
public class McpClientCallBaiDuMcpController
{
    @Resource 
    private ChatClient chatClient; //添加了MCP调用能力

    @Resource
    private ChatModel chatModel; //没有添加MCP调用能力

    /**
     * 添加了MCP调用能力
     */
    @GetMapping("/mcp/chat")
    public Flux<String> chat(String msg)
    {
        return chatClient.prompt(msg).stream().content();
    }



```







## 十三、阿里云百炼平台知识库和工作流使用

#### 1、知识库使用

DashScopeApi增加业务空间id

```java
@Configuration
public class DashScopeConfig {

    @Value("${spring.ai.dashscope.api-key}")
    private String apiKey;

    @Bean
    public DashScopeApi dashScopeApi(){
        return DashScopeApi.builder()
                .apiKey(apiKey)
                .workSpaceId("ws-fjqpo6rw5szixohr")  // 这个是阿里云百炼业务空间id
                .build();
    }

    @Bean
    public ChatClient chatClient(ChatModel chatModel){
        return ChatClient.builder(chatModel).build();
    }
}
```

根据知识库名称调用

```java
@RestController
public class YunRAGController {

    @Resource
    private ChatClient chatClient;

    @Resource
    private DashScopeApi dashScopeApi;


    @GetMapping("/bailian/rag/chat")
    public Flux<String> chat(@RequestParam(name = "msg",defaultValue = "00000错误信息") String msg)
    {
        //1 RetrieverOptions参数配置
        DashScopeDocumentRetrieverOptions documentRetrieverOptions = DashScopeDocumentRetrieverOptions.builder()
                .withIndexName("ops") // 知识库名称
                .build();

        //2 百炼平台RAG知识库构建器
        DocumentRetriever retriever = new DashScopeDocumentRetriever(dashScopeApi,documentRetrieverOptions);

        return chatClient.prompt()
                .user(msg)
                .advisors(new DocumentRetrievalAdvisor(retriever))
                .stream()
                .content();
    }
}
```



#### 2、工作流调用

在百炼平台创建工作流，在yml中配置工作流id

```
spring.ai.dashscope.agent.options.app-id=29f1986913a04be48a8801369cda4eb6
```

同样需要在DashScopeApi增加业务空间id

```java
@RestController
public class MenuCallAgentController
{
    // 百炼平台的appid
    @Value("${spring.ai.dashscope.agent.options.app-id}")
    private String APPID;
    // 百炼云端智能体调用对象只能通过构造方法注入
    private DashScopeAgent agent;
    //构造方法注入，创建百炼云端智能体对象
    public MenuCallAgentController(DashScopeAgentApi agentApi)
    {
        this.agent = new DashScopeAgent(agentApi);
    }

   
    @GetMapping("/eatAgent")
    public String eatAgent(@RequestParam(name = "topic",defaultValue = "今天中午吃什么") String topic)
    {
        DashScopeAgentOptions options = DashScopeAgentOptions.builder().withAppId(APPID).build();

        Prompt prompt = new Prompt(topic, options);

        return agent.call(prompt).getResult().getOutput().getText();
    }
}
```











## （======）面试可能问到的

#### 1. RAG + Redis Stack

> 向量数据怎么生成的？Embedding模型具体是哪个？检索准确率如何评估？

**向量数据怎么生成**：我把房源信息（标题、描述、位置标签等）存储到文本文件中，先将文本文件分割成块，再调用嵌入模型或直接调用向量存储（vectorstore.add（））将文本转换为向量数据，存入Redis Stack向量数据库。

在redis中的向量数据的索引：**在yml文件中配置是否开启自动索引生成，索引名称，索引前缀。**

![image-20260616195406947](C:\Users\缪雨\AppData\Roaming\Typora\typora-user-images\image-20260616195406947.png)

**Embedding模型：**text-embedding-v3，中文语义理解好。

**RAG如何检索向量数据库**：这些  （**RetrievalAugmentationAdvisor**）Advisor **自动处理检索和上下文增强**

**向量数据库查询**：

```java
 SearchRequest searchRequest = SearchRequest.builder()
                .query(msg)
                .topK(3)		//查询3个最相似的向量数据
                .build();
        List<Document> list = vectorStore.similaritySearch(searchRequest);
```

**检索准确率方法**：构造一组不同意图的查询数据（100条），人工标注预期查询出的房源。调用agent去执行这组查询，记录结果。

计算 **召回率**（相关房源是否被检索出来）。

（也可以构造混淆矩阵分析数据，计算出召回率，准确率，精确率。）



#### 2. MCP + 百度地图

> MCP协议你理解到什么程度？百度地图MCP服务具体返回了什么数据？如何与LLM集成？

**MCP 是一个标准化协议**，使 AI 模型能够以结构化的方式与外部工具和资源交互。

**如何与LLM集成：** 导入依赖，配置yml文件，第三方服务端所需配置文件，ToolCallbackProvider类作为chatclient创建参数，即可使用mcp.

```java
@Configuration
public class SaaLLMConfig
{
    @Bean
    public ChatClient chatClient(ChatModel chatModel, ToolCallbackProvider tools)
    {
        return ChatClient.builder(chatModel)
                .defaultToolCallbacks(tools.getToolCallbacks())  //mcp协议，配置见yml文件
                .build();
    }
}
```

**服务返回什么数据**：



#### 3、对话记忆持久存储

**MessageChatMemoryAdvisor**

```
模型实现对话记忆
* 要解决的两个问题：
*         1.持久化媒介（redis、数据库） RedisChatMemoryRepository
*         2.消息对话窗口，聊天记录上限问题  MessageWindowChatMemory（消息窗口聊天记忆）;
* 对话记忆实现
*         MessageChatMemoryAdvisor（顾问），每次交互中，会检索历史对话将其作为新的消息放入提示中。
```

```java
return qwenChatClient.prompt(msg)
    			//CONVERSATION_ID, userId 组成redis中存储用户历史对话的key
                .advisors(advisorSpec -> advisorSpec.param(CONVERSATION_ID, userId))
                .call()
                .content();
```



**value**: 历史消息以hash类型存储，

**key:** CONVERSATION_ID+userId

![image-20260616195714259](C:\Users\缪雨\AppData\Roaming\Typora\typora-user-images\image-20260616195714259.png)



**上下文窗口有限（200K-1MB），可以在消息窗口被使用60%时，压缩历史消息，仅保留关键信息。**

**可以设置消息淘汰策略，一周未使用删除**















# Spring Cloud Alibaba

- Nacos：注册中心+配置中心，实现服务自动注册与发现，以及动态配置管理。
- Sentinel（服务保护）：流量控制、熔断降级、系统保护，保障服务的高可用。
- RocketMQ：分布式消息中间件，实现异步解耦、削峰填谷、事件驱动。
- Seata：分布式事务 解决方案，提供AT、TCC等事务模式，保证跨服务的数据一致性

Spring Cloud

 **OpenFeign**（声明式远程调用）：用接口+注解的方式调用远程服务，隐藏 HTTP 细节

**Gateway**（网关）：请求入口怎么统一处理，基于 WebFlux，性能更高，支持限流、路由等



### 一、Nacos

**nacos启动**：进入bin目录，cmd执行startup.cmd -m standalone ，单机模式启动。

#### 1、服务注册与服务发现

**服务注册流程：**导入依赖，yml配置nacos地址，启动模块，访问注册中心观查注册结果。

![image-20260617164318270](C:\Users\缪雨\AppData\Roaming\Typora\typora-user-images\image-20260617164318270.png)

**服务发现流程：**

![image-20260617164305842](C:\Users\缪雨\AppData\Roaming\Typora\typora-user-images\image-20260617164305842.png)



```java
@SpringBootTest
public class ProductApplicationTest {

    @Autowired
    DiscoveryClient discoveryClient;    //服务发现api,两个均可，获取我们注册的所有服务，订单服务，商品服务

    @Test
    public void testDiscoveryClient() {
        //获取服务列表
        List<String> services = discoveryClient.getServices();
        for (String service : services) {
            System.out.println(service);
            //获取ip和端口
            List<ServiceInstance> instances = discoveryClient.getInstances(service);
            for (ServiceInstance instance : instances) {
                System.out.println("ip: "+instance.getHost() + "   port:" + instance.getPort());
            }
        }
    }
}

```



#### 2、远程调用

![image-20260617170419150](C:\Users\缪雨\AppData\Roaming\Typora\typora-user-images\image-20260617170419150.png)

**流程：**

| 内容  | 流程                 | 核心                              |
| ----- | -------------------- | --------------------------------- |
| 步骤1 | 引入负载均衡依赖     | spring-cloud-starter-loadbalancer |
| 步骤2 | 测试负载均衡API      | LoadBalancerClient                |
| 步骤3 | 测试远程调用模板类   | RestTemplate                      |
| 步骤4 | 测试负载均衡远程调用 | @LoadBalanced                     |

**代码示例：三种不同方式order服务调用product服务**

**最佳实践：** 基于注解实现负载均衡发送请求

RestTemplate模板类用于发送请求

```java
@Configuration
public class OrderConfig {

    @Bean			
    @LoadBalanced   //负载均衡远程调用
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}
```

```java
	//原始
    private Product remoteProductList(Long productId) {
        //获取商品服务的全部实例只能选择一个调用
        List<ServiceInstance> instances = discoveryClient.getInstances("service-product");
        ServiceInstance instance = instances.get(0);
        //远程url
        String url = "http://" + instance.getHost() + ":" + instance.getPort() + "/product/"+productId;
        log.info("远程url:{}", url);
        //给远程发送请求
        Product product = restTemplate.getForObject(url, Product.class);
        return product;
    }

    //进阶：  实现负载均衡发送请求
    private Product remoteProductListLoadBalancer(Long productId) {
        //循环调用service-product的一个实例
        ServiceInstance instance = loadBalancerClient.choose("service-product");
        //远程url
        String url = "http://" + instance.getHost() + ":" + instance.getPort() + "/product/"+productId;
        log.info("远程url:{}", url);
        //给远程发送请求
        Product product = restTemplate.getForObject(url, Product.class);
        return product;
    }


    //最终版本：  基于注解实现负载均衡发送请求，@LoadBalanced配置restTemplate
    private Product remoteProductListLoadBalancerAnnotation(Long productId) {
        String url ="http://service-product/product/"+productId;
        //给远程发送请求
        Product product = restTemplate.getForObject(url, Product.class);
        return product;
    }

```



#### 3、配置中心

**（1)基本使用**

启动nacos, 导入依赖，配置yml，配置中心创建数据集（就是yml类似配置文件）

```java
<dependency>
	<groupId>com.alibaba.cloud</groupId>
	<artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
</dependency>
```

```java
spring.cloud.nacos.server-addr=127.0.0.1:8848
spring.config.import=nacos:service-order.properties   //导入配置中心创建的数据集
```

**（2）动态刷新获取数据集内容**

第一种方式：@Value(“${xx}”) 获取配置 + @RefreshScope（类上） 实现自动刷新

**第二种方式：**@ConfigurationProperties 无感自动刷新

通过NacosConfigManager 监听配置变化



**（3）Nacos中的数据集 和 application.properties 有相同的 配置项，哪个生效？**

先导入优先，外部优先。从nacos导入的数据集优先。



##### （4）数据隔离

难点

• 区分多套环境  ——命名空间

• 区分多种微服务 ——group组

• 区分多种配置 ——数据集

• 按需加载配置 ——spring.config.activate.on-profile区分多套环境，spring.profiles.active激活不同环境

![屏幕截图 2026-06-17 190759](C:\Users\缪雨\Pictures\Screenshots\屏幕截图 2026-06-17 190759.png)

```yaml
#标准使用方法
spring:
  profiles:
    active: test
  cloud:
    nacos:
      config:
        namespace: ${spring.profiles.active:public} #动态命名空间选择环境 默认public
        
---   #---表示多层配置
spring:
  config:
    import:
      - nacos:common.properties?group=order
      - nacos:database.properties?group=order
    activate:
      on-profile: dev
      
---
spring:
  config:
    import:
      - nacos:common.properties?group=order
      - nacos:database.properties?group=order
      - nacos:empty.properties?group=order
    activate:
      on-profile: test
```





### 二、OpenFeign(远程调用，自动负载均衡)

```java
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

配置feignClient接口

• 指定远程地址：@FeignClient

• 指定请求方式：@GetMapping、@PostMapping、@DeleteMapping ... 

• 指定携带数据：@RequestHeader、@RequestParam、@RequestBody ... 

• 指定结果返回：响应模型

**1、调用业务API**

直接复制对方Controller签名即可

```java
@FeignClient(value = "service-product",fallback = ProductFeignFallBack.class) // 调用服务名为service-product的服务   fallback指定兜底回调的feignClient接口实现类
public interface GetProductFeign {

    //mvc注解的两套使用逻辑
    //1、标注在controller上，是接收这样的请求
    //2、标注在FeignClient上，是发送这样的请求
    @GetMapping("/api/product/product/{id}")
    Product getProductById(@PathVariable("id") Long id);
}
```

**2、调用第三方API**

根据接口文档确定请求如何发



**3、进阶使用-日志**（注入容器即可用下面也是）

日志记录远程调用的详细信息

```
logging:
  level:
    order.feign.GetProductFeign: DEBUG
```

```java
@Configuration
public class OrderConfig {

    //Feign的日志级别
    @Bean
    Logger.Level feignLoggerLevel() {
        return Logger.Level.FULL;
    }
}
```

**4、进阶使用-超时控制与重试机制**

connectTimeout—连接超时， readTimeout—读取超时，

![image-20260617200326622](C:\Users\缪雨\AppData\Roaming\Typora\typora-user-images\image-20260617200326622.png)

```java
@Configuration
public class OrderConfig {

    //feign重试机制，即可重试
    @Bean
    Retryer retryer() {
        return new Retryer.Default();
    }

}
```

**5、进阶使用-拦截器**

RequestInterceptor、ResponseInterceptor。不是传统的自定义拦截器HandlerInteceptor

```java
@Component
public class XTokenRequestInterceptor implements RequestInterceptor {
    /**
     * 请求拦截器
     * @param requestTemplate  请求模板
     */
    @Override
    public void apply(RequestTemplate requestTemplate) {
        System.out.println("XTokenRequestInterceptor------执行");
        requestTemplate.header("X-Token", "123456");
    }
}
```

**6、进阶使用-Fallback兜底返回**

注意：此功能需要**整合 Sentinel** 才能实现

```
# 开启feign对sentinel的支持（服务熔断）
feign:
  sentinel:
    enabled: true
```

```java
@Component
public class ProductFeignFallBack implements GetProductFeign {
    @Override
    public Product getProductById(Long id) {
        System.out.println("兜底回调------");
        //兜底数据
        Product product = new Product();
        return product;
    }
}
```





### 三、Sentinel（服务保护）

##### **1、核心概念**

**定义资源：**

• 主流框架自动适配（Web Servlet、Dubbo、Spring Cloud、gRPC、Spring WebFlux、Reactor）；

所有Web接口均为资源

• 编程式：SphU API

• 声明式：@SentinelResource

**定义规则：**

• 流量控制（FlowRule）

• 熔断降级（DegradeRule）

• 系统保护（SystemRule）

• 来源访问控制（AuthorityRule）

• 热点参数（ParamFlowRule）



**2、基本使用**

下载sentinel控制台，通过 **java -jar sentinel-dashboard-1.8.8.jar** 启动 ，访问控制台Localhost:8080

项目引入依赖

```
<!--  服务熔断  -->
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
</dependency>
```

```
spring.cloud.sentinel.transport.dashboard=localhost:8080
#项目启动立即初始化sentinel，不用等第一次请求
spring.cloud.sentinel.eager=true   
```



##### 3、异常处理

![image-20260617204712798](C:\Users\缪雨\AppData\Roaming\Typora\typora-user-images\image-20260617204712798.png)

（1）web接口异常，返回自定义信息

```java
@Component
public class MyBlockExceptionHandler implements BlockExceptionHandler {
    private ObjectMapper objectMapper = new ObjectMapper();
    @Override
    public void handle(HttpServletRequest request, HttpServletResponse response,
                       String resourceName, BlockException e) throws Exception {
        response.setContentType("application/json;charset=utf-8");
        PrintWriter writer = response.getWriter();
        R r = R.error(500, resourceName+"被sentinel限制了, 原因："+e.getClass());
        String json = objectMapper.writeValueAsString(r);
        writer.write(json);
        writer.flush();
        writer.close();
    }
}
```

（2）SentinelResource定义的资源出现异常，blockHandler属性指定兜底回调方法，否则由全局异常处理

```java
@SentinelResource(value = "createOrder", blockHandler = "createOrderFallback")
@Override
public Order createOrder(Long userId, Long productId) {
    Product product = getProductFeign.getProductById(productId);
    //TODO 总金额
    //TODO 远程查寻商品列表
    return order;
}


//兜底回调
public Order createOrderFallback(Long userId, Long productId, BlockException e) {
    Order order = new Order();
    return order;
}
```

(3) openfeign远程调用，抛出异常同样兜底回调处理，没有则全局异常来处理

```java
@FeignClient(value = "service-product",fallback = ProductFeignFallBack.class) // 调用服务名为service-product的服务   fallback指定兜底回调的feignClient接口实现类
```





##### 4、流控规则

**流量控制：**限制多余请求，从而保护系统资源不被耗尽

![image-20260618151050853](C:\Users\缪雨\AppData\Roaming\Typora\typora-user-images\image-20260618151050853.png)

**（1）阈值类型：**

QPS：统计每秒请求数

并发线程数：统计并发线程数

**（2）是否集群**
默认否：阈值为5，整个集群ops=5

是：阈值为5,每一个服务ops=5，如果有3台服务器，ops=15

  **(3）流控模式**

![image-20260618151639660](C:\Users\缪雨\AppData\Roaming\Typora\typora-user-images\image-20260618151639660.png)

链路策略

限流资源B,实际限流的是资源C。

```
spring.cloud.sentinel.web-context-unify=false 
#关闭上下文统一，实现资源流控链路模式
```

关联策略

有两个资源readDB,writeDB

对readDB限流ops=1，但关联了writeDB； readDB看似限流实则没有，writeDB被限流。只有大量并发访问writeDB时，readDB才被限流。

**（4）流控效果**

**注意：只有快速失败支持流控模式（直接、关联、链路）的设置**

**快速失败：**qps=1，只允许一个请求成功，其他多余请求直接丢弃。

**Warm Up:** 预热/冷启动，设置预热周期=3s,qps=10.预热阶段qps逐渐增加，最后稳定为10,以该qps发送请求。

**排队等待：**qps=2,timeout=20s。多余请求会排队等待，等待超过超时时间会被丢弃。





##### 5、熔断规则

**描述：**切断不稳定调用，快速返回不积压，避免雪崩效应

**最佳实践：**熔断降级作为保护自身的手段，通常在客户端（调用端）进行配置。

依靠**断路器**实现

<img src="C:\Users\缪雨\AppData\Roaming\Typora\typora-user-images\image-20260618155123373.png" alt="image-20260618155123373" style="zoom:67%;" />

![image-20260618160056296](C:\Users\缪雨\AppData\Roaming\Typora\typora-user-images\image-20260618160056296.png)

熔断策略：慢调用比例，异常比例，异常数。

底层执行逻辑都是上图的断路器工作原理。





##### 6、热点参数

参数级别限流。

需求1：每个用户秒杀 QPS 不得超过 1（秒杀下单 userId 级别）

**效果：携带此参数的参与流控，不携带不流控**

需求2：6号用户是vvip，不限制QPS（例外情况）

需求3：666号是下架商品，不允许访问



<img src="C:\Users\缪雨\AppData\Roaming\Typora\typora-user-images\image-20260618161215413.png" alt="image-20260618161215413" style="zoom: 67%;" />









### 四、GateWay网关

网关是响应式服务，服务器是Netty

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>

<dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>
```

**创建网关**：到依赖，@EnableDiscoveryClient，启动。

#### 1、路由

• 1. 客户端发送 /api/order/** 转到 service-order

• 2. 客户端发送 /api/product/** 转到 service-product

• 3. 以上转发有负载均衡效果

```
<!--  再导入负载均衡   -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-loadbalancer</artifactId>
</dependency>
```

**yml配置文件—路由规则配置**

单独路由配置文件：application-route.yml

```yaml
spring:
  profiles:
    include: route
```

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: order-route
          uri: lb://service-order  # 负载均衡
          predicates:              # 断言
            - Path=/api/order/**
          filters:
            - OnceToken=X-Token,uuid  # 自定义的过滤器工厂
          order: 0      # 路由的优先级
        - id: product-route
          uri: lb://service-product
          predicates:
            - Path=/api/product/**
          order: 1
```

#### 2、断言

已有断言怎么写

<img src="C:\Users\缪雨\AppData\Roaming\Typora\typora-user-images\image-20260618171609251.png" alt="image-20260618171609251" style="zoom: 80%;" />

![image-20260618185956173](C:\Users\缪雨\AppData\Roaming\Typora\typora-user-images\image-20260618185956173.png)

自定义断言

![image-20260618191029178](C:\Users\缪雨\AppData\Roaming\Typora\typora-user-images\image-20260618191029178.png)

已有断言都是通过**继承AbstractRoutePredicateFactory类**定义的。我们只需仿照写xxxRoutePredicateFactory类继承并重写方法即可。





#### 3、过滤器

基本使用：**路径重写**

不用在每一个服务上定义该服务的统一前缀路径。直接在网关配置。还有很多其他参数见官网。

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: order-route
          uri: lb://service-order  # 负载均衡
          predicates:              # 断言
            - Path=/api/order/**
          filters:
            - RewritePath=/api/order/(?<segment>.*)   #路径重写
            - AddResponseHeader=X-Response-Default-Foo, Default-Bar  # 添加响应头
            - OnceToken=X-Token,uuid  # 自定义的过滤器工厂
```

![image-20260618192441363](C:\Users\缪雨\AppData\Roaming\Typora\typora-user-images\image-20260618192441363.png)

**default-filters**所有路由均生效

```yaml
spring:
  cloud:
    gateway:
      default-filters:
        - name: AddResponseHeader
          args:
            name: X-Response-Default-Foo
```

**全局GlobalFilter**所有路由均生效

```java
@Component
@Slf4j
// 自定义全局过滤器，实现GlobalFilter接口,在gateway中全局生效，
public class RTGlobalFilter implements GlobalFilter {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();
        ServerHttpResponse response = exchange.getResponse();
        String uri = request.getURI().toString();
        long start = System.currentTimeMillis();
        log.info("请求地址：{}, 请求开始时间：{}", uri, start);
        //==========以上是前置逻辑==========

        Mono<Void> voidMono = chain.filter(exchange).doFinally(result -> {
            // ==========后置逻辑==========
            long end = System.currentTimeMillis();
            log.info("请求：{}, 请求结束时间：{},耗时:{}ms", uri, end, end - start);
        });
        return voidMono;
    }
}
```

**自定义过滤器工厂**

参考已有的仿写

**分布式系统跨域**

```yaml
spring:
  cloud:
    gateway:
      globalcors:
        cors-configurations:
          '[/**]':
            allowedOriginPatterns: "*"
            allowedMethods:	"*"
            allowedHeaders: "*"
```







### 五、Seata（分布式事务）

![image-20260619140117947](C:\Users\缪雨\AppData\Roaming\Typora\typora-user-images\image-20260619140117947.png)

##### **步骤**



**1、@Transactional**注解，需标注**@EnableTransactionManagement**开启事务管理。mybatis框架下无需手动标注，由框架自动开启。

**2、seata启动：**进入seata的 bin目录cmd执行seata-server.bat

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-seata</artifactId>
</dependency>
```

**3、给每个引入seata的微服务配置file.conf文件**

指定seata的地址

```
service {
  #transaction service group mapping
  vgroupMapping.default_tx_group = "default"
  #only support when registry.type=file, please don't set multiple addresses
  default.grouplist = "127.0.0.1:8091"
  #degrade, current not support
  enableDegrade = false
  #disable seata
  disableGlobalTransaction = false
}
```

**4、seata事务管理要生效须在最大的业务入口上标注全局事务**

此处为business服务（TM）

```java
@GlobalTransactional  // 分布式全局事务
@Override
public void purchase(String userId, String commodityCode, int orderCount) {
    //1. 扣减库存
    storageFeignClient.deduct(commodityCode, orderCount);

    //2. 创建订单
    orderFeignClient.create(userId, commodityCode, orderCount);
}
```

其他小微服务（RM）

**@Transactional****注解，标注**@EnableTransactionManagement**开启事务管理。



##### 原理-二阶提交协议

**一阶：本地事务提交**

（业务数据 + undo_log）

提交修改后的业务数据；undo_log中保存了数据库的前镜像和后镜像。前镜像为未修改前的数据，后镜像为修改后的数据。

**二阶：成功 + 失败**

成功：所有人删除undo_log

失败：所有人拿到自己的前镜像，恢复，删除undo_log

**全局锁（数据级别）**

保证数据一致性，分布式事务正确执行。



四种事务模式：AT  TCC  SAGA   XA

**流程图架构图在**"D:\百度网盘Download\SpringCloud尚硅谷2025\资料\SpringCloud-图例.svg"









# 前端工程化速通补充

### **2.2 npm包管理**

npm init :执行命令生成package.json文件。这是npm管理项目的文件。

npm install xxx: 执行命令安装所需工具，会生成package-lock.json文件（依赖的详细说明） 和 **node_modules**目录存放下载的依赖。

传输项目时该目录删除不用传输。

npm run xxx: 该命令启动项目，也可以启动package.json文件中定义的script脚本命令。

node main.js ：node 命令直接运行js文件。



### **4.4.5响应式数据**

```
最佳实践：
1、我该用哪个函数？ 答：可以 ref() 一把梭、也可以 ref()包装基本数据、reactive()包装对象
2、使用 const 声明响应式常量
3、响应式数据具有深层响应式特性（属性.属性.属性 也都是响应式的）
```

**监听响应式数据**

**watch**监听一个响应式数据

**watchEffect监听所有**响应式数据

```java
watchEffect(() => {
  if (num.value > 3) {
    alert("超出限购数量")
    num.value = 3;
  }

  if (car.price > 11000) {
    alert("太贵了")
    car.price = 11000;
  }
})
```



### **vue组件生命周期**（Hook钩子）

```java
//更新： 前内容未变，数据变了
onBeforeUpdate(()=>{
  console.log("更新前：count",count.value)
  console.log("更新前：btn内容",document.getElementById("btn01").innerHTML)
})

onUpdated(()=>{
  console.log("更新完：count",count.value)
  console.log("更新完：btn内容",document.getElementById("btn01").innerHTML)
})
```



### **组件传值**

父传子：v-bind(:)属性绑定传数据，defineProps接收属性。

子传父：子定义事件defineEmits，父感知事件接收事件值。

```java
//2、使用emit: 定义事件    Son
let emits = defineEmits(['buy']);
function buy(){
  // props.money -= 5;
  emits('buy',-5);
}
```

```java
<script>		//Father
function moneyMinis(arg){
  // alert("感知到儿子买了棒棒糖"+arg)
  data.money+=arg;
}
</script>

<template>
	<Son :books="data.books" :money="data.money" @buy="moneyMinis"/>
</template>
```

**兄弟传值**：子传父，父再传子



### **slot插槽**

子组件在父组件中看作一个标签，**标签中的内容会显示在子组件的所有插槽中**

```java
<Son v-bind="data">    //父
	哈哈
</Son>
```

```java
<h3>    //子
  <slot name="title">
    哈哈Son    //插槽默认值，父未传则为默认值
  </slot>
</h3>
<button @click="buy">   //标签值也可用插槽
    <slot name="btn"/>
</button>
```

**具名插槽**：父数据精确传递到一个子插槽

```Java
<Son v-bind="data">
  <template v-slot:title>   //父用template模板，v-slot 简写 #
    哈哈SonSon
  </template>

  <template #btn>
    买飞机
  </template>
</Son>
```

```Java
<h3>    //子
  <slot name="title">
    哈哈Son    //插槽默认值，父未传则为默认值
  </slot>
</h3>
<button @click="buy">   //标签值也可用插槽
    <slot name="btn"/>
</button>
```





### vue-router路由表

编写⼀个新的组件，测试路由功能；步骤如下：

**整合vue-router   npm install vue-router@4**

\1. 编写 router/index.js ⽂件

\2. 配置路由表信息

\3. 创建路由器并导出

\4. 在 main.js 中使⽤路由

\5. 在⻚⾯使⽤ router-link router-view 完成路由功能

```java
//1、配置路由表
const routes = [              
    {path: '/', component: Home},
    {path: '/hello', component: Hello}
]

//2、创建路由器
const router = createRouter({
    // 4. 内部提供了 history 模式的实现。为了简单起见，我们在这里使用 hash 模式。
    history: createWebHashHistory(),
    routes,
});

//3、导出路由器
export default router;
```

```java
let app = createApp(App);

//1、使用router
app.use(router)
app.mount('#app');
```

```java
<router-link to="/">首页</router-link>
```



**路径参数**

```java
// :变量名 接受动态参数；这个成为路径参数
const routes = [              
    {path: '/', component: Home},
    {path: '/hello/:id', component: Hello}
]
```

```java
<h1>我是哈哈 {{$route.params.id}}</h1>   //获得具体参数
```



### 嵌套路由

路由表中定义嵌套路由

```Java
const routes = [
 {
 	path: '/user/:id',
 	component: User,
 	children: [
 	{
 		// 当 /user/:id/profile 匹配成功
 		// UserProfile 将被渲染到 User 的 <router-view> 内部
 		path: 'profile',
 		component: UserProfile,
 	},
 	{
 		// 当 /user/:id/posts 匹配成功
 		// UserPosts 将被渲染到 User 的 <router-view> 内部
 		path: 'posts',
 		component: UserPosts,
 	},
 	],
 },
]
```

User.vue组件

```Java

<router-link to="/user/007/profile">个人信息</router-link>
<router-link to="/user/007/posts">我的邮件</router-link>
<div style="border: 5px solid red">
  <router-view></router-view>    //展示路由到的子路由的页面信息
</div>
```



**useRouter：路由器**

拿到路由器；可以控制跳转、回退等。

```
router.push('/')  //跳转
router.go(1);   //前进
router.go(-1);   //回退
```



### 路径传参

params 参数 query参数

**路径跳转方式：**

1、<router-link to="/user/007/posts">我的邮件</router-link>

2、const router = useRouter();

​	router.push({ path: '/register', query: { plan: 'private' } })

**获取参数**

1、{{$route.params.id}}

2、useRoute()对象获取路由参数， {{route.params.id}}。

```java
<script setup>
import {useRoute} from 'vue-router'
let route = useRoute();
</script>

<template>
<h1>我是哈哈 {{$route.params.id}}==> {{$route.params.name}} ==> {{$route.params.age}}</h1>
  <div>
    {{route.params.id}} <br/>
    {{route.params.name}} <br/>
    {{route.params.age}} <br/>
  </div>
</template>
```





### 导航守卫

类似后端拦截器。控制路由器导航逻辑

```java
//2、创建路由器
const router = createRouter({
    // 4. 内部提供了 history 模式的实现。为了简单起见，我们在这里使用 hash 模式。
    history: createWebHashHistory(),
    routes,
});


router.beforeEach(async (to, from) => {
    console.log("to",to)
    console.log("from",from)

    // await fetch();
    // ...
    //1、返回 false：取消导航
    //2、返回true，不返回：继续导航
    //3、返回'路径'：跳转到指定页面
    if(to.path==='/hello'){
        console.log("禁止访问")
        return "/"
    }
})
```

