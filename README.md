# DevelopmentLog
记录开发中遇到的一些问题以及解决方法/经验



### 跨域问题

#### 问题

发送数据后端无法接收。



#### 解决

vue中若使用vite需要在`vite.config.js`中配置代理，进行转发。

```js
server:{
    proxy:{
      '/api':{  // 获取路径中包含了/api的请求
        target: 'http://localhost:8080',  //后台服务所在的源
        changeOrigin: true, //修改源
        rewrite:(path)=>path.replace(/^\/api/, '') ///api替换为''
      }
    }
}
```



### Axios发送数据格式格式

#### 问题

后端使用`@RequestBody`注解，前端使用axios发送数据后端无法接收。



#### 解决

对于前端发送的数据一般有三种传输方式：在URI里，Form和Json格式；如果前端传递Form格式的数据，后端直接使用实体类接值即可；若前端使用Json格式传递数据，则后端需要使用`@Requestbody`注解进行接值。axios默认使用Json格式发送数据。



### 局域网内无法访问Vue服务

#### 问题

在局域网内无法访问公开的vue服务，本机通过`127.0.0.1`也无法访问，只有通过`localhost`可以访问。



#### 解决

这是vite的安全策略，如需更改需要将`package.json`中`scripts`中的`dev`改为`vite --host`，如下：

```js
"scripts": {
    "dev": "vite --host",	
    "build": "vite build",
    "preview": "vite preview"
  },
```



### RabbitMQ监听数据失败

#### 问题

某方法需要监听到`trade.topic`中的数据，对它配置如下

```java
@RabbitListener(bindings = @QueueBinding(
    value = @Queue(name = "cart.clear.queue", durable = "true"),
    exchange = @Exchange(name = "trade.topic"),
    key = "order.create"
))
```

但是一旦传递消息会报错：

```bash
inequivalent arg 'type' for exchange 'trade.topic' in vhost '/hmall': received 'direct' but current is 'topic'
```



#### 解决

对`@Exchange`注解中声明交换机的类型，更正后如下：

```java
@RabbitListener(bindings = @QueueBinding(
    value = @Queue(name = "cart.clear.queue", durable = "true"),
    exchange = @Exchange(name = "trade.topic", type = "topic"),	// 此处声明交换机类型
    key = "order.create"
))
```



### RabbitMQ接收数据反序列化导致数据类型更改

#### 问题

在使用`rabbitTemplate`的`convertAndSend`方法传递数据时，数据的类型在传递后发生变化。

场景为：需要传递一个`Set<Long>`类型的集合以及一个`Long`类型的userId，但是`converAndSend`方法只能传递一个object对象，所以将两个对象存放在一个`Map`里作为Message发出去。但是在消费者接收到消息后，数据类型分别变成了`ArrayList`和`Integer`。



#### 解决

上述问题的产生可能是由于Message对象的序列化问题导致的，其在官方文档中有说明：

> 从 `1.5.7`、`1.6.11`、`1.7.4` 和 `2.0.0` 版本开始，如果一个 message body 是一个序列化的 `Serializable` 的java对象，在执行 `toString()` 操作（如在日志消息中）时，它不再被反序列化（默认）。这是为了防止不安全的反序列化。默认情况下，只有 `java.util` 和 `java.lang` 类被反序列化。要恢复到以前的行为，你可以通过调用 `Message.addAllowedListPatterns(…)` 来添加允许的类/包 pattern。支持一个简单的通配符，例如 `com.something.*, *.MyClass`。不能被反序列化的 body 在日志消息中用 `byte[<size>]` 表示。

参考：https://springdoc.cn/spring-amqp/#message中末尾部分。

蕾蕾的方案为：将两个参数设计为一个对象来传递，实践后证明可行。如果传递的是多个对象应该聚合为一个类。

针对该问题（包含userId的），实际上有更好的解决办法，具体参考下个问题。



### RabbitMQ传递UserId

#### 问题

场景如上个问题：converAndSend方法只能传递一个object对象，而使用Map传递参数又有可能存在序列化错误的问题，仅两个对象又无合并为一个类的必要。



#### 解决

Message对象实际上携带了额外的`properties`参数，可以使用`properties`传递必要的参数，就类似于Http请求中的头部数据。

> 可以通过调用 `setHeader(String key, Object value)` 方法用用户定义的 'header' 来扩展这些属性。

也就是说可以使用header存储userId，具体实现方法是在`converAndSend`方法后再加一个匿名对象MessagePostProcessor，内部重写`postProcessMessage`方法：在内部使用setHeader方法将userId存入。实现如下：

```java
rabbitTemplate.convertAndSend("trade.topic", "order.create", itemIds, new MessagePostProcessor() {
    @Override
    public Message postProcessMessage(Message message) throws AmqpException {
        message.getMessageProperties().setHeader("userId", UserContext.getUser());
        return message;
    }
});
```

使用Lambda表达式简化后如下：

```java
rabbitTemplate.convertAndSend("trade.topic", "order.create", itemIds, message -> {
    message.getMessageProperties().setHeader("userId", UserContext.getUser());
    return message;
});
```

上述方法虽然解决了传递问题，但是实际上，很多service层中的方法的参数都不包含userId，这是因为userid往往被存于`ThreadLocal`中，随用随取。所以需要将userId存入`ThreadLocal`之中，而如果每次Listener都添加一次又十分繁琐，所以应该设计一个方法使得userId自动被存于`ThreadLocal`之中。

一个解决办法就是：通过重写消息转化器的`fromMessage`方法，使得每次消费者接收消息完成反序列化时自动将userId存于`TreadLocal`之中。该方法作为共有的配置类，存放于common模块下（这种共有的配置类一般会存放于一个普通的模块下）。具体实现如下：

```java
@Configuration
public class MqConfig {
    @Bean
    public MessageConverter messageConverter(){
        // 1.定义消息转换器
        Jackson2JsonMessageConverter jackson2JsonMessageConverter = new Jackson2JsonMessageConverter(){
            @Override
            public Object fromMessage(Message message) throws MessageConversionException {
                Long userId = message.getMessageProperties().getHeader("userId");
                if (ObjectUtil.isNotNull(userId)) {
                    UserContext.setUser(userId);
                }
                return super.fromMessage(message);
            }
        };
        // 2.配置自动创建消息id，用于识别不同消息，也可以在业务中基于ID判断是否是重复消息
        jackson2JsonMessageConverter.setCreateMessageIds(true);
        return jackson2JsonMessageConverter;
    }
}
```

该方法源于：https://b11et3un53m.feishu.cn/wiki/OQH4weMbcimUSLkIzD6cCpN0nvc?comment_id=7381508989902684163&comment_type=0&comment_anchor=true



### Vue页面铺满问题

#### 问题

vue默认无法全局铺满



#### 解决办法

给app.vue设置`style`属性，如下:

```vue
<style>
#app {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
}
</style>
```



### Vue+vite修改启动端口

#### 解决办法

在`package.json`中将启动项增添参数`--port 端口号`，如下：

```json
"scripts": {
    "dev": "vite --port 8000",
  },
```



### 微服务项目启动失败：缺失插件

#### 内容

对Api模块引入了一个实体类，该实体类继承自一个`PageQuery`类，内部有MyBatis的东西，启动任何引入了API的项目均报错。这是因为即便是引用也需要导入对应Maven工程



### ElasticSearch搜索条件为空

#### 内容

ElasticSearch的搜索条件为空时，如果Key为`""`或`null`时仍然算作是搜索条件，这时候应该进行判断处理：是否添加该条件



### 面试经背诵开始，从Java Guide开始



### API项目启动



### Ant Design Pro项目初始化

#### 内容

##### 创建

使用`pro create appname`创建项目，使用`yarn`安装依赖，在选择安装`simple`还是`complete`时，应选择`simple`便于二次开发，而选用`complete`则可能产生初始化失败的问题：路由问题。



##### 缩减项目

删除国际化：执行`npm run i18n-remove`删除国家化语言模块，手动删除`.\src\locales`目录，再删除`.\src\components\index.ts`中的`SelectLang`模块



### Ant Design Pro提交失败

#### 问题

报错信息为`.husky/_/husky.sh: No such file or directory`



#### 解决办法

将项目的`package.json`文件中`devDependencies`节点下的`husky`库删掉，然后删掉项目文件中的`.husky`文件夹，重新`yarn`一次即可。



### 解决Spring Boot启动警告的问题（强迫症

#### 问题

Spring Boot项目每次启动都会产生警告，如：

```bash
2024-08-21 12:19:22.370  WARN 22788 --- [  restartedMain] o.m.s.mapper.ClassPathMapperScanner      : Skipping MapperFactoryBean with name 'userMapper' and 'com.qingmuy.mapper.UserMapper' mapperInterface. Bean already defined with the same name!
2024-08-21 12:19:22.370  WARN 22788 --- [  restartedMain] o.m.s.mapper.ClassPathMapperScanner      : No MyBatis mapper was found in '[com.qingmuy.mapper]' package. Please check your configuration.
```

```bash
2024-08-21 12:19:24.036  WARN 22788 --- [  restartedMain] o.s.b.a.f.FreeMarkerAutoConfiguration    : Cannot find template location(s): [classpath:/templates/] (please add some templates, check your FreeMarker configuration, or set spring.freemarker.checkTemplateLocation=false)
```



#### 解决

对于第一个警告而言，这是说明一个Bean类被重复扫描，经过检测之后，这是由于在启动类和MyBatisPlus分页设置中均进行了`@MapperScan`注解，这导致mapper文件被重复扫描。只需要删除分页配置中的`@MapperScan`注解即可。

对于第二个警告：这是缺少模板的警告，只需要在`application.yaml`中配置即可：

```yaml
spring:
  freemarker:
    check-templateLocation: false
```



### Ant Design Pro登录两次

#### 问题

使用Ant Design Pro登录时，初次登录成功但是页面不跳转，第二次登录成功才发生跳转。



#### 解决

产生原因是因为在app.tsx中的layout里，有个切换页面判断逻辑，非登录页currentUser没有值就会重定向到登录页，而组件销毁了还在设置状态（setState方法是异步更新的）

所以设置登录状态后，不能立即跳转，需要设置一个延迟时间setTimeout。

或者是使用`window.location`，设置跳转

```js
window.location.href="/";
```



### API签名认证

**API 签名认证算法**是一种用于保证 API 请求的**安全性**和**完整性**的一种机制，通过在 API 请求中添加**签名字段**来验证请求发送者的身份，并确保请求在传输过程中没有被篡改。

其仅能保证请求未被篡改而不能保证请求是否泄露。



#### 作用

1. 身份验证：验证请求发送者的身份是否合法
2. 防止篡改：防止请求参数在传输过程中被篡改
3. 防止重放攻击

> **重放攻击**是一种网络安全攻击方式，攻击者会在网络通信过程中，通过截获合法的通信数据包，然后将这些数据包重新发送到目标系统，以欺骗系统进行非法操作或者获取敏感数据。



#### 实现思路

首先为用户分配两个字段：`accessKey`和`secretKey`，即用户标识和用于生成签名的密钥；类似公钥和私钥。



##### 生成签名

生成签名时，将请求参数拼接`secretKey`后，使用加密算法(一般为不可解加密，如：md5，sha256)，即得到签名`sign`。

> 请求参数  + sk 密钥 => 加密算法 => 签名



##### 校验签名(身份验证)

用户生成签名后，将`accessKey`与`签名`放入请求中一起发送(可放在请求头中)。

服务端接收请求后，从请求中取出`请求参数`、`accessKey`、`签名`。

通过`accessKey`查询指定的`secretKey`，取出请求参数，使用签名算法重新生成签名，对比该签名与请求中的签名是否相等，若相等即为校验成功，失败则反之。



##### 防止篡改

由于签名时使用请求参数拼接`secretKey`密钥后生成的，所以请求参数不同则签名不同，即类似哈希值的计算。

一旦请求参数被篡改则服务端计算出的签名与请求头的签名不同即签名校验失败。



##### 防止重放攻击

每次在请求头中添加一个随机数`nonce`(基本不会重复)，放入请求中，服务端每次保存使用过的随机数。

具体校验逻辑如下：

> 判断随机数是否已被保存在服务端。
>  如果是，则说明请求是重放的，拒绝请求。原因是，如果请求是合法请求重放的，对应的合法请求被服务端接收时，随机数就会被保存；请求被重放，随机数自然判断已存在。
>  如果不是，则说明是正常请求，保存其随机数。

虽然随机数可以防止重放攻击，但是若保存所有的随机数会导致系统资源的浪费；所以可以在请求中添加时间戳`timestamp`，如此只保存指定时间范围内的随机数，而不必保存全部随机数。

校验逻辑升级如下：

> 先判断时间戳是否在有效时间范围内（如 距当前时间 5 分钟范围内）：
>  如果不是，则拒绝请求；
>  如果是，则继续后面的逻辑判断。
>  
>
>  再判断随机数是否已被保存在服务端：
>  如果是，则说明请求是重放的，拒绝请求；
>  如果不是，则说明是正常请求，保存其随机数。

如何保存时间戳：

> 建议使用 redis 保存随机数，**可以将过期时间设置为时间戳的有效时间**，这样时间戳过期后，随机数也会自动移除。



服务端完整校验逻辑如下：

1. 先从请求中取得各个字段
2. 根据 accessKey 查询用户得到 secretKey
3. 服务端生成签名
4. 校验签名是否一致
5. 校验时间戳是否过期
6. 校验随机数是否存在



### 网关

所谓网关既是对前端请求的统一处理中心，形象理解为火车站的检票口。

其优点在于：统一进行一些操作、处理一些问题。



其作用有如下方面：

1. 路由
2. 鉴权
3. 跨域
4. 缓存
5. 流量染色
6. 访问控制
7. 统一业务处理
8. 发布控制
9. 负载均衡
10. 接口保护
    1. 限制请求
    2. 信息脱敏
    3. 降级（熔断）
    4. 限流
    5. 超时时间
11. 统一日志
12. 统一文档



#### 路由

起到转发作用，例如存在接口A和B，网关负责处理不同的请求，根据请求的地址和参数转发请求到对应的接口（服务器/集群）



#### 负载均衡

在路由的基础上负责多个服务、不同实例之间负载的平衡



#### 统一鉴权

对用户权限的校验，无论访问什么接口都进行判断权限，无需重复判断。



#### 统一处理跨域

由于是网关统一处理请求，所以只在网关处处理跨域问题即可



#### 统一业务处理

将重复的逻辑放到网关里统一处理，例如对接口调用的次数的统计



#### 访问控制

对访问进行控制，即黑白名单机制，限制了DDOS的IP



#### 发布控制

如灰度发布，在上线新接口时，先给新接口分配20%的流量，给旧接口分配80%的流量，以后调整比重



#### 流量染色

给请求添加独有的标识，防止绕过网关直接请求到接口



#### 统一接口保护

1. 限制请求
2. 信息脱敏
3. 降级（熔断）
4. 限流
5. 超时时间



#### 统一日志

统一的请求、响应信息记录



#### 统一文档

将下游项目的文档进行聚合，在一个页面内统一查看

可以参考：https://doc.xiaominfo.com/docs/middleware-sources/aggregation-introduction

相较于Spring cloud gateway更加便利



#### 网关的分类

1. 全局网关（接入层网关）：作用是负载均衡、请求日志等，与业务逻辑无关
2. 业务网关（微服务网关）：会有一些业务逻辑，将请求转发到不同的业务 / 项目 / 服务



#### 实现

1. Nginx（全局网关）、Kong弯管（API网关，分为社区版和专业版，收费，学习成本较高）
2. Spring Cloud GateWay（取代了Zuul）性能高、可以使用Java处理逻辑



### 注册中心

注册中心是服务实例信息的存储仓库，也是服务提供者和服务消费者进行交互的桥梁。它主要提供了服务注册和服务发现这两大核心功能。

![img](D:\Note\DevelopmentLog\assets\0da0f634e52a4683b6d615861c908605.jpeg)

针对服务的提供者而言，每当服务启动，就会通过这个注册中心的客户端组件，自动将自己注册到注册中心，这就是服务注册过程，有时候也叫服务发布过程。

而对于服务消费者来说，执行的是订阅操作而不是注册操作，也就是说它会对那些自己感兴趣的服务进行订阅，通过订阅操作就能从注册中心中，自动获取那些已经注册的服务提供者信息，这就是服务发现过程。

注册中心本质上是一种架构模型，目前业界主流的一些比较具有代表性的实现工具，包括 Consul 、Zookeeper、Eureka、Nacos 等。



### Linux防火墙端口设置

开启端口：

```bash
firewall-cmd --zone=public --add-port=port_num/tcp --permanent
```

关闭端口：

```bash
firewall-cmd --zone=public --remove-port=port_num/tcp --permanent
```

重启防火墙：

```bash
firewall-cmd --reload
service firewalld restart
```

查看已开放的端口列表：

```bash
firewall-cmd --permanent --list-port
```







### Zookeeper部署

#### Linux部署

在Linux上部署使用Docker即可，简化开发流程。

拉取镜像：

```bash
docker pull zookeeper
```

将其部署在 `/usr/local/zookeeper` 目录下：

```bash
cd /usr/local && mkdir zookeeper && cd zookeeper
```

创建data目录，用于挂载容器中的数据目录：

```bash
mkdir data
```

Docker部署镜像：

```bash
docker run -d -e TZ="Asia/Shanghai" -p 2181:2181 -v $PWD/data:/data --name zookeeper --restart always zookeeper
```



需要注意的地方在于：

大部分Docker的镜像被GFW屏蔽，目前可用的有：https://docker.1panel.live/

部署后端口可能被防火墙屏蔽，需要打开指定端口。



#### Windows部署

在[Zookeeper官网](https://zookeeper.apache.org/index.html)下载稳定版安装包。

使用如下命令解压tar包：

```bash
tar -zxvf apache-zookeeper-3.6.3.tar.gz
```

在解压后的文件目录下新增两个文件夹，一个命名为 data ，一个命名为 log。

找到解压目录下的 conf 目录，将目录中的 zoo_sample.cfg 文件，复制一份，重命名为 zoo.cfg。

修改 zoo.cfg 配置文件，将默认的 dataDir=/tmp/zookeeper 修改成 zookeeper 安装目录所在的data 文件夹，再增加数据日志的配置

```properties
dataLogDir=\\log
```

**注意**：路径需要使用`\\`而不是`\`

先启动`zkServer.cmd`，再启动`zkCli.cmd`



### Dubbo部署

**Dubbo的官方文档 = 💩**

由于Dubbo的依赖混乱等等，目前只摸索适用于Spring Boot 2的配置：

```xml
<properties>
    <dubbo.version>2.7.8</dubbo.version>
</properties>

<!-- Dubbo -->
<dependency>
    <groupId>org.apache.dubbo</groupId>
    <artifactId>dubbo-spring-boot-starter</artifactId>
    <version>${dubbo.version}</version>
</dependency>
<!--Zookeeper-->
<dependency>
    <groupId>org.apache.dubbo</groupId>
    <artifactId>dubbo-dependencies-zookeeper</artifactId>
    <version>${dubbo.version}</version>
    <type>pom</type>
    <exclusions>
        <exclusion>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-log4j12</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```

需要注意的是：

- 提供方和消费方的依赖版本必须一致

- `application.yaml`配置文件中关于Dubbo的配置必须要配置好连接重试时间：

  ```yaml
  dubbo:
    application:
      # 应用名称
      name: order-provider
    scan:
      # 接口实现者（服务实现）包
      base-packages: com.qingmuy.provider
    # 注册中心信息
    registry:
      address: zookeeper://127.0.0.1:2181
      timeout: 10000
    protocol:
      # 协议名称
      name: dubbo
      # 协议端口
      port: 20880
    config-center:
      timeout: 10000
  ```

- 即便是消费者和提供者双方使用了相同的依赖也可能存在依赖不一致的情况，具体参考：

  ![报错信息1](D:\Note\DevelopmentLog\assets\QQ图片20240913230514.png)

  ![报错图片2](D:\Note\DevelopmentLog\assets\QQ图片20240913230522.png)

  解决方案：导入相同版本的依赖的依赖

- 无论提供方还是提供方必须在主类上声明`@EnableDubbo`注解



### 对API开放平台项目的一些理解

接口的调用方式主要分为两种，一种是封装为SDK并提供给开发者直接调用，其中封装的SDK仅是对变量进行简单的处理，而后直接将请求发送给网关；另外一种是前端提供的测试调用，即在前端将参数传递给网关。

如果将接口平台视为允许用户上传自己的API的话，则应该注重处理SDK的内容，这里有两种处理方式：

1. 存在一个统一的SDK，内部封装所有方法，每次存在用户上传接口都会自动打包相关的接口信息和接收参数、返回值，具体的实现方法参考：[美团开放平台SDK自动生成技术与实践](https://tech.meituan.com/2023/01/05/openplatform-sdk-auto-generate.html)
2. 对每个API都封装一个特有的SDK。



对于网关的`流量染色`的处理方式：不应该使用简单的`source:gateway`继续校验，而是对每个不同接口设定一个特定的染色值用于校验（需要和接口提供者进行约定），进而避免其他流量或攻击。

此处染色的键命名为`source`，唯一密钥在数据库中命名为`sourceKey`，其生成自用户的唯一id与接口的唯一id等。



### 封装SDK内部AOP无法生效

自行封装了一个SDK，并在内部实现了自定义注解 + AOP环绕通知，其他项目引入SDK后使用注解AOP无法生效。



#### 解决

问题在于，SDK被引入后，项目默认扫描位置在启动类所在包下，但SDK并不处于扫描i位置，所以配置没有生效。

解决方法就是在SDK的主类上使用`@ComponentScan`注解指定位置即可



### MyBatis-Plus 3.4.3版本使用LambdaQueryWrapper查询报错

![img](D:\Note\DevelopmentLog\assets\H7%%TD(KD{0%{O3D`L_O)%5.png)

应将MyBatis-Plus版本提升至3.5.3



### TypeScript的基础使用

使用npm安装ts-node以及tsc，需要作为一个项目去运行：其配置很低，主要在于要有`tsconfig.json`，且`tsconfig.json`只需使用`tsc --init`生成即可，随后在项目中创建一个ts文件，使用`ts-node 文件名`进行编译。



### 保证Redis和Mysql的数据一致性

修改Redis中的数据导致的性能损失大于直接删除Redis中的性能损失，所以更新数据应直接删除数据再覆盖。

若业务要求短期一致性则应该加锁。



若要保证数据的强一致性只能通过上锁进行解决：但是上锁导致的性能损失与引入Redis提高性能这一目的冲突，所以应优先保证数据的最终一致性。

1. 先删缓存再删数据库
2. 先删数据库再删缓存



推荐使用方案二，其本身就可以保证数据一致性

方案一通过延迟双闪，尽管会导致一次脏数据读，但是保证最终一致性

方案二通过MQ进行删除重试，避免极端情况下删除缓存数据失败的情况



为了解决方案二的代码耦合问题，使用canal进行解耦：canal自动读取binlog（数据库日志），侦测到变化即通知到canal客户端使用消息队列删除缓存



### Canal的部署全过程

首先要准备好canal专用的数据库用户：

```sql
CREATE USER canal IDENTIFIED BY 'canal';  
```

该步骤创建的canal用户默认密码即为cana，实际上在实际场景下这个用户并不安全，应该有其他不易猜到的名字和密码：

```sql
create user '用户名'@'%' identified by '密码';
```

这里控制用户 Host 为`%`是因为 canal 被部署在虚拟机，所以要远程登录，该操作允许该用户使用远程登录，但**需要注意**的是：若 canal 与数据库部署在同一台机器，应控制 Host 为`localhost`，这是因为若开启远程登录则本地登录不可使用。

对该用户进行授权，`*.*`代表所有数据库：

```sql
GRANT SELECT, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'canal'@'%' (identified by '密码');
```

刷新权限：

```sql
FLUSH PRIVILEGES;
```

接着修改数据库的配置文件，mysql如下：

windows主机需要找到`my.ini`进行如下配置：

```ini
[mysqld]
# 打开binlog
log-bin=mysql-bin
# 选择ROW(行)模式
binlog-format=ROW
# 配置MySQL replaction需要定义，不要和canal的slaveId重复
server_id=1
```

对数据库进行重启即可：

```bash
# 使用管理员权限，进程名可能不同，该步骤同时完成了对权限的刷新
net stop mysql80
net start mysql80
```

如此完成了Mysql的配置，可使用如下语句检测是否配置成功：

```sql
# 是否开启mysql相关配置
show variables like 'log_bin';
show variables like 'binlog_format';
show binary logs;
# 查看用户权限
use mysql;
select * from user;
# 查看正在连接的用户
show processlist;
```



首先是使用 Docker 进行部署：

拉取镜像：

```bash
docker pull canal/canal-server:latest
```

运行 docker 镜像：

```dockerfile
docker run -p 11111:11111 --name canal -d canal/canal-server:version
```

进入容器内部修改配置文件：

```dockerfile
docker exec -it canal /bin/bash
```

其配置文件处于该路径下：

```bash
/home/admin/canal-server/conf/example/instance.properties
```

修改配置文件：

```properties
#################################################
## mysql serverId , v1.0.26+ will autoGen
canal.instance.mysql.slaveId=0		# 不能与数据库重复

# enable gtid use true/false
canal.instance.gtidon=false

# position info
canal.instance.master.address=127.0.0.1:3306		# 这里要修改为自己的数据库地址
canal.instance.master.journal.name=
canal.instance.master.position=
canal.instance.master.timestamp=
canal.instance.master.gtid=

# rds oss binlog
canal.instance.rds.accesskey=
canal.instance.rds.secretkey=
canal.instance.rds.instanceId=

# table meta tsdb info
canal.instance.tsdb.enable=true
#canal.instance.tsdb.url=jdbc:mysql://127.0.0.1:3306/canal_tsdb
#canal.instance.tsdb.dbUsername=canal
#canal.instance.tsdb.dbPassword=canal

#canal.instance.standby.address =
#canal.instance.standby.journal.name =
#canal.instance.standby.position =
#canal.instance.standby.timestamp =
#canal.instance.standby.gtid=

# username/password
canal.instance.dbUsername=canal		# 这里要修改为canal使用的数据库用户名
canal.instance.dbPassword=canal		# 这里要修改为canal使用的数据库密码
canal.instance.connectionCharset = UTF-8
# enable druid Decrypt database password
canal.instance.enableDruid=false
#canal.instance.pwdPublicKey=MFwwDQYJKoZIhvcNAQEBBQADSwAwSAJBALK4BUxdDltRRE5/zXpVEVPUgunvscYFtEip3pmLlhrWpacX7y7GCMo2/JM6LeHmiiNdH1FWgGCpUfircSwlWKUCAwEAAQ==

# table regex
canal.instance.filter.regex=.*\\..*		# 需要监听的数据库
# table black regex
canal.instance.filter.black.regex=mysql\\.slave_.*		# 数据库黑名单（不监听）
# table field filter(format: schema1.tableName1:field1/field2,schema2.tableName2:field1/field2)
#canal.instance.filter.field=test1.t_product:id/subject/keywords,test2.t_company:id/name/contact/ch
# table field black filter(format: schema1.tableName1:field1/field2,schema2.tableName2:field1/field2)
#canal.instance.filter.black.field=test1.t_product:subject/product_image,test2.t_company:id/name/contact/ch

# mq config
canal.mq.topic=example
# dynamic topic route by schema or table regex
#canal.mq.dynamicTopic=mytest1.user,topic2:mytest2\\..*,.*\\..*
canal.mq.partition=0
# hash partition config
#canal.mq.enableDynamicQueuePartition=false
#canal.mq.partitionsNum=3
#canal.mq.dynamicTopicPartitionNum=test.*:4,mycanal:6
#canal.mq.partitionHash=test.table:id^name,.*\\..*
#
# multi stream for polardbx
canal.instance.multi.stream.on=false
#################################################
```

最后重启该镜像即可。



直接在主机内部署Canal：

从[项目地址](https://github.com/alibaba/canal/releases)下载deplover版本，解压即可。在linux中解压指令如下：

```bash
tar -zxvf 压缩包名
```

解压完成后过程如上，启动与终止方式如下：

启动：在`/root/bin`下使用`./startup.sh`进行启动

停止：如上，使用`./stop.sh`停止

日志存放位置如下：`/root/logs/example/example.log`



在windows上部署Canal：

同上，不过启动方式为使用`/bin`目录下的`startup.dat`

要注意的是，需要修改`startup.dat`内的部分内容：

删掉启动配置里的：`-PermSize=128m`，以及`-Dlogback.configurationFile="%logback_configurationFile%"`即可。

如出现报错，可在文件的最底部加上`pause`查看报错



**需要特别注意的地方：**

由于是在虚拟机内部署的 docker + canal ， 以及在主机部署的数据库 ，所以产生了一个问题：canal无法连接到主机的Mysql，这个问题困扰了我很久：主要是由于**主机Windows防火墙的策略**导致的，通过 telnet 测试表明 3306 端口是不通的，这表明主机的防火墙在 tcp 层级拦截了请求，所以需要提前在防火墙上建立规则。



### Spring Boot 整合 Canal

首先导入maven：

```xml
<dependency>
    <groupId>top.javatool</groupId>
    <artifactId>canal-spring-boot-starter</artifactId>
    <version>1.2.1-RELEASE</version>
</dependency>
```

在`application.yaml`配置 canal：

```yaml
canal:
  server: 192.168.197.133:11111
  destination: example
```

建立类进行配置：

使用`@CanalTable`注解指定监听的数据库

实现`EntryHandler<实体类>`接口

该类可以重写`insert、 update、 delete`方法，即监听到相关操作即可接收到相关信息

```java
@CanalTable("user")
@Component
@Slf4j
public class ItemHandler implements EntryHandler<User> {
    
    @Resource
    StringRedisTemplate stringRedisTemplate;

    /**
     * 监听数据库插入新数据
     * @param user 新用户数据
     */
    @Override
    public void insert(User user) {
        log.info("监听到新用户注册：{}", user);
        Map<String, Object> usermap = BeanUtil.beanToMap(user, new HashMap<>(),
                CopyOptions.create()
                        .setIgnoreNullValue(true)
                        .setFieldValueEditor((fieldName, fieldValue) -> fieldValue.toString()));
        stringRedisTemplate.opsForHash().putAll("user:" + user.getId(), usermap);
    }
}
```



**需要注意的是：**

若数据库中的字段名与实体类定义的属性名不一致：即驼峰命名和下划线命名不一致，此时需要在实体类的属性上使用`@Column(name = "数据库字段名")`注解进行配置。



若 canal 接收到的数据库中的空字段会报错，若接收到一些特殊类型的数据也会报错（如LocalDataTime）：

这是由于`top.javatool.canal.client.util.StringConvertUtil.java`内部的`StringConvertUtil`这一方法仅实现了对一些常用类的转换，而对于空值和其他的类没有做处理，源代码如下：

```java
static Object convertType(Class<?> type, String columnValue) {
    if (columnValue == null) {
        return null;
    } else if (type.equals(Integer.class)) {
        return Integer.parseInt(columnValue);
    } else if (type.equals(Long.class)) {
        return Long.parseLong(columnValue);
    } else if (type.equals(Boolean.class)) {
        return convertToBoolean(columnValue);
    } else if (type.equals(BigDecimal.class)) {
        return new BigDecimal(columnValue);
    } else if (type.equals(Double.class)) {
        return Double.parseDouble(columnValue);
    } else if (type.equals(Float.class)) {
        return Float.parseFloat(columnValue);
    } else if (type.equals(Date.class)) {
        return parseDate(columnValue);
    } else {
        return type.equals(java.sql.Date.class) ? parseDate(columnValue) : columnValue;
    }
}
```

所以应该利用类加载器的加载顺序：双亲委派机制，只要创建一个同包同名的类。然后把原来的类复制过来，自己修改扩展即可，如下：

![image-20241010190443871](D:\Note\DevelopmentLog\assets\image-20241010190443871.png)

```java
static Object convertType(Class<?> type, String columnValue) {
    if (StringUtils.isBlank(columnValue)) {
        return null;
    } else if (type.equals(Integer.class)) {
        return Integer.parseInt(columnValue);
    } else if (type.equals(Long.class)) {
        return Long.parseLong(columnValue);
    } else if (type.equals(Boolean.class)) {
        return convertToBoolean(columnValue);
    } else if (type.equals(BigDecimal.class)) {
        return new BigDecimal(columnValue);
    } else if (type.equals(Double.class)) {
        return Double.parseDouble(columnValue);
    } else if (type.equals(Float.class)) {
        return Float.parseFloat(columnValue);
    } else if (type.equals(Date.class)) {
        return parseDate(columnValue);
    } else if (type.equals(LocalDateTime.class)) {
        // 这里对LocalDataTime类型做了处理
        DateTimeFormatter df = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
        return LocalDateTime.parse(columnValue,df);
    } else {
        return type.equals(java.sql.Date.class) ? parseDate(columnValue) : columnValue;
    }
}
```



还有个小问题：canal 的日志刷新频率很高，导致一直在刷屏，可以在`application.yaml`进行如下配置：

```yaml
logging:
  level:
    top.javatool.canal.client: warn
```

或者修改AbstractCanalClient，注释掉日志打印



实验报告

数据库

简历投递
