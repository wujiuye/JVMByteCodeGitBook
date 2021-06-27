# 一款轻量级的分布式调用链路追踪Java探针vine

![Java艺术](../qrcode/javaskill_qrcode_01.png)

vine[^1]是一款轻量级的分布式调用链路追踪-Java探针。对于同进程内，vine在调用链路入口处创建用于收集日记的单向链表， 调用链路上每经过一个方法都会向链表尾部追加日记，最终在调用链路出口处打印整个调用链路的日记；不同进程间，通过传递事件ID将整条链路串连起来。

* vine目前已经适配OpenFeign、WebMvc框架；
* vine通过类加载器实现与Spring Boot应用环境隔离，各自依赖的jar包不受影响；
* vine还支持配置采样率，降低对应用性能的损耗；
* vine使用logback输出日记，对于同进程内，一次调用链路只打印一次日记。

vine只是探针，不负责日记的收集。 如果您项目中也是将日记收集到阿里云日记服务（或者其它日记服务），那么可以使用vine将调用链路日记输出到阿里云日记服务（或者其它日记服务）。

## 使用说明

启动示例：
```shell
java -javaagent:{绝对路径}/vine-agent.jar={绝对路径}/vine-core-jar-with-dependencies.jar,\
{绝对路径}/vine-spy.jar=agent.package={应用包名，如：com.wujiuye.app} -jar {应用的jar包}
```

Docker镜像构建配置文件示例：
```shell
FROM java:8-jdk-alpine

WORKDIR /usr/app

COPY ./test-app.jar ./

COPY ./agent/vine-agent.jar ./
COPY ./agent/vine-core-jar-with-dependencies.jar ./
COPY ./agent/vine-spy.jar ./

ENTRYPOINT ["java", "-server", "-XX:+UseG1GC",\
    "-javaagent:/usr/app/vine-agent.jar=/usr/app/vine-core-jar-with-dependencies.jar,/usr/app/vine-spy.jar=agent.package=com.test",\
    "-jar", "test-app.jar"]
```

## 各模块说明
* vine-spy: 定义Spy和ContextSpy类，这两个类由启动类加载器加载，定义了方法入口和出口静态方法，由应用代码中的埋点代码调用；
* vine-core: 探针的核心功能实现，实现class字节码修改、接收Spy、ContextSpy上报的事件、构造调用链路、日记输出，适配主流框架（webmvc、openfeign）；
* vine-agent: 定义premain、agentmain方法，负责使用不同类加载器加载vine-spy.jar与vine-core.jar；

## vine的核心设计

### 实现环境隔离
使用不同类加载器加载不同模块，实现环境隔离：
* vine-spy.jar：由启动类加载器加载；
* vine-core.jar：自定义的类加载器加载，与应用隔离；
* vine-agent.jar：由系统类加载器加载；



---

[^1]: https://github.com/wujiuye/vine

<font color= #666666>发布于：2021 年 06 月 27 日</font><br><font color= #666666>作者: 吴就业</font><br><font color= #666666>链接: https://github.com/wujiuye/vine</font><br><font color= #666666>来源: Github开源项目：vine，未经作者许可，禁止转载!</font><br>

![Java艺术](../qrcode/javaskill_qrcode_02.png)

