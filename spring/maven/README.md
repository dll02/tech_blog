# MAVEN

## maven 常用仓库镜像

* 阿里云
```bash
<mirror>
    <id>alimaven</id>
    <name>aliyun maven</name>
    <url>http://maven.aliyun.com/nexus/content/groups/public/</url>
    <mirrorOf>central</mirrorOf>
</mirror>
``` 
* Maven 公共代理库
```bash
<mirror>
    <id>alimaven public</id>
    <name>aliyun maven</name>
    <url>http://maven.aliyun.com/nexus/content/groups/public/</url>
    <mirrorOf>central</mirrorOf>
</mirror>
<mirror>
  <id>alimaven central</id>
  <name>aliyun maven</name>
  <url>http://maven.aliyun.com/nexus/content/repositories/central/</url>
  <mirrorOf>central</mirrorOf>
</mirror>
<mirror>
  <id>alimaven apache-snapshots</id>
  <name>apache-snapshots</name>
  <url>https://maven.aliyun.com/repository/apache-snapshots</url>
  <mirrorOf>apache-snapshots</mirrorOf>
</mirror>

# *号包含所有仓库
<mirror>
    <id>aliyunmaven</id>
    <mirrorOf>*</mirrorOf>
    <name>阿里云公共仓库</name>
    <url>https://maven.aliyun.com/repository/public</url>
</mirror>
``` 

* 华为云
```bash
<mirror>
    <id>huaweicloud</id>
    <name>mirror from maven huaweicloud</name>
    <url>https://mirror.huaweicloud.com/repository/maven/</url>
    <mirrorOf>central</mirrorOf>
</mirror>
```  


* 腾讯云
```bash
<mirror>
　　<id>nexus-tencentyun</id>
　　<mirrorOf>*</mirrorOf>
　　<name>Nexus tencentyun</name>
　　<url>http://mirrors.cloud.tencent.com/nexus/repository/maven-public/</url>
</mirror>

``` 

## 代理的仓库列表

| 仓库名         | 远程仓库URL                                       | 阿里云镜像URL（或Nexus地址）                               |
| -------------- | ------------------------------------------------- | -------------------------------------------------------- |
| central        | https://repo1.maven.org/maven2/                   | https://maven.aliyun.com/repository/central 或 https://maven.aliyun.com/nexus/content/repositories/central |
| jcenter        | http://jcenter.bintray.com/                      | https://maven.aliyun.com/repository/jcenter 或 https://maven.aliyun.com/nexus/content/repositories/jcenter |
| public         | central仓和jcenter仓的聚合仓                       | https://maven.aliyun.com/repository/public 或 https://maven.aliyun.com/nexus/content/groups/public |
| google         | https://maven.google.com/                        | https://maven.aliyun.com/repository/google 或 https://maven.aliyun.com/nexus/content/repositories/google |
| gradle-plugin  | https://plugins.gradle.org/m2/                   | https://maven.aliyun.com/repository/gradle-plugin 或 https://maven.aliyun.com/nexus/content/repositories/gradle-plugin |
| spring         | http://repo.spring.io/libs-milestone/             | https://maven.aliyun.com/repository/spring 或 https://maven.aliyun.com/nexus/content/repositories/spring |
| spring-plugin  | http://repo.spring.io/plugins-release/            | https://maven.aliyun.com/repository/spring-plugin 或 https://maven.aliyun.com/nexus/content/repositories/spring-plugin |
| grails-core    | https://repo.grails.org/grails/core              | https://maven.aliyun.com/repository/grails-core 或 https://maven.aliyun.com/nexus/content/repositories/grails-core |
| apache snapshots | https://repository.apache.org/snapshots/        | https://maven.aliyun.com/repository/apache-snapshots 或 https://maven.aliyun.com/nexus/content/repositories/apache-snapshots |
