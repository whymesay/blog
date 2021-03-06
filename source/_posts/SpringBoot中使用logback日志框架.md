---
title: SpringBoot中使用logback日志框架
date: 2021-03-06 19:55:54
tags: SpringBoot,Java
categories: 后端
updated:
keywords:
description:
top_img:
comments:
cover:
toc:
toc_number:
copyright:
copyright_author:
copyright_author_href:
copyright_url:
copyright_info:
mathjax:
katex:
aplayer:
highlight_shrink:
aside:
---


## 前言
SpringBoot默认的日志框架就是logback,所以使用的时候直接使用就可以了,不需要添加其他依赖,所以这里记录几个关于配置的小问题.
## 配置
1.首先在resources目录下建立文件logback-spring.xml,添加下面的内容
```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration scan="true" scanPeriod="60 seconds">
    <!-- 控制台输出设置 -->
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
	 <!--这里读取的是application.properties中的logging.pattern.console -->
            <pattern>${CONSOLE_LOG_PATTERN}</pattern>
        </encoder>
    </appender>


    <logger name="com.example.demo" additivity="false">
        <level value="INFO"/>
        <appender-ref ref="STDOUT"/>
    </logger>

<!--这里有个问题就是level会优先取application.properties中的logging.level.root属性,所以如果按照下面的application.properties中的配置,这里的日志级别实际上是DEBUG-->
    <root level="INFO">
        <appender-ref ref="STDOUT"/>
    </root>

</configuration>

```
2.在application.properties中添加以下配置
```properties
logging.config=classpath:logback-spring.xml
logging.pattern.console=[%d{yyyy-MM-dd HH:mm:ss}] -- [%-5p]: [%c] -- %m%n
logging.level.root=DEBUG
```
