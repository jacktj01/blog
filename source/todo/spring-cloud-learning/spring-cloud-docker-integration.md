# Spring Cloud集成docker-maven-plugin构建Docker并提交到私有仓库

![](http://ojoba1c98.bkt.clouddn.com/img/spring-cloud-docker-integration/dev-ops)

> 微服务架构下，微服务在带来良好的设计和架构理念的同时，也带来了运维上的额外复杂性，尤其是在服务部署和服务监控上。单体应用是集中式的，就一个单体跑在一起，部署和管理的时候非常简单，而微服务是一个网状分布的，有很多服务需要维护和管理，对它进行部署和维护的时候则比较复杂。

# 准备工作

- 安装Docker
- IDE（使用IDEA）
- Maven环境
- Docker私有仓库

集成Docker需要的插件`docker-maven-plugin`：*[https://github.com/spotify/docker-maven-plugin](https://github.com/spotify/docker-maven-plugin)*

此篇使用`Spring Cloud Eureka`作为例子

# Maven setting.xml配置

`settings.xml`配置私有库的访问：

首先使用你的私有仓库访问密码生成主密码：

```
mvn --encrypt-master-password <password>

```

其次在`settings.xml`文件的同级目录创建`settings-security.xml`文件，将主密码写入：

```
<?xml version="1.0" encoding="UTF-8"?>
<settingsSecurity>
  <master>{Ns0JM49fW9gHMTZ44n*****************=}</master>
</settingsSecurity>

```

最后使用你的私有仓库访问密码生成服务密码，将生成的密码写入到`settings.xml`的`<services>`中（可能会提示目录不存在，解决方法是创建一个.m2目录并把`settings-security.xml`复制进去）

```
mvn --encrypt-password <password>
{D9YIyWYvtYsHayLjIenj***********=}
```
```
 <server>
      <id>docker-registry</id>
      <username>admin</username>
      <password>{D9YIyWYvtYsHayLjIenj***********=}</password>
      <configuration>
        <email>yangbingdong1994@gmail.com</email>
      </configuration>
    </server>
```

# Step1、利用IDEA的Spring Initializr构建高可用Eureka工程

项目结构：

![](http://ojoba1c98.bkt.clouddn.com/img/spring-cloud-docker-integration/structure.png)

```
@SpringBootApplication
@EnableEurekaServer
public class EurekaserverApplication {

	public static void main(String[] args) {
		SpringApplication.run(EurekaserverApplication.class, args);
	}
}
```

`application-peer1.properties`:

```
server.port=5001
spring.application.name=eureka-center-server
eureka.instance.hostname=peer1
#eureka.instance.prefer-ip-address=true
#eureka.instance.instance-id=${spring.cloud.client.ipAddress}:${server.port}
#eureka.client.register-with-eureka=false
eureka.client.fetch-registry=true
eureka.server.enable-self-preservation=false
#eureka.instance.ip-address=true
spring.output.ansi.enabled=ALWAYS
eureka.client.serviceUrl.defaultZone=http://peer2:5002/eureka/
```

`application-peer2.properties`:
```
server.port=5002
spring.application.name=eureka-center-server
eureka.instance.hostname=peer2
#eureka.instance.prefer-ip-address=true
#eureka.instance.instance-id=${spring.cloud.client.ipAddress}:${server.port}
#eureka.client.register-with-eureka=false
eureka.client.fetch-registry=true
eureka.server.enable-self-preservation=false
#eureka.instance.ip-address=true
spring.output.ansi.enabled=ALWAYS
eureka.client.serviceUrl.defaultZone=http://peer1:5001/eureka/
```



# Step2、创建Dockerfile

在`src/main`下面新建`docker`文件夹，并创建`Dockerfile`：

```
FROM frolvlad/alpine-oraclejdk8:slim
MAINTAINER ybd <yangbingdong1994@gmail.com>
VOLUME /tmp
ENV PROJECT_NAME="@project.build.finalName@.@project.packaging@" JAVA_OPTS=""
ADD $PROJECT_NAME app.jar
RUN sh -c 'touch /app.jar'
CMD ["sh", "-c", "java $JAVA_OPTS -Djava.security.egd=file:/dev/./urandom -Dspring.profiles.active=${ACTIVE:-docker}  -jar /app.jar"]
# ENTRYPOINT [ "sh", "-c", "java $JAVA_OPTS -Djava.security.egd=file:/dev/./urandom -jar /app.jar" ]
```

# Step3、添加插件

在完整的`pom.xml`

```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>

	<groupId>com.ybd.server</groupId>
	<artifactId>eureka-center-server</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<packaging>jar</packaging>
	<name>eureka-center-server</name>
	<description>统一服务注册中心</description>

    <properties>
        <resources.plugin.version>3.0.2</resources.plugin.version>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <java.version>1.8</java.version>
        <spring-boot.version>1.5.9.RELEASE</spring-boot.version>
        <spring-boot-maven-plugin.version>1.5.9.RELEASE</spring-boot-maven-plugin.version>
        <spring-cloud.version>Edgware.RELEASE</spring-cloud.version>
        <maven.test.skip>true</maven.test.skip>

        <docker.plugin.version>1.0.0</docker.plugin.version>
        <dockerfile.compiled.position>${project.build.directory}/docker</dockerfile.compiled.position>
        <docker.registry.name>discover-server</docker.registry.name>
        <docker.registry.url>192.168.6.113:8888</docker.registry.url>
        <docker.skip.build>false</docker.skip.build>
        <docker.push.image>true</docker.push.image>
    </properties>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-dependencies</artifactId>
                <version>${spring-boot.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

	<dependencies>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-eureka-server</artifactId>
		</dependency>
	</dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <version>${spring-boot-maven-plugin.version}</version>
                <executions>
                    <execution>
                        <goals>
                            <goal>repackage</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>

            <!-- resources插件，使用@变量@形式获取Maven变量到Dockerfile中 -->
            <plugin>
                <artifactId>maven-resources-plugin</artifactId>
                <version>${resources.plugin.version}</version>
                <executions>
                    <execution>
                        <id>prepare-dockerfile</id>
                        <phase>validate</phase>
                        <goals>
                            <goal>copy-resources</goal>
                        </goals>
                        <configuration>
                            <!-- 编译后Dockerfile的输出位置 -->
                            <outputDirectory>${dockerfile.compiled.position}</outputDirectory>
                            <resources>
                                <!-- Dockerfile位置 -->
                                <resource>
                                    <directory>${project.basedir}/src/main/docker</directory>
                                    <filtering>true</filtering>
                                </resource>
                            </resources>
                        </configuration>
                    </execution>
                </executions>
            </plugin>

            <!-- 集成Docker maven 插件 -->
            <plugin>
                <groupId>com.spotify</groupId>
                <artifactId>docker-maven-plugin</artifactId>
                <version>${docker.plugin.version}</version>
                <executions>
                    <execution>
                        <phase>package</phase>
                        <goals>
                            <goal>build</goal>
                        </goals>
                    </execution>
                    <execution>
                        <id>push-image</id>
                        <phase>deploy</phase>
                        <goals>
                            <goal>push</goal>
                        </goals>
                        <configuration>
                            <imageName>${docker.registry.url}/${docker.registry.name}/${project.artifactId}:latest</imageName>
                        </configuration>
                    </execution>
                </executions>
                <configuration>
                    <!--配置变量，包括是否build、imageName、imageTag，非常灵活-->
                    <skipDocker>${docker.skip.build}</skipDocker>
                    <!--最后镜像产生了两个tag，版本和和最新的-->
                    <forceTags>true</forceTags>
                    <imageTags>
                        <imageTag>${project.version}</imageTag>
                        <imageTag>latest</imageTag>
                    </imageTags>
                    <!--install阶段也上传，否则只有deploy阶段上传-->
                    <pushImage>${docker.push.image}</pushImage>
                    <!--时区配置-->
                    <env>
                        <TZ>Asia/Shanghai</TZ>
                    </env>
                    <runs>
                        <run>ln -snf /usr/share/zoneinfo/$TZ /etc/localtime</run>
                        <run>echo $TZ > /etc/timezone</run>
                        <run>wget https://raw.githubusercontent.com/vishnubob/wait-for-it/master/wait-for-it.sh</run>
                        <run>chmod 777 wait-for-it.sh</run>
                    </runs>
                    <!-- 配置镜像名称，遵循Docker的命名规范： springio/image -->
                    <imageName>${docker.registry.url}/${docker.registry.name}/${project.artifactId}</imageName>
                    <!-- Dockerfile位置，由于配置了编译时动态获取Maven变量，真正的Dockerfile位于位于编译后位置 -->
                    <dockerDirectory>${dockerfile.compiled.position}</dockerDirectory>
                    <resources>
                        <!-- 构建时需要的资源文件，这些文件和Dockerfile放在一起，这里只需要Spring Boot生成的jar文件即可 -->
                        <resource>
                            <targetPath>/</targetPath>
                            <directory>${project.build.directory}</directory>
                            <include>${project.build.finalName}.jar</include>
                        </resource>
                    </resources>
                    <!--push到私有的hub-->
                    <serverId>docker-registry</serverId>
                    <registryUrl>192.168.6.113:8888</registryUrl>
                </configuration>
            </plugin>
        </plugins>
    </build>

    <distributionManagement>
        <repository>
            <id>nexus-releases</id>
            <name>Nexus Release Repository</name>
            <url>http://192.168.0.200:8081/repository/maven-releases/</url>
        </repository>
        <snapshotRepository>
            <id>nexus-snapshots</id>
            <name>Nexus Snapshot Repository</name>
            <url>http://192.168.0.200:8081/repository/maven-snapshots/</url>
        </snapshotRepository>
    </distributionManagement>
</project>

```

**说明**：

* 这里的`serverId`要与maven `setting.xml`里面的一样


* Dockerfile构建文件在`src/main/docker`中
* 如果Dockerfile文件需要maven构建参数（比如需要构建后的打包文件名等），则使用`@@`占位符（如`@project.build.finalName@`）原因是Sping Boot 的pom将resource插件的占位符由`${}`改为`@@`，非继承Spring Boot 的pom文件，则使用`${}`占位符
* 如果不需要动态生成Dockerfile文件，则可以将Dockerfile资源拷贝部分放入docker-maven-plugin插件的`<resources>`配置里
* `spring-boot-maven-plugin`插件一定要在其他构建插件之上，否则打包文件会有问题。

# Step4、构建

如果`<pushImage>false</pushImage>`则install阶段将不提交Docker镜像，只有maven的`deploy`阶段才提交。

```
mvn clean install
```

```
[INFO] --- spring-boot-maven-plugin:1.5.9.RELEASE:repackage (default) @ eureka-center-server ---
[INFO] 
[INFO] --- docker-maven-plugin:1.0.0:build (default) @ eureka-center-server ---
[INFO] Using authentication suppliers: [ConfigFileRegistryAuthSupplier, NoOpRegistryAuthSupplier]
[WARNING] Ignoring run because dockerDirectory is set
[INFO] Copying /home/ybd/data/git-repo/bitbucket/ms-iba/eureka-center-server/target/eureka-center-server-0.0.1-SNAPSHOT.jar -> /home/ybd/data/git-repo/bitbucket/ms-iba/eureka-center-server/target/docker/eureka-center-server-0.0.1-SNAPSHOT.jar
[INFO] Copying /home/ybd/data/git-repo/bitbucket/ms-iba/eureka-center-server/target/docker/eureka-center-server-0.0.1-SNAPSHOT.jar -> /home/ybd/data/git-repo/bitbucket/ms-iba/eureka-center-server/target/docker/eureka-center-server-0.0.1-SNAPSHOT.jar
[INFO] Copying /home/ybd/data/git-repo/bitbucket/ms-iba/eureka-center-server/target/docker/Dockerfile -> /home/ybd/data/git-repo/bitbucket/ms-iba/eureka-center-server/target/docker/Dockerfile
[INFO] Building image 192.168.6.113:8888/discover-server/eureka-center-server
Step 1/7 : FROM frolvlad/alpine-oraclejdk8:slim

 ---> 491f45037124
Step 2/7 : MAINTAINER ybd <yangbingdong1994@gmail.com>

 ---> Using cache
 ---> 016c2033bd32
Step 3/7 : VOLUME /tmp

 ---> Using cache
 ---> d2a287b6ed52
Step 4/7 : ENV PROJECT_NAME="eureka-center-server-0.0.1-SNAPSHOT.jar" JAVA_OPTS=""

 ---> Using cache
 ---> 34565a7de714
Step 5/7 : ADD $PROJECT_NAME app.jar

 ---> 64d9055ce969
Step 6/7 : RUN sh -c 'touch /app.jar'

 ---> Running in 66f4eb550a57
Removing intermediate container 66f4eb550a57
 ---> 93486965cad9
Step 7/7 : CMD ["sh", "-c", "java $JAVA_OPTS -Djava.security.egd=file:/dev/./urandom -Dspring.profiles.active=${ACTIVE:-docker}  -jar /app.jar"]

 ---> Running in 8b42c471791f
Removing intermediate container 8b42c471791f
 ---> 2eb3dbbab6c5
ProgressMessage{id=null, status=null, stream=null, error=null, progress=null, progressDetail=null}
Successfully built 2eb3dbbab6c5
Successfully tagged 192.168.6.113:8888/discover-server/eureka-center-server:latest
[INFO] Built 192.168.6.113:8888/discover-server/eureka-center-server
[INFO] Tagging 192.168.6.113:8888/discover-server/eureka-center-server with 0.0.1-SNAPSHOT
[INFO] Tagging 192.168.6.113:8888/discover-server/eureka-center-server with latest
[INFO] Pushing 192.168.6.113:8888/discover-server/eureka-center-server
The push refers to repository [192.168.6.113:8888/discover-server/eureka-center-server]
40566d372b69: Pushed 
40566d372b69: Layer already exists 
4fd38f0d6712: Layer already exists 
d7cd646c41bd: Layer already exists 
ced237d13962: Layer already exists 
2aebd096e0e2: Layer already exists 
null: null 
null: null 
[INFO] 
[INFO] --- maven-install-plugin:2.4:install (default-install) @ eureka-center-server ---
[INFO] Installing /home/ybd/data/git-repo/bitbucket/ms-iba/eureka-center-server/target/eureka-center-server-0.0.1-SNAPSHOT.jar to /home/ybd/data/application/maven/maven-repo/com/iba/server/eureka-center-server/0.0.1-SNAPSHOT/eureka-center-server-0.0.1-SNAPSHOT.jar
[INFO] Installing /home/ybd/data/git-repo/bitbucket/ms-iba/eureka-center-server/pom.xml to /home/ybd/data/application/maven/maven-repo/com/iba/server/eureka-center-server/0.0.1-SNAPSHOT/eureka-center-server-0.0.1-SNAPSHOT.pom
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 15.962 s
[INFO] Finished at: 2017-12-25T13:33:39+08:00
[INFO] Final Memory: 55M/591M
[INFO] ------------------------------------------------------------------------
```

可以看到本地以及私有仓库都多了一个镜像：

![](http://ojoba1c98.bkt.clouddn.com/img/spring-cloud-docker-integration/portainer.png)

![](http://ojoba1c98.bkt.clouddn.com/img/spring-cloud-docker-integration/harbor.png)

**此处有个疑问**，很明显看得出来这里上传了两个一样大小的包，不知道是不是同一个jar包，但id又不一样：

![](http://ojoba1c98.bkt.clouddn.com/img/spring-cloud-docker-integration/duplicate01.png)

![](http://ojoba1c98.bkt.clouddn.com/img/spring-cloud-docker-integration/duplicate02.png)

# Step5、运行

运行程序

```
docker run --name discover-server1 -e ACTIVE=peer1 -p 5001:5001 -d --network host [IMAGE]
# 限制内存加上：-e "JAVA_OPTS=-Xmx128m"
```

就是这么简单粗暴。

# Docker Compose

`docker-compose.yml`:

```
version: '3'
services:
  eureka1:
    image: 192.168.6.113:8888/discover-server/eureka-center-server
    ports:
      - "5001:5001"
    environment:
      - ACTIVE=peer1
    network_mode: "host"
  eureka2:
    image: 192.168.6.113:8888/discover-server/eureka-center-server
    ports:
      - "5002:5002"
    environment:
      - ACTIVE=peer2
    network_mode: "host"
```

compose启动 `docker-compse up -d`

![](http://ojoba1c98.bkt.clouddn.com/img/spring-cloud-docker-integration/compose-up02.png)

![](http://ojoba1c98.bkt.clouddn.com/img/spring-cloud-docker-integration/compose-up02.png)

# Finally

> 参考
>
> [***http://blueskykong.com/2017/11/02/dockermaven/***](http://blueskykong.com/2017/11/02/dockermaven/)
>
> 源码
>
> [***https://github.com/masteranthoneyd/spring-boot-docker-demo***](https://github.com/masteranthoneyd/spring-boot-docker-demo ) 
