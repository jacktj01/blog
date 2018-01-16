> Spring Boot作为当下最流行的微服务项目构建基础，有的时候我们根本不需要额外的配置就能够干很多的事情，这得益于它的一个核心理念：“习惯优于配置”。。。
>
> 说白的就是大部分的配置都已经按照比较便准的编程规范配置好了

# 构建父工程



# 配置连接池

# 热部署

`pom.xml`添加依赖：

```
    <dependencies>
        <!--支持热启动jar包-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-devtools</artifactId>
            <!-- optional=true,依赖不会传递，该项目依赖devtools；之后依赖该项目的项目如果想要使用devtools，需要重新引入 -->
            <optional>true</optional>
        </dependency>
    </dependencies>
```

`application.yml`配置文件中添加：

```
spring:
  devtools:
    restart:
      #热部署生效 默认就是为true
      enabled: true
      #classpath目录下的WEB-INF文件夹内容修改不重启
      exclude: WEB-INF/**
```

关于DevTools的键值如下：
```
# DEVTOOLS (DevToolsProperties)
spring.devtools.livereload.enabled=true # Enable a livereload.com compatible server.
spring.devtools.livereload.port=35729 # Server port.
spring.devtools.restart.additional-exclude= # Additional patterns that should be excluded from triggering a full restart.
spring.devtools.restart.additional-paths= # Additional paths to watch for changes.
spring.devtools.restart.enabled=true # Enable automatic restart.
spring.devtools.restart.exclude=META-INF/maven/**,META-INF/resources/**,resources/**,static/**,public/**,templates/**,**/*Test.class,**/*Tests.class,git.properties # Patterns that should be excluded from triggering a full restart.
spring.devtools.restart.poll-interval=1000 # Amount of time (in milliseconds) to wait between polling for classpath changes.
spring.devtools.restart.quiet-period=400 # Amount of quiet time (in milliseconds) required without any classpath changes before a restart is triggered.
spring.devtools.restart.trigger-file= # Name of a specific file that when changed will trigger the restart check. If not specified any classpath file change will trigger the restart.

# REMOTE DEVTOOLS (RemoteDevToolsProperties)
spring.devtools.remote.context-path=/.~~spring-boot!~ # Context path used to handle the remote connection.
spring.devtools.remote.debug.enabled=true # Enable remote debug support.
spring.devtools.remote.debug.local-port=8000 # Local remote debug server port.
spring.devtools.remote.proxy.host= # The host of the proxy to use to connect to the remote application.
spring.devtools.remote.proxy.port= # The port of the proxy to use to connect to the remote application.
spring.devtools.remote.restart.enabled=true # Enable remote restart.
spring.devtools.remote.secret= # A shared secret required to establish a connection (required to enable remote support).
spring.devtools.remote.secret-header-name=X-AUTH-TOKEN # HTTP header used to transfer the shared secret.
```

当我们修改了java类后，IDEA默认是不自动编译的，而`spring-boot-devtools`又是监测`classpath`下的文件发生变化才会重启应用，所以需要设置IDEA的自动编译：

（1）**File-Settings-Compiler-Build Project automatically**

![](http://ojoba1c98.bkt.clouddn.com/img/spring-boot-learning/spring-boot-devtools01.png)

（2）**ctrl + shift + alt + /,选择Registry,勾上 Compiler autoMake allow when app running**

![](http://ojoba1c98.bkt.clouddn.com/img/spring-boot-learning/spring-boot-devtools02.png)

OK了，重启一下项目，然后改一下类里面的内容，IDEA就会自动去make了。

> **热部署可能会牺牲一定的系统性能，因为是动态的编译**

# 打包成可执行的Jar

默认情况下Spring Boot打包出来的jar包是不可执行的，