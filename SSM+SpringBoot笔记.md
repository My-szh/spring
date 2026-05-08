

# Spring



## IOC



### 一、注册组件

```
1. bean组件创建时机：ioc容器启动过程中创建组件对象
* 组件具有单实例特性：组件默认是单实例的，每次获取直接从容器中取，不用重新创建，容器会提前创建
*
* 组件：框架的底层配置
*     配置文件（.yaml .properties） ：指定配置
*     配置类：分类管理组件的配置，配置类也是容器中的一种组件。
```

```
2. spring 为我们提供了mvc分层注解
*      @Controller ：控制器
*      @Service ：业务逻辑层
*      @Repository ：数据访问层
*      @Component ：组件
* 这些注解要生效的默认条件是：必须位于主启动类的包下，或者主启动类所在包下的子包下
*      可以改变默认条件，通过@ComponentScan指定要扫描的包名
```

```
3.导入第三方组件
*      1). @Bean: 自己new，再放入容器中
*      2). @Import: 导入第三方组件
```

```
4.@Scope 调整组件的作用域
*       1). @Scope("prototype") ：多实例
*           ioc容器创建时不会创建多实例对象
*           组件每次被调用时才创建一个实例对象
*       2). @Scope("singleton") ：单实例（默认）
*           ioc容器创建时创建一个实例对象，以后每次获取都从容器中拿
*       3). @Scope("request") ：webmvc作用域，一次请求创建一个实例
*       4). @Scope("session") ：webmvc作用域，一次会话创建一个实例
```

```
5.@Lazy 懒加载: 作用于单实例对象，使用后，单实例对象不会提前创建，获取时才创建
```

```
6.FactoryBean: 工厂Bean
*      场景：当创建对象比较复杂，或者需要通过其他组件创建对象时，使用工厂Bean
*      组件名字就是我们类名（首字母自动转小写）
```

```
7.@Conditional:条件注册
*      满足条件，创建这个组件
```

### 二、注入组件（DI）

```
*  1.@Autowired:自动注入   (先按类型，再按名字)
*  2.@Qualifier:精确指定注入的组件
*  3.@Primary:指定注入的组件，当有多个同类型组件时，默认使用这个组件
*  4.@Resource:自动注入
*       *******面试题********
*         @Autowired和@Resource的区别
*         1. @Autowired和@Resource都是用于自动注入的，都可以标注在属性上
*         2. @Autowired是spring框架提供的，默认按类型，再按名称，不支持其他框架使用；
*         @Resource是java标准下的注解，默认按名称，再按类型，具有通用性
*         3.@Autowired有required属性，默认为true，表示必须注入成功，否则报错
*         可以通过required属性，设置false，表示不注入成功，也不报错。对象创建成功，会返回对象，否则返回null
*         而@Resource不支持required属性
*
*
*   5.依赖注入方式
*    1）.setter注入 ：需要标注@Autowired
*    2）.构造器注入：多个构造器需要标注@Autowired，单个构造器不需要标注@Autowired
*    3）.字段注入： 需要标注@Autowired
*
*
*   6.@Value
*     1).  @Value("字面值")  ：直接赋值
*     2).  @Value("${key}")  ：从配置文件中获取属性值
*           @Value("${key:default}")  : 获取属性值，如果获取不到，则使用默认值
*            @PorpertySource来指定配置文件的路径，以便获取配置文件的内容
*     3).  @Value("#{SpEL}")  : SpEL表达式，执行代码语句
*
* 
*   7.@Profile:指定组件在哪个环境生效
*   1). @Profile("环境名称") ：环境名称是我们自定义的，默认是default
*   2). 环境激活：在配置文件中添加  spring.profiles.active=环境名称
```





## AOP

### 一、静态代理和动态代理

```
* 静态代理:   编码期间就决定好了代理的关系
*    定义：代理对象是目标对象的接口的子类型，代理对象本身并不是目标对象，而是将目标对象作为自己的属性
*    优点：同一种类型的所有对象都能代理
*    缺点：如果要代理的对象很多，则需要编写大量的静态代理类 范围太小了，只能负责部分接口
* 动态代理 ：编码期间不决定代理关系，而是在运行时动态创建代理对象。
*    定义：目标对象在执行期间会被动态拦截，插入指定逻辑
*    优点：可以代理任何接口的实现对象
*    缺点：不好写

* 动态代理：Jdk动态代理，强制要求目标对象必须要有接口。代理的也只是接口中的方法
*  ClassLoader loader, 类加载器
*  Class<?>[] interfaces, 目标对象实现的接口
*  InvocationHandler h        调用处理器
```

```
public class DynamicProxy {
    public static Object getProxyInstance(Object target) {
       return   Proxy.newProxyInstance(target.getClass().getClassLoader(),
                target.getClass().getInterfaces(),
                (proxy,method,args)->{
                    //目标方法执行前
                    System.out.println("执行了"+method.getName()+"方法   "+"参数："+ Arrays.toString(args));

                    //执行目标对象的方法  真正执行前可以拦截   invoke:调用
                    Object result =method.invoke(target, args);

                    //目标方法执行后
                    System.out.println("执行了"+method.getName()+"方法   "+"结果："+result);

                    return result;
                }
                );
    }
}
```

### 二、切面类定义与使用

```
* 告诉spring 以下四个通知方法何时何地执行
* 何时？
*      @Before ：方法执行前执行
*      @After ：方法执行后执行
*      @AfterReturning ：方法正常执行返回结果后运行
*      @AfterThrowing ：方法执行异常后运行
*
* 何地
*      切入点表达式：execution(修饰符 返回值类型 方法名(参数列表))
*      全写法：execution(【public】 int 【spring02aop.calculator.CalculatorImpl.】 add(int,int) 【throws Exception】)
*      省略写法：execution(返回值类型 方法名(参数列表))
*      execution(int add(int,int))
*
*      通配符：
*      * 表示任意字符
*      .. 表示任意参数
*          1 参数位置：表示多个参数，任意类型
*          2 类型位置（包路径）: 表示任意包路径 ，代表多个层级
*
*
* 通知方法的执行顺序：
*     1 正常链路： 前置通知->目标方法->返回通知->后置通知
*     2 异常链路： 前置通知->目标方法->异常通知->后置通知
*
* JoinPoint ：切入点对象，封装了当前正在执行的方法的信息
```

**普通通知**

```
@Order(100)  // 定义切面优先级，数字越小越先执行,套到最外层
@Component
@Aspect  // 定义一个切面类
public class LogAspect {

    @Pointcut("execution(int *(int, int))")
    public void pointCut() {}
    
    @Before("pointCut()")
    public void logStart(JoinPoint joinPoint) {
        // 获取当前正在执行的方法的签名信息
        MethodSignature signature =(MethodSignature) joinPoint.getSignature();

        // 获取当前正在执行的方法的名称
        String name = signature.getName();
        // 获取当前正在执行的方法的参数
        Object[] args = joinPoint.getArgs();

        System.out.println("日志切面---方法："+name+"开始, 参数："+ Arrays.toString(args));
    }

    @After("pointCut()")
    public void logEnd(JoinPoint joinPoint) {
        // 获取当前正在执行的方法的签名信息
        MethodSignature signature =(MethodSignature) joinPoint.getSignature();
        // 获取当前正在执行的方法的名称
        String name = signature.getName();
        System.out.println("日志切面---方法："+name+"结束");
    }

    @AfterThrowing(value = "pointCut()",
            throwing = "ex")// throwing ：指定异常对象的名称，用于接收目标方法抛出的异常
    public void logError(JoinPoint joinPoint, Throwable ex) {
        // 获取当前正在执行的方法的签名信息
        MethodSignature signature =(MethodSignature) joinPoint.getSignature();
        // 获取当前正在执行的方法的名称
        String name = signature.getName();
        System.out.println("日志切面---方法："+name+"异常"+"  错误信息: "+ex.getMessage());
    }

    @AfterReturning(value = "pointCut()",
            returning = "result")// returning ：指定目标方法返回值的名称，用于接收返回值
    public void logReturn(JoinPoint joinPoint, Object result) {
        // 获取当前正在执行的方法的签名信息
        MethodSignature signature =(MethodSignature) joinPoint.getSignature();

        // 获取当前正在执行的方法的名称
        String name = signature.getName();
        System.out.println("日志切面---方法："+name+"返回, 结果："+result);

    }
}
```

**环绕通知**

```
* 【声明式】  vs  【编程式】
* 声明式：通过注解等方式，告诉框架，我要做什么，框架帮我做
*      优点：代码简洁，清晰
*      缺点：封装太多，排错困难
* 编程式：通过编码的方式，自己写代码实现功能
*      优点：灵活，排错简单
*      缺点：代码繁琐，冗余
```

```
* @Around: 环绕通知：可以控制目标方法是否执行，修改目标方法参数，执行结果等。
* 环绕通知固定写法如下：
*  Object :返回值
*  ProceedingJoinPoint :  可以继续推进的切点 （目标方法执行需要的参数）
*
* 自己处理异常后还要将异常抛出给别人识别
```

```
@Component
@Aspect
public class AroundAspect {

    @Pointcut("execution(int *(int, int))")
    public void pointCut() {}

    //环绕通知
    @Around("pointCut()")
    public Object around(ProceedingJoinPoint pjb) throws Throwable {
        Object[] args = pjb.getArgs();//获取目标方法的参数

        //前置通知
        System.out.println("环绕通知-前置通知 , 参数： "+ Arrays.toString(args));
        Object proceed = null; //目标方法执行结果
        try{
            //接收传入参数的proceed ,实现修改目标方法执行用的参数
            proceed = pjb.proceed(args);//继续执行目标方法；类似于反射的method.invoke()
            //正常返回通知
            System.out.println("环绕通知-正常返回通知 ，返回值："+proceed);
        }catch(Exception e){
            //异常通知
            System.out.println("环绕通知-异常通知 ,异常信息："+e.getMessage());
            throw e; //抛出异常，让别人知道这里有错误
            //throw new RuntimeException("环绕通知-异常通知 , 异常信息："+e.getMessage());
        }finally {
            //后置通知
            System.out.println("环绕通知-后置通知");
        }
        //返回目标方法的执行结果(可以修改返回值)
        return proceed;
    }
}
```

### 三、AOP的主要应用场景

```
* 1. 日志记录
* 在项目开发中，日志记录是一个常见的横切关注点。通过AOP，可以在方法执行前后自动记录日志，而无需在每个方法中手动编写日志代码。
*
* 2. 事务管理
* 事务管理是数据库操作中的一个重要方面。AOP可以简化事务的开启、提交和回滚操作
*
* 3. 安全检查
*安全检查是确保系统安全性的重要环节。AOP可以在方法执行前进行权限验证、身份认证等安全检查
*
* 4. 缓存管理
*缓存是提高系统性能的有效手段。AOP可以用于管理方法的缓存逻辑，如缓存数据的读取和更新。
*
*5. 异常处理
* 异常处理是确保系统稳定性的重要环节。AOP可以用于集中处理方法执行过程中抛出的异常。
```

### 四、增强器链与AOP底层原理

```
* 增强器链：切面中的所有通知方法其实就是增强器。 他们被组织成一个链路放到集合中。目标方法真正执行前后
* 会去增强器链中执行这些那些需要提前执行的方法。
*
* AOP的底层原理
*      1 Spring会为每个被切面切入的组件创建代理对象（Spring CGLIB 创建的代理对象，无视接口）。
*      2 代理对象中保存了切面类里面所有通知方法构成的增强器链。
*      3 在目标方法执行时，会去增强器链中拿到需要提前执行的通知方法去执行。
```

## 底层原理（面试题）

### 一、SpringBean的生命周期



![image-20260412193523274](C:\Users\缪雨\AppData\Roaming\Typora\typora-user-images\image-20260412193523274.png)

### 二、Spring循环依赖

<img src="C:\Users\缪雨\AppData\Roaming\Typora\typora-user-images\image-20260412194008262.png" alt="image-20260412194008262" style="zoom: 50%;" />

### 三、容器底层的三级缓存（Map）机制

![屏幕截图 2026-04-12 194103](C:\Users\缪雨\Pictures\Screenshots\屏幕截图 2026-04-12 194103.png)

```
protected @Nullable Object getSingleton(String beanName, boolean allowEarlyReference) {
        Object singletonObject = this.singletonObjects.get(beanName);
        if (singletonObject == null && this.isSingletonCurrentlyInCreation(beanName)) {
            singletonObject = this.earlySingletonObjects.get(beanName);
            if (singletonObject == null && allowEarlyReference) {
                if (!this.singletonLock.tryLock()) {
                    return null;
                }

                try {
                    singletonObject = this.singletonObjects.get(beanName);
                    if (singletonObject == null) {
                        singletonObject = this.earlySingletonObjects.get(beanName);
                        if (singletonObject == null) {
                            ObjectFactory<?> singletonFactory = (ObjectFactory)this.singletonFactories.get(beanName);
                            if (singletonFactory != null) {
                                singletonObject = singletonFactory.getObject();
                                if (this.singletonFactories.remove(beanName) != null) {
                                    this.earlySingletonObjects.put(beanName, singletonObject);
                                } else {
                                    singletonObject = this.singletonObjects.get(beanName);
                                }
                            }
                        }
                    }
                } finally {
                    this.singletonLock.unlock();
                }
            }
        }

        return singletonObject;
    }
```

### 四、双检查锁机制

双检查锁是一种用于延迟初始化单例对象的优化技术,它在保证线程安全的同时减少了同步开销。

核心思想
		1、第一次检查:不加锁的情况下检查实例是否已创建
		2、加锁:如果未创建,才进行同步
		3、第二次检查:在同步块内再次检查实例是否已创建

    public class Singleton {
        // 必须使用 volatile 关键字
        private static volatile Singleton instance;
    
        private Singleton() {
            // 私有构造函数
        }
    
        public static Singleton getInstance() {
            // 第一次检查(不加锁)
            if (instance == null) {
                // 同步块
                synchronized (Singleton.class) {
                    // 第二次检查(加锁后)
                    if (instance == null) {
                        instance = new Singleton();
                    }
                }
            }
            return instance;
        }
     }
1、关键点说明
		volatile 关键字必不可少
		防止指令重排序
		保证多线程间的可见性
		new Singleton() 分为三步:分配内存、初始化对象、引用指向内存,volatile 防止步骤2和3重排序
2、为什么需要两次检查?
		第一次检查:避免不必要的同步,提高性能
		第二次检查:确保只有一个线程创建实例
3、优点
		线程安全
		延迟加载
		性能好(只在首次创建时同步)

4、应用场景
		* 单例模式实现 *
		资源密集型对象的延迟初始化
		需要线程安全且高性能的场景

## 声明式事务

### 一、原理

```
*  1 transactionManager:事务管理器；控制事务的获取、提交、回滚。
*     底层默认使用JdbcTransactionManager（事务管理器）;
*      原理:
*      1、事务管理器(接口)：TransactionManager:控制事务获取、提交、回滚
*      2 事务拦截器（是一个切面）：TransactionInterceptor：控制何时提交和回滚
*                 completeTransactionAfterThrowing(txInfo ex); 在这时进行回滚
*                 completeTransactionAfterReturning(txInfo); 在这时进行提交
```

### 二、事务细节

```
* 2、propagation:事务的传播行为 【一定要注意异常传播链，某处报错下面代码不执行】
*        Propagation.REQUIRED:支持当前事务，如果当前没有事务，就新建一个事务。这是最常见的选择
*        Propagation.SUPPORTS:支持当前事务，如果当前没有事务，就以非事务方式执行。
*        Propagation.MANDATORY:使用当前事务，如果当前没有事务，就抛出异常。
*        Propagation.REQUIRES_NEW:新建事务，如果当前存在事务，把当前事务挂起。
*        Propagation.NOT_SUPPORTED:以非事务方式执行操作，如果当前存在事务，就把当前事务挂起。
*        Propagation.NEVER:以非事务方式执行，如果当前存在事务，则抛出异常。
*        Propagation.NESTED:如果当前存在事务，则在嵌套事务内执行。如果当前没有事务，则执行与REQUIRED类似的操作。
*        嵌套事务：内部事务依赖于外部事务，外部事务失败时，内部也会回滚。
*     REQUIRED  ：你炸我也炸
*     REQUIRES_NEW ：你炸我可以跑
*
*  传播行为：参数设置项也会传播；如果小事务和大事务公用一个事务，小事务要按照大事务的设置，小事务自己的设置失效。
*
* 3、isolation:事务的隔离级别
*        读未提交
*        读已提交（oracle默认）
*        可重复读（mysql默认）
*        串行化
*
*
* 4、timeout(同timeoutString):超时时间  ,单位秒
*         一旦超过约定时间，事务自动回滚
*         超时时间是指从方法开始，到最后一次数据库操作结束的时间。
*
* 5、readOnly:是否只读（只读优化）
*
* 6、rollbackFor:指定哪些异常发生时，事务回滚. 不是所有的异常都一定引起事务回滚
*      异常：
*          运行时异常：【非受检异常】
*          编译时异常：【受检异常】
*     【回滚机制】：
*          运行时异常回滚
*          编译时异常不回滚
*
*     【可以指定那些异常需要回滚】：【可回滚：运行时异常+指定回滚异常】
*
* 7、noRollbackFor:指定哪些异常发生时，事务不回滚
*          【不回滚：编译时异常+指定不回滚异常】
```





# SpringMvc



## 一、请求处理

### 1、@RequestMapping路径映射与请求限定

（了解）请求限定（@RequestMapping的参数）
		请求方式：method
		请求参数：params
		请求头：headers
		请求内容类型：consumes
		响应内容类型：produces

```
@RestController     【 @ResponseBody(响应体)，@Controller】

*精确路径必须全局唯一
     *路径位置通配符，多个都能匹配上，那就精确优先
     *      *：匹配任意多个字符（0~N） 不能匹配多个路径
     *      **：匹配任意多层路径 （放在最后）
     *      ?：匹配任意一个字符（1个）
     *     精确程度：完全匹配 > ? > *  > **
```





### 2、常用的三种请求处理方式

1、接收普通参数

```
@RequestParam： 取出某个参数的值，默认一定要携带。
*          required=false：参数可以不携带
*          defaultValue="11"：默认值,参数可以不带


如果目标方法参数是一个pojo；springmvc会自动封装参数，将请求参数和pojo属性进行匹配
     * 效果：
     *      1、pojo属性名要和请求参数的名称一致
     *      2、如果请求参数没带，封装为null;
```

![屏幕截图 2026-04-14 194852](C:\Users\缪雨\Pictures\Screenshots\屏幕截图 2026-04-14 194852.png)

2、接收json数据

@RequestBody(请求体)

```
@RequestBody Person person
*      1、 拿到请求体中的json字符串
*      2、把json字符串转为person对象
```

![屏幕截图 2026-04-14 194953](C:\Users\缪雨\Pictures\Screenshots\屏幕截图 2026-04-14 194953.png)

3、接收文件上传

```
文件上传： @RequestParam取出文件项，封装为MultipartFile ,就可以拿到文件内容
可以在配置文件中设置文件上传大小限制
spring.servlet.multipart.max-file-size=1GB
spring.servlet.multipart.max-request-size=10GB
```

![屏幕截图 2026-04-14 195043](C:\Users\缪雨\Pictures\Screenshots\屏幕截图 2026-04-14 195043.png)









## 二、响应处理

### 1、响应JSON格式

@ResponseBody自动将返回数据转为Json

SpringMVC 底层使用 HttpMessageConverter 处理json数据的序列化与反序列化

### 2、文件下载

```
/** 只要响应文件都这样写，更改文件路径即可
 * 文件下载：
 *      HttpEntity:拿到整个请求数据
 *      ResponseEntity:拿到响应数据（响应头、响应体、状态码）
 *
 * @return
 * @throws IOException
 */
@RequestMapping("/download")
public ResponseEntity<InputStreamResource> download() throws IOException {

    FileInputStream fileInputStream = new FileInputStream("C:\\Users\\缪雨\\Pictures\\Camera Roll\\1.jpeg");

    //一口气读取文件内容，会溢出
    //byte[] bytes = fileInputStream.readAllBytes();

    //1、文件名为中文名乱码问题，解决
    String encode = URLEncoder.encode("测试.jpg", "UTF-8");

    //2、文件太大，一次性读取会溢出内存
    InputStreamResource inputStreamResource = new InputStreamResource(fileInputStream);

    return ResponseEntity.ok()
            //内容类型为：二进制流
            .contentType(MediaType.APPLICATION_OCTET_STREAM)
            //文件大小
            .contentLength(fileInputStream.available())
            //Content-Disposition:内容处理方式，这里是附件下载
            .header("Content-Disposition","attachment;filename=" + encode)
            //响应体 响应回去的内容
            .body(inputStreamResource);
}
```





## 三、RESTful-CRUD(retrieve)

| **URI**         | **请求方式** | **请求体**    | **作用**     | **返回数据**        |
| --------------- | ------------ | ------------- | ------------ | ------------------- |
| /employee/{id}  | GET          | 无            | 查询某个员工 | Employee JSON       |
| /employee       | POST         | employee json | 新增某个员工 | 成功或失败状态      |
| /employee       | PUT          | employee json | 修改某个员工 | 成功或失败状态      |
| /employee/{id}  | DELETE       | 无            | 删除某个员工 | 成功或失败状态      |
| /employees      | GET          | 无/查询条件   | 查询所有员工 | List<Employee> JSON |
| /employees/page | GET          | 无/分页条件   | 查询所有员工 | 分页数据 JSON       |

统一返回结果类型（R）和浏览器跨域（@CrossOrigin）

```
/**         写一个统一的common类 返回格式
 *
 * code:  业务的状态码，200是成功，剩下都是失败；前后端将来会一起商定不同的业务状态码，前端显示不同的效果
 * msg: 服务端发送给前端的提示信息，比如：操作失败，数据不存在等
 * data:  服务端发送给前端的业务数据，比如：查询到的员工信息等
 *
 * 统一格式
 * {
 *      "code": 200,
 *      "msg":"操作成功",
 *      "data":{
 *          id:1,
 *          name:'张三',
 *          age:30,
 *          gender:'男'
 *      }
 * }
 *
 *
 *  前端统一处理：
 *          1、前端发送请求，接收服务端数据
 *          2、判断状态码，成功显示数据，失败提示其他消息或执行其他的操作
```

```
* CORS policy : 同源策略（限制ajax请求、图片、css、js）   跨域问题
*  跨源资源共享（CORS）
*      浏览器为了安全，默认会遵循同源策略（请求要去的服务器和当前服务器必须一致），否则，请求发不出去
*      复杂的跨越请求会发送2次：
*      1、预检请求（浏览器自动发送）：OPTIONS。询问服务器是否允许跨域访问
*      2、真正的请求
*
* 浏览器页面所在： http://localhost   /employee/base
* 服务端： http://localhost:8080   /api/v1/employees
*  /前面的必须一致，请求才能发出
*
*
* 跨域问题的解决：
*      1、前端自己解决
*      2、服务端解决：
*            原理：在服务端设置允许跨域的请求 Access-Control-Allow-Origin = *
*            （加@CrossOrigin注解）
```







## 四、最佳实践

### 1、拦截器

SpringMVC 内置拦截器机制 ，允许在请求被目标方法处理的前后进行拦截，执行一些额外操作；比如：权限验证、日志记录、数据共享等...
使用步骤
		实现 HandlerInterceptor 接口的组件即可成为拦截器
		创建 WebMvcConfigurer 组件，并配置拦截器的拦截路径
		查看执行顺序效果：preHandle => 目标方法 => postHandle => afterCompletion

```
@Component //拦截器还需要配置（告诉springmvc，这个拦截器主要拦截什么请求）  config包中进行配置
public class MyHandlerInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        System.out.println("preHandle执行了");
        //放行
        return true;
    }


    /**
     * 目标方法（controller方法）执行完毕，才执行
     */
    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, org.springframework.web.servlet.ModelAndView modelAndView) throws Exception {
        System.out.println("postHandle执行了");
    }


    /**
     *preHandler执行成功返回true,对应的afterCompletion才执行
     */
    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
        System.out.println("afterCompletion执行了");
    }
}
```

```
/**
 * 1、容器中需要有这样一个组件：【WebMvcConfigurer】
 *      获得该组件：
 *          1、@Bean 放一个 WebMvcConfigurer
 *          2、配置类实现WebMvcConfigurer
 *
 */
@Configuration  //专门对springmvc底层进行配置
public class MySpringMvcConfig implements WebMvcConfigurer {

    @Autowired
    MyHandlerInterceptor myHandlerInterceptor;

    //添加拦截器
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
       registry.addInterceptor(myHandlerInterceptor)
               .addPathPatterns("/**"); //拦截所有请求
    }

    /*@Bean
        WebMvcConfigurer webMvcConfigurer(){
            return new WebMvcConfigurer() {
                @Override
                public void addInterceptors(InterceptorRegistry registry) {

                }
            };
        }*/
}
```





### 2、异常处理

编程式异常处理：
	try - catch、throw、exception
声明式异常处理：
	SpringMVC 提供了 @ExceptionHandler、@ControllerAdvice 等便捷的声明式注解来进行快速的异常处理
	@ExceptionHandler：可以处理指定类型异常
	@ControllerAdvice：可以集中处理所有Controller的异常
	@ExceptionHandler + @ControllerAdvice： 可以完成全局统一异常处理

```
异常处理的最终方式：
*  1、必须要有业务异常类：BizException
*  2、必须要有业务异常枚举类：BizExceptionEnum 枚举类中定义了业务异常码和业务异常信息
*  3、编写业务代码的时候，只需要编写正确逻辑，如果出现预期的问题，需要以抛异常的方式中断逻辑并通知上层
*  4、全局异常处理器：GlobalExceptionHandler：处理所有的异常，返回给前端约定的json数据和错误码

打印异常错误堆栈方便排错
```

```
super(...)：调用父类的构造方法，必须在子类构造方法的第一行
目的：让父类完成它自己的初始化工作（比如保存异常消息）
必要性：如果父类没有无参构造方法，子类必须显式调用 super(...)


@Data
public class BizException extends RuntimeException{

    private Integer code;//业务异常码

    private String msg;//业务异常信息

    public BizException(Integer code, String message) {
        super(message);    // 告诉父类 RuntimeException：异常信息是 message
        this.code = code;
        this.msg = message;
    }

    public BizException(BizExceptionEnum bizExceptionEnum) {
        super(bizExceptionEnum.getMsg());
        this.code = bizExceptionEnum.getCode();
        this.msg = bizExceptionEnum.getMsg();
    }
 }
```

```

public enum BizExceptionEnum {

    ORDER_CLOSED(40001,"订单已关闭，无法更新"),
    ORDER_NOT_EXIST(40002,"订单不存在"),
    ORDER_LIMIT_EXCEEDED(40003,"订单超出限额"),
    PRODUCT_NOT_EXIST(40004,"商品不存在"),
    PRODUCT_LIMIT_EXCEEDED(40005,"商品超出限额");
	
	@Getter
    private final Integer code;
    @Getter
    private final String msg;

   private BizExceptionEnum(Integer code, String message) {
        this.code = code;
        this.msg = message;
    }
}
```

```
//ControllerAdvice
//ResponseBody（将异常信息以JSON格式返回）

@RestControllerAdvice // 告诉springmvc，这个组件是专门负责进行全局处理异常的
public class GlobalExceptionHandler {

    /**
     * 如果出现异常，本类和全局都不能处理。
     *      springboot底层对springmvc有兜底处理机制，自适应处理(浏览器响应页面，移动端响应json)
     *
     * 最佳处理不使用兜底机制，而是自定义兜底处理机制。
     *
     */

    @ExceptionHandler(ArithmeticException.class)
    public R error(ArithmeticException e){
        System.out.println("【全局】-- ArithmeticException异常
        e.printStackTrace();  // 打印异常信息
        return R.error(500,e.getMessage());
    }

    @ExceptionHandler(Throwable.class)
    public R error(Throwable e){
        System.out.println("【全局】-- Throwable异常");
        e.printStackTrace();  // 打印异常信息
        return R.error(500,e.getMessage());
    }


    @ExceptionHandler(BizException.class)
    public R BizException(BizException e){
        System.out.println("【全局】-- BizException异常");
        e.printStackTrace();  // 打印异常信息
        return R.error(e.getCode(),e.getMsg());
    }
}
```





### 3、数据校验

数据校验使用流程
1、引入校验依赖：spring-boot-starter-validation
2、定义封装数据的Bean
3、给Bean的字段标注校验注解，并指定校验错误消息提示
4、使用@Valid（java标准）、@Validated（spring框架，被vo封层代替）开启校验
5、使用 BindingResult 封装校验结果
6、使用自定义校验注解 + 校验器(implements ConstraintValidator)  完成gender字段自定义校验规则
7、结合校验注解 message属性 与 i18n 文件，实现错误消息国际化
8、结合全局异常处理，统一处理数据校验错误

```
数据校验：
*  1、导入检验包  【spring-boot-starter-validation】
*  2、编写校验注解  【实体类属性上标注】  
	@Email(message = "邮箱格式不正确")  
	也可以通过占位符动态获取msg   @Email(message = "{email.message}")
	通过正则表达式校验
	@Pattern(regexp = "^男|女$", message = "性别只能是'男'或'女'")
	
*  3、使用@Valid（标在Controller接口方法参数前） 告诉springmvc进行校验
*      效果：
*      1、校验失败，则该方法不执行
*      2、校验失败，以示例格式返回信息
*      {
*          "code": 500,
*          "msg": "校验失败",
*          "data":
*          {
*              "name":"姓名不能为空",
*              "email":"邮箱格式不正确",
*              "age":"年龄不能为空",
*          }
*      }
*  4、【不推荐用】在@Valid 注解后面，通过BindingResult result拿到校验结果的信息，然后手动封装成统一的格式返回
*    效果2：全局异常处理机制
*  5、【推荐】：编写一个全局异常处理器，处理（检验出错时的异常），然后统一封装成统一的格式返回
*  6、自定义校验器= 自定义校验注解 + 自定义校验器
```

编写一个全局异常处理器，处理（检验出错时的异常），然后统一封装成统一的格式返回

```
@RestControllerAdvice // 告诉springmvc，这个组件是专门负责进行全局处理异常的
public class GlobalExceptionHandler {

	@ExceptionHandler(MethodArgumentNotValidException.class)
	public R MethodArgumentNotValidException(MethodArgumentNotValidException e){
   	 	//result中包含了所有校验的错误信息
    	BindingResult result = e.getBindingResult();
    	Map<String,String> map= new HashMap<>();
    	//校验失败
    	for (FieldError fieldError : result.getFieldErrors()) {
        	//获取校验失败的字段名
        	String field = fieldError.getField();
        	//获取校验失败的信息
       	 	String defaultMessage = fieldError.getDefaultMessage();
        	map.put(field,defaultMessage);
    	}
    	//手动封装返回
    	return R.error(500,"校验失败",map);
	}
}
```

自定义校验器= 自定义校验注解 + 自定义校验器

```
@Documented
@Constraint(
        validatedBy = {GenderValidator.class}  //校验器去真正完成校验功能
)
@Target({ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
public @interface Gender {

    String message() default "{jakarta.validation.constraints.Email.message}";

    Class<?>[] groups() default {};

    Class<? extends Payload>[] payload() default {};

}
```

```
public class GenderValidator implements ConstraintValidator<Gender, String> {

    /**
     *
     * @param value 前端提交来的准备让我们校验的属性值
     * @param context 校验上下文
     * @return
     */
    @Override
    public boolean isValid(String value, ConstraintValidatorContext context) {
        return "男".equals(value) || "女".equals(value);
    }
}
```







### 4、VO分层

```
设计模式：单一职责
*  JavaBean也要分层
*  各种xxo:
*      VO: view object 视图对象 封装前端需要的数据
*      DTO: data transfer object 数据传输对象
*      TO: transfer object 传输对象
*      pojo: plain old java object 普通的java对象
*      BO: business object 业务对象
*      DAO: data access object 数据访问对象
```







### 5、接口文档

使用步骤

1、引入依赖

2、配置文件

3、可能因版本问题或全局异常处理器而不能正常使用

Swagger 可以快速生成实时接口文档，方便前后开发人员进行协调沟通。遵循 OpenAPI 规范。
Knife4j 是基于 Swagger之上的增强套件

访问 http://ip:port/doc.html 即可查看接口文档

| **注解**       | **标注位置**        | **作用**               |
| -------------- | ------------------- | ---------------------- |
| **@Tag**       | controller 类       | 描述 controller 作用   |
| **@Parameter** | 参数                | 标识参数作用           |
| @Parameters    | 参数                | 参数多重说明           |
| **@Schema**    | model 层的 JavaBean | 描述模型作用及每个属性 |
| **@Operation** | 方法                | 描述方法作用           |
| @ApiResponse   | 方法                | 描述响应状态码等       |

### 6、数据转换

@JsonFormat：日期处理

```
//只要是日期，标注统一注解：
@JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss",timezone = "GMT+8")
private Date birth;

```













# MyBatis



## 一、MyBatis入门

- 1. 每个Dao 接口 对应一个 XML 实现文件
  2. Dao 实现类 是一个由 MyBatis 自动创建出来的代理对象
  3. XML 中 namespace 需要绑定 Dao 接口 的全类名
  4. XML 中使用 select、update、insert、delete 标签来代表增删改查
  5. 每个 CRUD 标签 的 id 必须为Dao接口的方法名
  6. 每个 CRUD标签的 resultType 是Dao接口的返回值类型全类名
  7. 未来遇到复杂的返回结果封装，需要指定 resultMap 规则
- 8. 以后 xxxDao 我们将按照习惯命名为 xxxMapper，这样更明显的表示出 持久层是用 MyBatis 实现的
  8. 批量xxxMapper接口扫描  @MapperScan,被扫描的包下的接口不标@Mapper也可识别



### 1、mybatis操做数据库步骤及properties文件配置

* 使用mybatis操做数据库步骤：
* 1、导入mybatis依赖
* 2、配置数据源
* 3、编写一个JavaBean 对应数据库的一个表模型
* 4、
*      以前：Dao接口-->Dao实现类-->标注@Repository注解
*      现在：Mapper接口-->Mapper.xml实现-->标注@Mapper注解
*   【安装mybatisx插件，自动为mapper类生成mapper文件】，在mapper.xml文件中配置接口方法的实现sql
*   5、（告诉mybatis去哪里找mapper文件）在application.properties中配置 			    	mybatis.mapper.locations=classpath:mapper/**.xml
*   6、#数据表中的下划线式转为驼峰式
	mybatis.configuration.map-underscore-to-camel-case=true
	#增加日志功能,查看sql语句
	logging.level.mybatis01helloworld.mapper=debug
*   7、编写单元测试

### 2、使用CRUD标签完成完整的功能

```

```

```
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd" >
<mapper namespace="mybatis01helloworld.mapper.EmpMapper">

    <!--namespace：编写mapper接口的全类名，代表，这个xml文件和这个mapper接口进行绑定-->

<!-- Emp getEmpById(Integer id);
   select 标签代表一次查询操作
        id ：绑定方法名
        resultType ：返回值类型
   -->

<!--
 #{}:参数位置动态取值，安全，无sql注入问题
 ${}：jdbc层面 ，表名等位置，不支持预编译，只能用${}   拼接sql片段，不安全，有sql注入问题
 -->
    <select id="getEmpById" resultType="mybatis01helloworld.bean.Emp">
            select id,emp_name empName,age,emp_salary empSalary from t_emp where id = #{id}
    </select>

    <select id="getAllEmps" resultType="mybatis01helloworld.bean.Emp">
        select * from t_emp
    </select>

    <!--
        useGeneratedKeys: 使用自动生成的id
        keyProperty=： 将自动生成的id赋值给哪个属性，这里指的是Emp对象的id属性
        自增id回填机制
    -->
    <insert id="addEmp" useGeneratedKeys="true"  keyProperty="id">
        insert into t_emp(id, emp_name, age, emp_salary) values (#{id}, #{empName}, #{age}, #{empSalary})
    </insert>

    <update id="updateEmp">
        update t_emp set emp_name = #{empName}, age = #{age}, emp_salary = #{empSalary} where id = #{id}
    </update>

    <delete id="deleteEmpById">
        delete from t_emp where id = #{id}
    </delete>

</mapper>
```

### 3、自增id回显

​		<!--
​        useGeneratedKeys: 使用自动生成的id
​        keyProperty=： 将自动生成的id赋值给哪个属性，这里指的是Emp对象的id属性
​        自增id回填机制
​    -->
​    <insert id="addEmp" useGeneratedKeys="true"  keyProperty="id">
​        insert into t_emp(id, emp_name, age, emp_salary) values (#{id}, #{empName}, #{age}, #{empSalary})
​    </insert>

<!--
 #{}:参数位置动态取值，安全，无sql注入问题
 ${}：jdbc层面 ，表名等位置，不支持预编译，只能用${}   拼接sql片段，不安全，有sql注入问题
 -->





## 二、参数传递



### 1、#{}与${}

#{}：底层使用 PreparedStatement 方式，SQL预编译后设置参数，无SQL注入攻击风险
${}：底层使用 Statement 方式，SQL无预编译，直接拼接参数，有SQL注入攻击风险
			所有参数位置，都应该用 #{}
			需要动态表名等，才用 ${}
最佳实践：
			凡是使用了 ${}  的业务，一定要自己编写防SQL注入攻击代码



### 2、参数取值

**最佳实践：即使只有一个参数，也用 @Param 指定参数名**

| **传参形式**        | **示例**                                                     | **取值方式**                                                 |
| ------------------- | :----------------------------------------------------------- | ------------------------------------------------------------ |
| 单个参数 - 普通类型 | getEmploy(Long id)                                           | #{变量名}                                                    |
| 单个参数 - List类型 | getEmploy(List<Long> id)                                     | #{变量名[0]}                                                 |
| 单个参数 - 对象类型 | addEmploy(Employ e)                                          | #{对象中属性名}                                              |
| 单个参数 - Map类型  | addEmploy(Map<String,Object> m)                              | #{map中属性名}                                               |
| 多个参数 - 无@Param | getEmploy(Long id,String name)                               | #{变量名} //新版兼容                                         |
| 多个参数 - 有@Param | getEmploy(@Param(“id”)Long id,@Param(“name”)String name)     | #{param指定的名}                                             |
| 扩展：              | getEmploy(@Param(“id”)Long id,@Param(“ext”)Map<String,Object> m,@Param(“ids”)List<Long> ids,@Param(“emp”)Employ e) | #{id}、#{ext.name}、#{ext.age}，#{ids[0]}、#{ids[1]}，#{e.email}、#{e.age} |





## 三、结果封装

### 1、普通类型、普通对象

​			返回基本类型、普通对象 都只需要在 resultType 中声明返回值类型全类名即可



### 2、集合（List、Map、Set）

```
返回集合【list,map】，resultType="集合中元素的全类名"
```

​			List：resultType 为集合中的 元素类型
​			Map：resultType 为 map，配合 @MapKey 指定哪一列的值作为Map 的 key，Map 的 Value 为这一行数据的完整信息
​				Map<Key,Map>

```
@MapKey("id")
Map<Integer,Emp> getEmp();
```

```
<select id="getEmp" resultType="mybatis01helloworld.bean.Emp">
    select * from t_emp;
</select>
```



### 3、自定义结果集ResultMap

id 标签：必须指定主键列映射规则
result 标签：指定普通列映射规则
collection 标签：指定自定义对象封装规则，一般用户联合查询一对一关系的封装。比如一个用户对应一个订单
		ofType：指定集合中每个元素的类型
		select：指定分步查询调用的方法
		column：指定分步查询传递的参数列
association 标签：指定自定义对象封装规则，一般用户联合查询一对一关系的封装。比如一个用户对应一个订单
		javaType：指定关联的Bean的类型
		select：指定分步查询调用的方法
		column：指定分步查询传递的参数列

```
最佳实践：
*      1、开启驼峰命名
*      2、1搞不定的，用自定义映射（ResultMap）
*     默认封装规则（resultType），JavaBean中的属性名去数据库中找对应列明的值。————映射封装
*     自定义规则（resultMap）：我们来告诉mybatis 如何把结果封装到bean中。
*         明确指定每一列如何封装到指定的bean属性中。
```

```
<resultMap id="EmpResultMap" type="mybatis01helloworld.bean.Emp">
    <!-- id:声明主键的映射规则  -->
    <id column="id" property="id"></id>
    <!-- result:声明非主键的映射规则 -->
    <result column="emp_name" property="empName"></result>
    <result column="age" property="age"></result>
    <result column="emp_salary" property="empSalary"></result>
</resultMap>
```



**一对一关联查询**（association）

```
<resultMap id="orderRM" type="mybatis01helloworld.bean.Order">
    <id column="id" property="id"/>
    <result column="address" property="address"/>
    <result column="amount" property="amount"/>
    <result column="customer_id" property="customerId"/>
    <!--    一对一关联封装    -->
    <association property="customer" javaType="mybatis01helloworld.bean.Customer">
            <id column="c_id" property="id"/>
            <result column="customer_name" property="customerName"/>
            <result column="phone" property="phone"/>
    </association>
</resultMap>
<select id="getOrderByIdWithCunstomer" resultMap="orderRM">
    select o.*, c.id c_id,c.customer_name,c.phone
    from t_order o
            left join t_customer c on o.customer_id = c.id
            where o.id = #{id}
</select>
```

**一对多关联查询	**(collection)

```
<resultMap id="CustomerRM" type="mybatis01helloworld.bean.Customer">
    <id column="c_id" property="id"/>
    <result column="customer_name" property="customerName"/>
    <result column="phone" property="phone"/>
    <collection property="order" ofType="mybatis01helloworld.bean.Order">
        <id column="id" property="id"/>
        <result column="address" property="address"/>
        <result column="amount" property="amount"/>
        <result column="customer_id" property="customerId"/>
    </collection>
</resultMap>

<select id="getCustomerByIdWithOrders" resultMap="CustomerRM">
    select c.id c_id,c.customer_name,c.phone, o.*
    from t_customer c
             left join t_order o on o.customer_id= c.id
    where c.id=#{id}
</select>
```

**多对多关联查询**（多个一对多）

**分布查询（了解）**

select：指定分步查询调用的方法
column：指定分步查询传递的参数列

**延迟加载了解**











## 四、动态SQL



### 1、if、where标签

```
<!--
  if标签：判断；
  where标签：解决where后面语法错误问题（多and，or,无任何条件多where）
 -->
<select id="getEmpByNameAndSalary" resultType="mybatis01helloworld.bean.Emp">
    select * from t_emp
    <where>
      <if test="name != null ">
          emp_name = #{name}
      </if>
      <if test="salary != null ">
          and emp_salary = #{salary}
      </if>
    </where>
</select>
```



### 2、set 标签

```
<update id="updateEmp">
    update t_emp
    <set>
        <if test="emp.empName != null ">
            emp_name = #{emp.empName},
        </if>
        <if test="emp.age!= null ">
            age = #{emp.age},
        </if>
        <if test="emp.empSalary!= null ">
            emp_salary = #{emp.empSalary}
        </if>
    </set>
    where id = #{emp.id}
</update>
```



### 3、trim标签（了解）

trim 可以 实现 set 去掉多余逗号，where 去掉多余and/or 的功能， 不过写起来比较麻烦

```
<!--
        trim标签：
        prefix：前缀     如果标签体中有东西，就给他拼一个前缀
        prefixOverrides：前缀覆盖   标签体中最终生成的字符串，如果以指定前缀开始，就把这个前缀去掉
        suffix：后缀
        suffixOverrides：后缀覆盖
-->
    <select id="getEmpByNameAndSalary" resultType="mybatis01helloworld.bean.Emp">
        select * from t_emp
            <trim prefix="where" prefixOverrides="and || or">
                <if test="name != null ">
                    emp_name = #{name}
                </if>
                <if test="salary != null ">
                    and emp_salary = #{salary}
                </if>
            </trim>
    </select>
```



### 4、choose, when , otherwise 标签

在多个分支条件中，执行一个

```
<select id="selectEmployeeByConditionByChoose"
        resultType="com.atguigu.mybatis.entity.Employee">
    select emp_id,emp_name,emp_salary from t_emp where
    <choose>
        <when test="empName != null">emp_name=#{empName}</when>
        <when test="empSalary &lt; 3000">emp_salary &lt; 3000</when>
        <otherwise>1=1</otherwise>
    </choose>
</select>
```



### 5、foreach标签 *

用来遍历，循环；常用于批量插入场景；批量单个SQL

```
<!--
    foreach标签：:遍历List,Set,Map,Array
        collection：指定要遍历的集合
        item：指定遍历出来的每一个元素，给一个别名
        open：拼接在遍历内容的前面
        close：拼接在遍历内容的后面
        separator：遍历集合时，每个元素之间的分隔符
-->
    <select id="getEmpByIdIn" resultType="mybatis01helloworld.bean.Emp">
        select * from t_emp
            <if test="ids != null">
            <foreach collection="ids" item="id" open="where id in ("  close=")" separator=",">
                #{id}
            </foreach>
            </if>
    </select>
```



批量多个SQL
配置文件  jdbc:mysql:///mybatis-example?allowMultiQueries=true
allowMultiQueries：允许多个SQL用;隔开，批量发送给数据库执行

```
<update id="updateEmployeeBatch">
    <foreach collection="empList" item="emp" separator=";">
        update t_emp set emp_name=#{emp.empName} where emp_id=#{emp.empId}
    </foreach>
</update>
```



### 6、抽取可复用的Sql片段与xml中特殊字符表示

```
<sql id="empColumn">
         emp_id,emp_name,emp_age,emp_salary,emp_gender
</sql>

<select id="getEmp" resultType="com.atguigu.mybatis.entity.Employee">
        select
            <include refid="empColumn"/>
        from `t_emp` where id = #{id}
</select>
```

| **原始字符** | **转义字符** |
| ------------ | ------------ |
| &            | &amp;        |
| <            | &lt;         |
| >            | &gt;         |
| "            | &quot;       |
| '            | &apos;       |











## 五、Mybatis 扩展



### 1、缓存机制与插件机制（了解）



### 2、分页插件（PageHelper）

分页插件就是利用MyBatis 插件机制，在底层编写了 分页Interceptor，每次SQL查询之前会自动拼装分页数据



使用时参考官网

1、引入依赖

```
<dependency>
    <groupId>com.github.pagehelper</groupId>
    <artifactId>pagehelper</artifactId>
    <version>6.1.1</version>
</dependency>
```

2、创建配置类将组件注入容器

```
@Configuration
public class MybatisConfig {

    @Bean
    PageInterceptor pageInterceptor() {
        //1、创建 分页插件 对象
        PageInterceptor pageInterceptor = new PageInterceptor();
        //2、设置 分页插件 的属性
        //.......
        Properties properties = new Properties();
        properties.setProperty("reasonable","true");
        pageInterceptor.setProperties(properties);
        return pageInterceptor;
    }
}
```

3、使用

```
 /**
     * 原理：拦截器；
     * 原业务底层：select * from t_emp;
     * 拦截器做两件事：
     *      1、统计这个表总数量
     *      2、给原业务底层sql动态拼接limit （pageNum-1）*pageSize ,3 ;
     *
     *  ThreadLocal:同一个线程共享数据
     *      1、第一个查询从ThreadLocal 中获取到共享数据，执行分页
     *      2、第一个执行完把ThreadLocal中的数据清除
     *      3、以后的查询从ThreadLocal 中获取不到共享数据，不执行分页
     */
    @Test
    void testPage() {

        PageHelper.startPage(1, 3);
        //紧跟在startPage后面的第一个Mybatis查询，将进行分页
        List<Emp> allEmp = empDynamicSQLMapper.getAllEmp();
        allEmp.forEach(System.out::println);

    }
}
```















# SpringBoot





## 一、快速入门



### 1、springboot特性-快速部署打包

<!--    SpringBoot应用打包插件-->
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>



打包：maven   clean package

在jar包目录下输入cmd 运行：java -jar demo.jar





### 2、场景启动器

场景启动器：导入相关的场景，拥有相关的功能。

​		官方提供的场景：命名为：spring-boot-starter-*

​				spring-boot-starter-web    spring-boot-starter-test

​		第三方提供场景：命名为：*-spring-boot-starter

​				mybatis-spring-boot-starter

把当前场景需要的jar包都导入进来，每个场景启动器都有一个基础依赖spring-boot-starter



### 3、依赖管理

为什么 不用写版本号？ 

​			maven父子继承，父项目可以锁定版本

哪些需要写版本号？
			父不管的都需要写版本

如何修改默认版本号？
			version 精确声明版本
			子覆盖父的属性设置







## 二、面试题（SpringBoot自动配置）





### 1、初步理解

自动配置
		导入场景，容器中就会自动配置好这个场景的核心组件。
		如 Tomcat、SpringMVC、DataSource 等
		不喜欢的组件可以自行配置进行替换
默认的包扫描规则
		SpringBoot只会扫描主程序所在的包及其下面的子包
配置默认值
		配置文件的所有配置项 是和 某个类的对象值进行一一绑定的。
		很多配置即使不写都有默认值，如：端口号，字符编码等
		默认能写的所有配置项：https://docs.spring.io/spring-boot/appendix/application-properties/index.html
按需加载自动配置
		导入的场景会导入全量自动配置包，但并不是都生效



### 2、完整核心流程

（1）核心流程总结：
		1: 导入 starter，就会导入autoconfigure 包。
		2: autoconfigure 包里面 有一个文件 META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports,里面指定的所有启动要加载的自动配置类
		3: @EnableAutoConfiguration 会自动的把上面文件里面写的所有自动配置类都导入进来。xxxAutoConfiguration 是有条件注解进行按需加载
		4: xxxAutoConfiguration 给容器中导入一堆组件，组件都是从 xxxProperties 中提取属性值
		5: xxxProperties 又是和配置文件进行了绑定
（2）效果：导入starter、修改配置文件，就能修改底层行为



![image-20260427171204348](C:\Users\缪雨\AppData\Roaming\Typora\typora-user-images\image-20260427171204348.png)







## 三、基础功能



### 1、属性绑定

将容器中任意组件的属性值和配置文件的配置项的值进行绑定
1、给容器中注册组件（@Component、@Bean）
2、使用 @ConfigurationProperties 声明组件和配置文件的哪些配置项进行绑定



### 2、YAML文件

```
#如果yaml和properties混合使用，则properties优先级更高
```

**（1）基本语法**

大小写敏感
键值对写法 k: v，使用空格分割k,v
使用缩进表示层级关系
		缩进时不允许使用Tab键，只允许使用空格。换行
		缩进的空格数目不重要，只要相同层级的元素左侧对齐即可
“#”  表示注释，从这个字符一直到行尾，都会被解析器忽略。
Value支持的写法
		对象：键值对的集合，如：映射（map）/ 哈希（hash） / 字典（dictionary）
		数组：一组按次序排列的值，如：序列（sequence） / 列表（list）
		字面量：单个的、不可再分的值，如：字符串、数字、bool、日期



![屏幕截图 2026-04-27 190029](C:\Users\缪雨\Pictures\Screenshots\屏幕截图 2026-04-27 190029.png)



数组写法：

```
text: [aa, bb, cc]
或
dogs:
  - name: 狗1
    age: 1
  - name: 狗2
    age: 2
  - {name: 狗3, age: 3}
```

对象写法：

```
cats:
  cat1:
    name: 猫1
    age: 1
  cat2:
    name: 猫2
    age: 2
```

日期格式

```
birthDay: 2003/01/01 12:00:00
```



### 3、自定义SpringApplication

new SpringApplication
new SpringApplicationBuilder

可以在容器启动前做一些配置



## 四、日志系统  *

SpringBoot 默认使用 slf4j + logback

### 1、日志格式和日志级别

```
格式： 时间  级别   进程ID  项目名  线程名   类名    日志内容

注意： logback 没有FATAL级别，对应的是ERROR

级别：由低到高 ALL TRACE DEBUG INFO WARN ERROR FATAL OFF
只会打印指定级别及以上级别的日志
ALL：打印所有日志
TRACE：追踪框架详细流程日志，一般不使用
DEBUG：开发调试细节日志
INFO：关键、感兴趣信息日志
WARN：警告但不是错误的信息日志，比如：版本过时
ERROR：业务错误日志，比如出现各种异常
FATAL：致命错误日志，比如jvm系统崩溃
OFF：关闭所有日志记录

#如果那个包、哪个类不说日志级别，就默认root的级别
#logging.level.root=info  默认为info
#还可以单独设置包或类的日志级别
logging.level.包名=DEBUG
#logging.level.springboot01hello.properties=info


级别越高日志越少：日志默认级别是info，只打印info及以上级别

```



### 2、日志分组

```
logging.group.tomcat=org.apache.catalina,org.apache.coyote,org.apache.tomcat
logging.level.tomcat=trace
```



### 3、日志输出

```
#日志文件输出
#当前项目所在的根文件夹下生成一个 指定名字的 日志文件
logging.file.name=log.txt
或者指定输出文件路径
logging.file.path=
```



### 4、文件归档与滚动切割 

归档：每天的日志单独存到一个文档中。
切割：每个文件10MB，超过大小切割成另外一个文件。
默认滚动切割与归档规则如下：

| **配置项**                                               | **描述**                                                     |
| -------------------------------------------------------- | ------------------------------------------------------------ |
| logging.**logback**.rollingpolicy.file-name-pattern      | 日志存档的文件名格式默认值：${LOG_FILE}.%d{yyyy-MM-dd}.%i.gz |
| logging.**logback**.rollingpolicy.clean-history-on-start | 应用启动时是否清除以前存档；默认值：false                    |
| logging.**logback**.rollingpolicy.max-file-size          | 每个日志文件的最大大小；默认值：10MB                         |
| logging.**logback**.rollingpolicy.total-size-cap         | 日志文件被删除之前，可以容纳的最大大小（默认值：0B）。设置1GB则磁盘存储超过 1GB 日志后就会删除旧日志文件 |
| logging.**logback**.rollingpolicy.max-history            | 日志文件保存的最大天数；默认值：7                            |



### 5、切换日志组合

logback 改为 log4j2

![image-20260428132903953](C:\Users\缪雨\AppData\Roaming\Typora\typora-user-images\image-20260428132903953.png)



### 6、最佳实践

1、导入任何第三方框架，先排除它的日志包，因为Boot底层控制好了日志
2、修改 application.properties 配置文件，就可以调整日志的所有行为。如果不够，可以编写日志框架自己的配置文件放在类路径下就行，比如logback-spring.xml，log4j2-spring.xml
3、如需对接专业日志系统，也只需要把 logback 记录的日志灌倒 kafka之类的中间件，这和SpringBoot没关系，都是日志框架自己的配置，修改配置文件即可
4、业务中使用slf4j-api记录日志。不要再 sout 了

```
#我们用日志， 再也不用sout
#1、配置（日志输出到文件、打印日志级别）
#2、记录日志（合适的时候选择合适的级别进行日志记录
```





## 五、进阶使用



### 1、Profiles环境隔离

```
* 环境隔离：
*  1、定义环境： dev(开发)、test(测试)、prod(生产)
*  2、定义这个环境下生效那些组件或那些配置
*      1）、生效那些组件： 给组件添加环境标识 @Profile("dev")
*      2）、生效那些配置： 给配置文件添加环境标识 application-{dev|test|prod}.properties
*  3、激活环境：
*      1）、在application.properties中配置：spring.profiles.active=dev
*      2）、命令行：java -jar xxx.jar --spring.profiles.active=dev
*
*注意： 激活的配置优先级高于默认配置
*生效的配置：默认配置+激活的配置（profiles.active）+包含的配置（profiles.include）

```

**环境隔离分组**

创建 prod 组，指定包含 db 和 mq 配置
spring.profiles.group.prod[0]=db
spring.profiles.group.prod[1]=mq
使用 --spring.profiles.active=prod ，激活prod，db，mq配置文件

### 2、外部化配置

外部配置优先于内部配置

**”激活优先，外部优先（且激活优先大于外部优先）“**

<img src="C:\Users\缪雨\AppData\Roaming\Typora\typora-user-images\image-20260428192511740.png" alt="image-20260428192511740" style="zoom: 67%;" />





### 3、单元测试进阶-断言机制与测试注解

断言机制：通俗说就是提前设置预期结果单元测试结果不符合报错

```
@Test
void test(){
    String hello = helloService.hello();
    //断言
    Assertions.assertEquals("hello", hello,"测试失败");
}
```

| **方法**          | **说明**                             |
| ----------------- | ------------------------------------ |
| assertEquals      | 判断两个对象或两个原始类型是否相等   |
| assertNotEquals   | 判断两个对象或两个原始类型是否不相等 |
| assertSame        | 判断两个对象引用是否指向同一个对象   |
| assertNotSame     | 判断两个对象引用是否指向不同的对象   |
| assertTrue        | 判断给定的布尔值是否为 true          |
| assertFalse       | 判断给定的布尔值是否为 false         |
| assertNull        | 判断给定的对象引用是否为 null        |
| assertNotNull     | 判断给定的对象引用是否不为 null      |
| assertArrayEquals | 数组断言                             |
| assertAll         | 组合断言                             |
| assertThrows      | 异常断言                             |
| assertTimeout     | 超时断言                             |
| fail              | 快速失败                             |

**测试注解**

@Test :表示方法是测试方法。

@DisplayName :为测试类或者测试方法设置展示名称
@BeforeEach :表示在每个单元测试之前执行
@AfterEach :表示在每个单元测试之后执行
@BeforeAll :表示在所有单元测试之前执行
@AfterAll :表示在所有单元测试之后执行



### 4、可观测性（ops运维监控）

SpringBoot 提供了 actuator 模块，可以快速暴露应用的所有指标
1、导入： spring-boot-starter-actuator

2、配置文件配置 management.endpoints.web.exposure.include=*

3、访问 http://localhost:8080/actuator；
展示出所有可以用的监控端点

```
management.endpoints.web.exposure.include=*
```











## 六、核心原理（自定义Starter）



### 1、基础抽取

1. 创建自定义starter项目，引入spring-boot-starter基础依赖
2. 编写模块功能，引入模块所有需要的依赖。

3. 编写xxxAutoConfiguration自动配置类，帮其他项目导入这个模块需要的所有组件
4. 别人导入我的stater只需要引入我的自动配置类



```
@EnableConfigurationProperties(Robot.class)
@Configuration //把这个场景要用的组件导入到容器中
public class RobotAutoConfiguration {

    @Bean
    public RobotService robotService(){
        return new RobotServiceImpl();
    }

    @Bean
    public Robotcontroller robotcontroller(){
        return new Robotcontroller();
    }
    
}
```

```
@Import(RobotAutoConfiguration.class)  第一层抽取
@SpringBootApplication
public class Springboot02DemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(Springboot02DemoApplication.class, args);
    }

}
```



### 2、@EnableXXX机制

1. 编写自定义 @EnableXxx 注解

2. @EnableXxx 导入 自动配置类
3. 测试功能组件生效

```
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(RobotAutoConfiguration.class)
public @interface EnableRobot {
    
}
```

```
@EnableRobot  //第二层抽取
@SpringBootApplication
public class Springboot02DemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(Springboot02DemoApplication.class, args);
    }

}
```





### 3、完全自动配置（用这个）

1、依赖 SpringBoot 的 SPI 机制
2、META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports 文件中编写好我们自动配置类的全类名即可
3、项目启动，自动加载我们的自动配置类

![image-20260429133430099](C:\Users\缪雨\AppData\Roaming\Typora\typora-user-images\image-20260429133430099.png)



**总结**

```
* 为什莫导入robot-spring-boot-starter，访问为404？
* 原因：主程序只会扫描到自己所在的包及其子包下的所有组件
*
* 自定义的starter：
* 1、第一层抽取：编写一个自动配置类，别人导入我的starter
*            无需关心需要给容器中导入那些组件，只需要导入自动配置类
*            自动配置类帮你给容器中导入所有的这个场景要用的组件
* 2、第二层抽取：只需要标注功能开关注解。
* 3、第三层抽取（推荐）：直接导入starter即可
```







# 掌握源码

![image-20260429133708129](C:\Users\缪雨\AppData\Roaming\Typora\typora-user-images\image-20260429133708129.png)

