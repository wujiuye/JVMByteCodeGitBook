# 实现类Spring框架@Async注解功能的asyncframework

![Java艺术](../qrcode/javaskill_qrcode_01.png)

asyncframework是笔者于2019年实现的一个类Spring框架@Async注解功能的异步框架，只需要在接口上添加一个`@AsyncFunction`注解就可让这个方法异步执行，并已发布在笔者的[Github](https://github.com/wujiuye/asyncframework)上。

asyncframework框架的`@AsyncFunction`注解不仅支持用在无返回值的方法上，与Spring框架一样，它同样支持`@AsyncFunction`注解用在有返回值的方法上。

但与Spring框架实现不同的是，asyncframework框架是完全基于动态字节码技术实现的，支持在非`spring`项目中使用，这也是我当初写它的原因。

如果你也对字节码感兴趣，我非常推荐你阅读这个框架的源码，浓缩的都是精华，十几个类包含了设计模式的使用、字节码、以及框架的设计思想，对你理解`spring`的`@Async`注解实现原理也有帮助。

首先允许笔者向您介绍如何使用asyncframework，再介绍asyncframework的实现原理。

## 如何使用asyncframework

第一步：在Java项目中添加依赖

```
<dependency>
  <groupId>com.github.wujiuye</groupId>
  <artifactId>asyncframework</artifactId>
  <version>1.2.0-RELEASE</version>
</dependency>
```

第二步：定义接口以及编写接口的实现类

```java
/**
 * @author wujiuye
 * @version 1.0 on 2019/11/24
 */
public interface AsyncMessageSubscribe {
    /**
     * 异步无返回值
     *
     * @param queue
     */
    @AsyncFunction
    void pullMessage(String queue);
    /**
     * 异步带返回值
     *
     * @param s1
     * @param s2
     * @return
     */
    @AsyncFunction
    AsyncResult<String> doAction(String s1, String s2);
}
```

编写实现类：

```java
public class AsyncMessageSubscribe implements AsyncMessageSubscribe {
    @Override
    public void pullMessage(String queue) {
        System.out.println(queue + "， current thread name:" + Thread.currentThread().getName());
    }
    @Override
    public AsyncResult<String> doAction(String s1, String s2) {
        return new AsyncResult<>("hello wujiuye! current thread name:" + Thread.currentThread().getName());
    }
}
```

第三步：配置全局线程池，以及使用`AsyncProxyFactory`创建代理对象

在调用`AsyncProxyFactory`的`getInterfaceImplSupporAsync`方法创建代理类实例时，需要指定异步执行使用哪个线程池，以及接口的实现类。

```java
public class AsmProxyTest {
   // 配置全局线程池
   static ExecutorService executorService = Executors.newFixedThreadPool(2);
   @Test
   public void testAutoProxyAsync() throws Exception {
         AsyncMessageSubscribe proxy = AsmProxyFactory.getInterfaceImplSupporAsync(
                        AsyncMessageSubscribe.class, impl, executorService);
         // 异步不带返回值
         proxy.pullMessage("wujiuye");
         // 异步带返回值
         AsyncResult<String> asyncResult = proxy.doAction("sssss", "ddd");
         System.out.println(asyncResult.get());
   }
}
```

你可能会问，这还要创建代理类去调用，我直接`new`一个`Runnable`放到线程池执行不是更方便？

确实如此，但如果通过包扫描自动创建代理对象那就不一样了，`spring`就是通过`BeanPostProcess`实现的。而且，当我们需要把异步改为同步时，只需要去掉注解，而当想同步改异步时，也只需要添加注解，不需要改代码。

## 异步无返回值的实现原理

我们以实现消息异步订阅为例，介绍在不使用任何框架的情况下，如何通过静态代理实现将订阅消息方法由同步切换到异步，而这正是asyncframework的实现原理，asyncframework只是将静态代理改为动态代理。

定义消息订阅接口：

```java
public interface MessageSubscribeTemplate {
      <T> void subscribeMessage(MessageQueueEnum messageQueue, 
                                OnReceiveMessageCallback<T> onReceiveMessageCallback,
                                Class<T> tagClass);
}
```

消息订阅接口实现类：

```java
public class AwsSqsMessageConsumer implements MessageSubscribeTemplate {    
    @Override
    public <T> void subscribeMessage(MessageQueueEnum messageQueue, 
                                   OnReceiveMessageCallback<T> onReceiveMessageCallback,
                                   Class<T> tagClass){
         // 编写实现逻辑 
    }
}
```

提示：为什么消息订阅抽象为接口？因为当时我们经常会切换MQ框架，一开始使用RocketMQ，后面由于成本问题又切换到了AWS的SQS服务。

下面就可以通过静态代理实现消息订阅的同步切异步，代码如下。

```java
public class MessageSubscribeTemplateProxy implements MessageSubscribeTemplate {
    private ExecutorService executorService; 
    private MessageSubscribeTemplate target;
  
    public MessageSubscribeTemplateProxy(ExecutorService executorService,
                                         MessageSubscribeTemplate target) {
        this.target = target;
      	this.executorService = executorService;
    }
  
    @Override
    public void subscribeMessage(MessageQueueEnum var1, OnReceiveMessageCallback var2, Class var3) {
        // 实现异步调用逻辑，就是放到线程池中去执行
         executorService.execute(()->this.target.subscribeMessage(var1, var2, var3)); 
    }
}
```

asyncframework框架就是实现动态编写MessageSubscribeTemplateProxy代理类，以此省去同步切异步或异步切同步时修改MessageSubscribeTemplateProxy代理类的麻烦。

有了asyncframework，我们只需要编写消息订阅模版的实现类即可，同步还是异步我们不必关系，当想让订阅方法异步执行就在方法上添加@AsyncSubscribe注解。并且支持接口多个方法，对某些方法添加注解，就只会是这些方法实现异步执行。

## 异步带返回值的实现原理

笔者在实现支持带返回值的方法异步执行这个功能时，遇到了两个大难题：

* 难点一：带返回值的方法如何去实现异步？

* 难点二：如何编写字节码实现泛型接口的代理类？

在`spring`项目中，如果想在带返回值的方法上添加`@Async`注解，就需要方法返回值类型为`AsyncResult<T>`，笔者也去看了一下`spring`的源码，发现`AsyncResult`是一个`Future`。

思路有是有了，但仅仅只是依靠`Future`还是实现不了的。

我们知道，`ExecutorService`的`submit`方法支持提交一个`Callable`带返回值的任务，并且`submit`方法返回一个`Future`，调用这个`Future`的`get`方法当前线程会阻塞，直到任务执行结束。

所以如果我们在代理类方法中调用`Future`的get方法等待结果，再将结果包装成`AsyncResult`返回，这就不是异步执行了，而是同步执行了。

所以我们要解决的问题就是：代理类必须要在将异步方法提交到线程池后，就要立即返回一个`AsyncResult`，并且要确保当外部调用这个`AsyncResult`的`get`方法时，获取到的结果就是最终方法执行后返回的结果。

笔者想到的方法是：在代理类将异步方法提交到线程池后，立即返回一个`AsyncResult`代理对象，这个`AsyncResult`代理对象代理的是`Future`的get方法，当外部调用这个`AsyncResult`代理对象的get方法时，再去调用`Future`的get方法。

先实现`AsyncResult`，这是一个非阻塞的`Future`，因为不需要阻塞。

```java
public class AsyncResult<T> implements Future<T> {
    private T result;
    public AsyncResult(T result) {
        this.result = result;
    }    
  
    @Override
    public T get() throws InterruptedException, ExecutionException {
        return result;
    }

    @Override
    public T get(long timeout, TimeUnit unit) throws InterruptedException, ExecutionException, TimeoutException {
        return get();
    }

    /**
     * 由字节码调用
     *
     * @param future 提交到线程池执行返回的future
     * @param <T>
     * @return
     */
    public static <T> AsyncResult<T> newAsyncResultProxy(final Future<AsyncResult<T>> future) {
        return new AsyncResult<T>(null) {
            @Override
            public T get() throws InterruptedException, ExecutionException {
                AsyncResult<T> asyncResult = future.get();
                return asyncResult.get();
            }
        };
    }

}
```

`newAsyncResultProxy`方法才是整个异步实现的最关键一步，该方法是给字节码生成的代理对象调用的，代理方法实际返回结果是newAsyncResultProxy方法返回的AsyncResult。当外部调用这个`AsyncResult`的`get`方法时，实际上是去调用`ExecutorService`的`submit`方法返回的那个`Future`的`get`方法。对使用者屏蔽了这个阻塞获取结果的实现过程。

还是以消息订阅为例：

```java
// 接口
public interface AsyncMessageSubscribe {
    @AsyncFunction
    AsyncResult<String> doAction(String s1, String s2);
}

// 接口实现类
private AsyncMessageSubscribe impl = new AsyncMessageSubscribe() {
    @Override
    public AsyncResult<String> doAction(String s1, String s2) {
        return new AsyncResult<>("current thread name:" + Thread.currentThread().getName());
    }
};
```

asyncframework框架使用动态字节码技术生成的将AsyncMessageSubscribe#doAction方法提交到线程池执行的Callable代码如下。  

```java
public static class AsyncMessageSubscribe_doActionCallable implements Callable<AsyncResult<String>> {
    private AsyncMessageSubscribe target;
    private String param1;
    private String param2;

    public AsyncMessageSubscribe_doActionCallable(AsyncMessageSubscribe var1, String var2, String var3) {
        this.target = var1;
        this.param1 = var2;
        this.param2 = var3;
    }

    public AsyncResult<String> call() throws Exception {
        return this.target.doAction(this.param1, this.param2);
    }
}
```

asyncframework框架使用动态字节码技术生成的AsyncMessageSubscribe的动态代理类如下。

```java
public class AsyncMessageSubscribeProxy implements AsyncMessageSubscribe {
    private ExecutorService executorService; 
    private AsyncMessageSubscribe target;
  
    public MessageSubscribeTemplateProxy(ExecutorService executorService,
                                         MessageSubscribeTemplate target) {
        this.executorService = executorService;
        this.target = target;
    }
    
    public AsyncResult<String> doAction(String s1, String s2) {
         AsyncMessageSubscribe_doActionCallable callable = new AsyncMessageSubscribe_doActionCallable(target, "wujiuye", "hello");
         Future result = executorService.submit(callable);
         AsyncResult<String> asyncResult = AsyncResult.newAsyncResultProxy(result);
         return asyncResult;
    }
}
```

## 在实现asyncframework中踩的动态字节码实现泛型接口的坑

asyncframework框架动态实现代理类异步方法的代码源码在`FutureFunctionHandler`这个类中。

```java
public class FutureFunctionHandler implements AsyncFunctionHandler{
        /**
             * asyncMethod有返回值，且返回值类型为Future的处理
             *
             * @param classWriter          类改写器
             * @param interfaceClass       接口
             * @param asyncMethod          异步方法
             * @param proxyObjClass        接口的实现类
             * @param executorServiceClass 线程池的类型
             */
        @Override
        public void doOverrideAsyncFunc(ClassWriter classWriter, Class<?> interfaceClass, Method asyncMethod, Class<?> proxyObjClass, Class<? extends ExecutorService> executorServiceClass) {
              ...........
            // invoke submit callable
            methodVisitor.visitVarInsn(ALOAD, 0);
            methodVisitor.visitFieldInsn(GETFIELD, ByteCodeUtils.getProxyClassName(proxyObjClass), "executorService", Type.getDescriptor(executorServiceClass));
            methodVisitor.visitVarInsn(ALOAD, index);
            if (!executorServiceClass.isInterface()) {
                methodVisitor.visitMethodInsn(INVOKEVIRTUAL, executorServiceClass.getName().replace(".", "/"),
                        "submit", ByteCodeUtils.getFuncDesc(Future.class, Callable.class), false);
            } else {
                methodVisitor.visitMethodInsn(INVOKEINTERFACE, executorServiceClass.getName().replace(".", "/"),
                        "submit", ByteCodeUtils.getFuncDesc(Future.class, Callable.class), true);
            }
            // 将返回值存到操作数栈
            methodVisitor.visitVarInsn(ASTORE, ++index);

            // 再来一层代理，对外部屏蔽线程阻塞等待
            methodVisitor.visitVarInsn(ALOAD, index);
            methodVisitor.visitMethodInsn(INVOKESTATIC, AsyncResult.class.getName().replace(".", "/"),
                    "newAsyncResultProxy", ByteCodeUtils.getFuncDesc(AsyncResult.class, Future.class),
                    false);

            methodVisitor.visitInsn(ARETURN);
            ..............
        }
}
```

线程池在调用`AsyncMessageSubscribe_doActionCallable`这个`Callable`的时候，它查找的call方法的方法描述符是()Ljava.lang.Object;。因为`Callable`是个泛型接口。

如果把实现类的签名和实现的`call`方法的签名改为下面这样反而不行。

```
类的签名：Ljava/lang/Object;Ljava/util/concurrent/Callable<Lcom/wujiuye/asyncframework/handler/async/AsyncResult<Ljava/lang/String;>;>;"

call方法的签名： 
()Lcom/wujiuye/asyncframework/handler/async/AsyncResult<Ljava/lang/String;>;
```

因为泛型`<T>`编译后的描述符是`Ljava.lang.Object;`。

如`AsyncResult`泛型类。(选部分)

```java
public class AsyncResult<T> implements Future<T> {

    private T result;
    
    @Override
    public T get() throws InterruptedException, ExecutionException {
        return result;
    }

}
```

`AsyncResult`泛型类编译后的字节码信息。(选部分)

```
public class com.wujiuye.asyncframework.handler.async.AsyncResult<T> implements java.util.concurrent.Future<T> {
  private T result;
    descriptor: Ljava/lang/Object;

  public T get() throws java.lang.InterruptedException, java.util.concurrent.ExecutionException;
    descriptor: ()Ljava/lang/Object;
    Code:
       0: aload_0
       1: getfield      #2   // Field result:Ljava/lang/Object;
       4: areturn
```

类型`T`的`descriptor（类型描述符）`为`Ljava/lang/Object;`，以及`get`方法中，`getfield`指令指定的类型描述符也是`Ljava/lang/Object;`。

`Callable`接口也是泛型接口，编译后`call`方法的方法描述符便是`()Ljava.lang.Object;`。

```
@FunctionalInterface
public interface Callable<V> {
    V call() throws Exception;
}
```

所以，如果通过字节码实现`Callable`接口，`call`方法不要设置方法签名，设置方法签名意味着也要改变方法的描述符，一改变就会导致线程池中调用这个`Callable`的`call`方法抛出抽象方法调用错误，原因是根据`Callable`接口的`call`方法的描述符在这个`Callable`对象的类(Class)中找不到对应的`call`方法。

---

A：既然`spring`都已经提供这样的功能，你为什么还要实现一个这样的框架呢？

Q：因为我之前写组件的时候有需要用到，但又不想为了使用这个功能就把spring依赖到项目中，会比较臃肿。其次，也是因为喜欢折腾，想要把自己的想法实现。

`asyncframework`可以取代`spring`的`@Async`使用，只要封装一个`starter`包，依靠`spring`提供的`BeanPostProcess`实现无缝整合。但`spring`都已经提供了，我就不想去造轮子了，`asyncframework`我推荐是在非`spring`项目中使用。 

<font color= #666666>发布于：2021 年 06 月 27 日</font><br><font color= #666666>作者: 吴就业</font><br><font color= #666666>链接:https://mp.weixin.qq.com/s/PHsUNpvo6UQT5M1o2ZhWjQ</font><br><font color= #666666>来源: 微信公众号“Java艺术”（《笔者开源的asyncframework框架是如何实现类Spring框架@Async注解功能的》），未经作者许可，禁止转载!</font><br>

![Java艺术](../qrcode/javaskill_qrcode_02.png)

