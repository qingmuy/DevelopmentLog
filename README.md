# DevelopmentLog
记录开发中遇到的一些问题以及解决方法/经验



### 跨域问题

#### Tag

vue 跨域 经典



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

#### Tag

vue axios 数据格式



#### 问题

后端使用`@RequestBody`注解，前端使用axios发送数据后端无法接收。



#### 解决

对于前端发送的数据一般有三种传输方式：在URI里，Form和Json格式；如果前端传递Form格式的数据，后端直接使用实体类接值即可；若前端使用Json格式传递数据，则后端需要使用`@Requestbody`注解进行接值。axios默认使用Json格式发送数据。



### 局域网内无法访问Vue服务

#### Tag

vue 网络



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

#### Tag

rabbitMq



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

#### Tag

rabbitMq 数据格式



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

#### Tag

rabbitMq



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

#### Tag

vue 页面设计



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

#### Tag

vue



#### 解决办法

在`package.json`中将启动项增添参数`--port 端口号`，如下：

```json
"scripts": {
    "dev": "vite --port 8000",
  },
```



### 微服务项目启动失败：缺失插件

#### Tag

SpringCloud



#### 内容

对Api模块引入了一个实体类，该实体类继承自一个`PageQuery`类，内部有MyBatis的东西，启动任何引入了API的项目均报错。这是因为即便是引用也需要导入对应Maven工程



### ElasticSearch搜索条件为空

#### Tag

ElasticSearch



#### 内容

ElasticSearch的搜索条件为空时，如果Key为`""`或`null`时仍然算作是搜索条件，这时候应该进行判断处理：是否添加该条件



### 面试经背诵开始，从Java Guide开始



### API项目启动
