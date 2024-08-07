# DevelopmentLog
记录开发中遇到的一些问题以及解决方法/经验



### 跨域问题

#### Tag

前端 vue 跨域 经典



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

前端  vue axios 数据格式



#### 问题

后端使用`@RequestBody`注解，前端使用axios发送数据后端无法接收。



#### 解决

对于前端发送的数据一般有三种传输方式：在URI里，Form和Json格式；如果前端传递Form格式的数据，后端直接使用实体类接值即可；若前端使用Json格式传递数据，则后端需要使用`@Requestbody`注解进行接值。axios默认使用Json格式发送数据。



### 局域网内无法访问Vue服务

#### Tag

前端 vue 网络



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

后端 rabbitMq



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

