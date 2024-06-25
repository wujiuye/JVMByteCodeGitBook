# JavaAgent入门级项目Beemite

Beemite[^1]是笔者刚开始学习JVM字节码时的一个实战Demo级项目，该项目采用`javaagent`+`asm`动态字节码插桩实现业务代码调用链监控，适合新手入门学习。

Beemite分两个版本：

* 旧版本：`master`分支，写于`2018`年年底，实现在类加载之前改写字节码，插入埋点；
* 新版本：`agentmain`分支，使用`Attach API`，在`Java`程序运行期间改写字节码，重新加载类。

主要实现的功能有：
* 业务代码调用链插桩，在方法执行之前拦截获取类名、方法名，方法调用的参数，在方法执行异常时，获取到异常信息；
* 为统计方法执行时间插入埋点，在方法执行之前和返回之前获取系统时间。

### 使用方法

#### 旧版本

1、将`bee-mite`模块执行`maven package`进行打包，获取`jar`包的绝对路径。

2、以项目中提供的`demo`为例，在`IDEA`中，在`bee-mite-webdemo`项目下，点击锤子->`Edit Config....`-> `VM options` ->输入下面内容

``` 
-javaagent:/MyProjects/asm-aop/insert-pile/target/bee-mite-1.2.0-jar-with-dependencies.jar=com.wujiuye 
```
等号后面是参数，意思是改写哪个包下的类。

如果报如下异常：
```
java.lang.VerifyError: Expecting a stackmap frame at branch target 18
```
jdk1.8可以添加参数：`-noverify`解决
``` 
-noverify -javaagent:/MyProjects/asm-aop/insert-pile/target/insert-pile-1.0-SNAPSHOT.jar=com.wujiuye
```

#### 新版本

* 1、先将`bee-mite`模块执行`maven package`进行打包；
* 2、将`bee-mite-boot`模块打包，或者直接在`idea`中启动。

`bee-mite-boot`这个模块只有一个类，就是`BeemiteBoot`，它的作用就是查询系统当前有哪些`java`进程，获取到进程的`id`，然后根据进程`id`，将上一步编译的`bee-mite`的`jar`包加载到目标进程。

需要告诉`BeemiteBoot`，`bee-mite`的`jar`包放在哪里，就是`bee-mite`的`jar`包的绝对路径。目前我是写死在代码中的，可以通过修改代码替换。

```text
try {
    vm.loadAgent("/Users/wjy/MyProjects/beemite/bee-mite/target/bee-mite-1.2.0-jar-with-dependencies.jar", pageckName + "`" + plus);
} finally {
    vm.detach();
}
```

当`BeemiteBoot`启动起来之后，就可以根据提示一步步操作了。

```
找到如下Java进程，请选择：
[1] 2352	
[2] 3818	com.wujiuye.ipweb.DemoAppcliction
[3] 3595	org.jetbrains.idea.maven.server.RemoteMavenServer36
[4] 3821	org.jetbrains.jps.cmdline.Launcher
```

选择对应进程之后，会提示输入目标进程的应用包名，目的是过滤只改写指定包名下的类。
```text
应用的包名：
com.wujiuye
```

输入包名后会提示选择插件，目前就两个插件，一个是监控调用链的，另一个是打印方法执行耗时的，多可选。
```text
选择插件：
1	打印调用链路插件
2	打印方法执行耗时插件
结束请输入：0
```
完成后，就已经成功将`bee-mite`的`jar`包加载到目标进程了。

#### 结果展示

`bee-mite-boot`是一个`web`项目，在浏览器中输入`http://127.0.0.1:8080/user/wujiuye/25`，即可看到插桩后输出的日记

```text
[接收到事件，打印日记]savaFuncStartRuntimeLog[null,com/wujiuye/ipweb/handler/UserHandler,queryUser,1585486646788]
[接收到事件，打印日记]savaBusinessFuncCallLog[null,com/wujiuye/ipweb/handler/UserHandler,queryUser]
[接收到事件，打印日记]savaFuncStartRuntimeLog[null,com/wujiuye/ipweb/service/impl/UserServiceImpl,queryUser,1585486646790]
[接收到事件，打印日记]savaBusinessFuncCallLog[null,com/wujiuye/ipweb/service/impl/UserServiceImpl,queryUser]
[接收到事件，打印日记]savaFuncEndRuntimeLog[null,com/wujiuye/ipweb/service/impl/UserServiceImpl,queryUser,1585486646791]
[接收到事件，打印日记]savaFuncEndRuntimeLog[null,com/wujiuye/ipweb/handler/UserHandler,queryUser,1585486646791]
```

### 源码导读

agentmain分支是最新版本的分支，以下是agentmain分支各package说明：

`core`包：实现字节码插桩，在方法执行之前拦截获取类名、方法名，方法调用的参数，在方法执行异常时，获取到异常信息，以及在方法执行之前和`return`之前获取当前系统时间，可用于统计方法执行耗时。

`tracsformer`包：代码插桩过滤器，使用责任连模式，对字节码进行多次插桩，每个插桩器只负责自己想要实现的逻辑。

`event`包：事件的封装，埋点代码抛出的事件放入事件队列，异步分派事件给监听器进行处理。

`logs`包：提供事件监听器接口，具体实现可扩展，我这里提供了两个默认的实现类，默认的实现类只是将日记打印，在控制台打印日记信息。

因为字节码是插入到业务代码中的，当执行业务代码的时候会执行埋点代码，如果处理程序也在业务代码中进行，那么这将是个耗时的操作，影响性能，拖慢一次请求的响应速度，所以当埋点代码执行的时候，Beemite是直接抛出一个事件，让线程池异步消费事件，分派事件给相应的监听器处理，这样就可以执行耗时操作，比如将日记输出到硬盘，再存储到`ES`，便于后期进行项目代码异常排查。

### Beemite是怎么改写class的

Beemite会将改写后的`class`字节码输出到文件，并会在控制台打印输出的文件路径，可以看下输出后的`class`。

以`UserServiceImpl`为例，这个类是插桩的目标，来看下对比改写后的代码，到底`bee-mite`改写字节码都做了什么。

源代码

```java
public class UserServiceImpl {

    public Map<String, Object> queryMap(String username,Integer age) {
        Map<String, Object> map = new HashMap<>();
        map.put("username", username);
        map.put("age", age);
        return map;
    }

}
```

`Beemite`插桩后

```java
@Service
public class UserServiceImpl implements UserService {
    public UserServiceImpl() {
    }

    public Map<String, Object> queryUser(String var1, Integer var2) {
        long var3 = System.currentTimeMillis();
        FuncRuntimeEvent.sendFuncStartRuntimeEvent(SessionIdContext.getContext().getSessionId(), "com/wujiuye/ipweb/service/impl/UserServiceImpl", "queryUser", var3);
        try {
            Object[] var8 = new Object[]{var1, var2};
            BusinessCallLinkEvent.sendBusinessFuncCallEvent(SessionIdContext.getContext().getSessionId(), "com/wujiuye/ipweb/service/impl/UserServiceImpl", "queryUser", var8);
            HashMap var9 = new HashMap();
            var9.put("username", var1);
            var9.put("age", var2);
            long var5 = System.currentTimeMillis();
            FuncRuntimeEvent.sendFuncEndRuntimeEvent(SessionIdContext.getContext().getSessionId(), "com/wujiuye/ipweb/service/impl/UserServiceImpl", "queryUser", var5);
            return var9;
        } catch (Exception var7) {
            BusinessCallLinkEvent.sendBusinessFuncCallThrowableEvent(SessionIdContext.getContext().getSessionId(), "com/wujiuye/ipweb/service/impl/UserServiceImpl", "queryUser", var7);
            throw var7;
        }
    }
}
```

因为使用了责任链模式，会对代码进行两次插桩，目的就是为了后面容易扩展功能，其实只是`18`年时候自己的水平问题，没有想到通过暴露切点的方式实现更好，少写字节码。我简单看了下`arthas`的部分源码，它的实现就是改写的字节码只插入三个埋点，其它功能不再操作字节码。

相信看了对比你也能知道Beemite都插入了哪些代码，这些代码都是通过`asm`写字节码指令插入的。当然也不是很难，对于新手而言，唯一的难点可能就是`try-catch`代码块的插入。`visitTryCatchBlock`方法的三个`label`的位置，以及`catch`块处理异常算是个难点，我们可以通过在源码类中添加`try-catch`块，然后使用`javap`查看异常处理表，例如。

```
 * Exception table:
 *        from    to  target type
 *            0    27    30   Class java/lang/Exception
```

那么三个`label`对应的就是`from`、`to`、`target`，且当`type`为`any`的时候就是`try-finally`。

---

[^1]: https://github.com/wujiuye/beemite

<font color= #666666>发布于：2021 年 06 月 27 日</font><br><font color= #666666>作者: [吴就业](https://www.wujiuye.com)</font><br><font color= #666666>GitHub链接: https://github.com/wujiuye/beemite</font><br><font color= #666666>博客链接: https://www.wujiuye.com/ebook/JVMByteCodeGitBook/chapter/opensourceprojects_vine.md</font><br><font color= #666666>来源: Github开源项目：beemite，未经作者许可，禁止转载!</font><br>

