本文和需要用到的两个Springboot程序我放到了github上, [**可以直接下载**](https://github.com/Rock2z/reposity/blob/master/nginxTestSpringboot.zip)


## 一.基本原理
##### 前置副本:什么是nginx呢
在我的理解来说:
nginx就是一个**HTTP服务器**, 或者说的更具体他常被用作一个**静态资源服务器**, 可以解析请求然后返回对应的静态文件, 比如html, css, 图片视频什么的, 但是不能处理动态请求, 比如jsp之类的, 就类似于apache, 要处理动态请求得借助像php这样的脚本语言;

那么为什么要用nginx呢, 因为他能实现非常**高效的反向代理和负载均衡**, 性能非常好;

#### 1.反向代理: 
就是我访问nginx的服务器, 比如我要访问天猫网站的搜索页面, 但是我不访问```tmall.com```(假设这是一个tomcat服务器), 而是去访问```xyz.com```(假设这是一个nginx服务器), 但在这个nginx服务器上并没有完整的搜索页面, 他就会把请求转发给tomcat, 然后获得response之后发给客户, 对于客户而言看似就是从nginx服务器上获取了页面, 但其实nginx上是没有页面的, 他是反向代理了tomcat;


#### 1.2正向代理: 
既然有反向代理自然有正向代理啦, 比如我要访问```tudou.com```, 但是我使用了正向代理, 比如vpn, 那么我输入```tudou.com```这个网址访问, 其实我们访问的是另外一个服务器, 而不是土豆的服务器, 然后再由那个服务器去访问```tudou.com```并把获得的页面返回给客户端, 在用户看来似乎就是直接访问了```tudou.com```;


#### 2.动静结合
其实在nginx服务器上也并不是什么都没有, 上面会存有一些静态资源, 比如我要访问天猫网站的搜索界面, 其中搜索内容是变化的, 是动态的, 要去向tomcat去request, 但是有一些内容是静态的, 比如```footer.html```这样的页眉页脚, 还有一些图片视频啥的, 是可以直接放在nginx上的, 那么这些内容就不需要向tomcat去请求了, 这样就非常大程度上减小了tomcat的压力, 实现动静结合;

#### 3.负载均衡
在nginx的配置中配置反向代理后, 可以配置反向代理到多个服务器, 对每个服务器设置对应的weight权重, 就可以简单的配置负载均衡了;

#### 4.Session共享
比如我现在有两个tomcat服务器做了按负载均衡, 然后我第一次连接练到了A服务器, 并且登录了, 然后这时候突然被我的流量被分到了B服务器上, 但是此时我在B服务器上还没有登录, 所以我要重新登录, 这样就很不方便, 所以需要做多个负载均衡的服务器上的Session共享;
<br/><br/><br/>

# 二.代码实现
#### 前置副本:准备两个完全一样的Springboot程序并且配置到不同端口模拟多服务器(我配置到了8081和8082端口)
这里两个Springboot程序我放到了github上, [**可以直接下载**](https://github.com/Rock2z/reposity/blob/master/nginxTestSpringboot.zip)
说一下这里面我做了什么:
首先我写了两个简单的springboot项目, 分别配置到了8081和8082端口, 然后实现了这样的功能, 当你访问```127.0.0.1/hello```的时候, 会看到一个妹子照片, 此时在session中写入一个user变量, 然后在访问```127.0.0.1/who```的时候会显示出来session中的user的值, 并清空session中的user变量, 所以刷新who页面之后就看不到session中user的内容了;
然后在tomcat的控制台中, 可以看到当前访问了什么样的url;

**最终我希望实现什么样的结果呢:
当我配置好了反向代理, 负载均衡, 动静结合, session共享之后;
我应该能看到: 我访问```127.0.0.1:9090/hello```这是nginx的服务器端口, 然后我能进入到8081或者8082服务器的hello页面, 并且设置session, 再进入who页面, 查看到session, 刷新后session清楚, 显示nothing; 同时当我们查看tomcat的控制台输出的时候, 可以看到8081和8082的服务器都有流量访问, 而且是在nginx负载均衡的情况下, 各自有概率地访问, 而且看到静态资源比如说图片并没有走tomcat获取, 而是走nginx获取**

这里在没有配置session共享之前, 会查看到进入hello页面后设置了session, 但是进入who页面不一定能看到session中user的内容, 因为此时可能访问的是还没有设置过session的另一个服务器;

#### 1.配置反向代理
我使用的是mac系统, 关于nginx安装配置的内容可以看我这篇博客[**mac下安装配置nginx**](https://blog.csdn.net/qq_33982232/article/details/89201468)
首先找到```nginx.conf```这个文件, 这是对nginx做配置的配置文件, 找到这一段, 改成:
```plain
        location / {
            proxy_pass http://127.0.0.1:8081;
            root   html;
            index  index.html index.htm;
    }
```
这里的意思是把所有的流量全部通过proxy代理到```http://127.0.0.1:8081```这个本地的8081端口;
这里就实现了反向代理;

#### 2.动静结合
在这里我们希望静态资源使用nginx获取, 而动态资源从tomcat获取, 在```nginx.conf```文件中, 在上面的配置信息的下方, 加上一段:
```plain
        location / {
            proxy_pass http://tomcat80818082;
            root   html;
            index  index.html index.htm;
        }

        location ~\.(css|js|jpg)$ {
        	root /your_ocation/nginx/nginxtest2/src/main/resources/static;
	    }
```
注意, 这一段```root /your_ocation/nginx/nginxtest2/src/main/resources/static;```要改成你自己的静态资源放置的位置, 可以进入我前面提供的springboot项目文件, 任选一个进入其中的```nginxtest2/src/main/resources/static```这个目录下获取真实路径复制进来即可;

#### 3.负载均衡
我之前已经准备好了两个tomcat的服务器, 分别在8081和8082端口, 所以我现在要做这两个服务器的负载均衡, 首先创建一个upstream, 在nginx.conf大概这样操作:
```plain
    #keepalive_timeout  0;
    keepalive_timeout  65;

    #gzip  on;

    upstream tomcat80818082{
    server  127.0.0.1:8081  weight=1;
    server  127.0.0.1:8082 weight=2;
    }

    server {
        listen       9090;
        server_name  localhost;
```

新建一个upstream, 注意名字不能有下划线, 然后在其中配置服务器的weight表示负载均衡是按照权重来做的;

然后把第一步中反向代理的服务器地址改为:
```plain
        location / {
            proxy_pass http://tomcat80818082;
            root   html;
            index  index.html index.htm;
        }
```
讲道理到这里为止, 已经配置完了反向代理, 动静结合和负载均衡, 已经可以看到的效果是, 访问localhost的9090端口的hello页面, 得到的8081或者8082端口的服务器的页面内容, 发现tomcat控制台中显示加载的图片没有从tomcat走, 刷新几次hello页面发现, 基本上8081和8082的流量是1:2的;

#### 4.Session共享
现在要做Session的共享, 问题在于, 做完前一步, 我们希望可以看到先进入hello页面设置了session, 然后进入who界面可以看到session中的user的值, 但是实际上有的时候, 可能hello访问的是8081服务器但是who页面就访问到了8082服务器, 但是此时在8082服务器中还没有设置session中的user, 所以看到的是nothing;
为了解决这个问题我们需要做Session共享;(这一部分我已经放在我前面提供的项目文件里了)

首先要在maven中导入两个包, 用于session共享
```xml
<dependency>  
        <groupId>org.springframework.boot</groupId>  
        <artifactId>spring-boot-starter-redis</artifactId>  
</dependency>  
<dependency>  
        <groupId>org.springframework.session</groupId>  
        <artifactId>spring-session-data-redis</artifactId>  
</dependency> 
```

然后要配置一个```RedisSessionConfig```来配置redis的session
```java


@Configuration  
@EnableRedisHttpSession  
public class RedisSessionConfig {  
}
```
然后在spring的propoties中配置redis的相关配置
```xml
spring.redis.host=localhost  
spring.redis.port=6379
```
这样就能实现把Session存储到redis里
到这里就实现了session共享, 可以看到最终效果了~