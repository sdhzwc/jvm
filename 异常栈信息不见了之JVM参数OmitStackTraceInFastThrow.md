OmitStackTraceInFastThrow， jdk 1.6开始，默认server模式下开启了这个参数，意为当jvm检测到程序在重复抛一个异常，在执行若干次后会将异常吞掉，这里的若干次在jdk1.7测得是20707。即执行20707次后，stackTrace 长度会为0。

有时这不利于我们排错，通过指定OmitStackTraceInFastThrow，可禁用这功能。


# 问题描述

某天收到生产环境error日志告警（对error.log监控，超过一定大小就会给开发人员发送告警短信）。但是tail查看最新的异常信息只有这些，好忧伤：

```css
... ...
java.lang.NullPointerException
java.lang.NullPointerException
java.lang.NullPointerException
java.lang.NullPointerException
java.lang.NullPointerException
java.lang.NullPointerException
... ...
```

后来有个同事从error.log前面开始看起，能看到完整的异常栈信息，大致如下，这样的信息就能够准确的定位问题：

```bash
... ...
java.lang.NullPointerException
    at com.afei.juc.WithNPESimulate.run(NPEMain.java:21)
    at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
    at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
    at java.lang.Thread.run(Thread.java:745)
java.lang.NullPointerException
    at com.afei.juc.WithNPESimulate.run(NPEMain.java:21)
    at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
    at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
    at java.lang.Thread.run(Thread.java:745)
... ...
```

那么前面那段只有简单的**java.lang.NullPointerException**，没有详细异常栈信息的原因是什么呢？这需要从一个JVM参数说起。

# 原因分析

JVM中有个参数：**OmitStackTraceInFastThrow**，字面意思是**省略异常栈信息从而快速抛出**，那么JVM是如何做到快速抛出的呢？JVM对一些特定的异常类型做了**Fast Throw**优化，如果检测到在代码里某个位置连续多次抛出同一类型异常的话，C2会决定用**Fast Throw**方式来抛出异常，而异常Trace即详细的异常栈信息会被清空。这种异常抛出速度非常快，因为不需要在堆里分配内存，也不需要构造完整的异常栈信息。相关的源码的JVM源码的**graphKit.cpp**文件中，相关源码如下：

```cpp
//------------------------------builtin_throw----------------------------------
void GraphKit::builtin_throw(Deoptimization::DeoptReason reason, Node* arg) {
  bool must_throw = true;

  ... ...
  // 首先判断条件是否满足
  // If this particular condition has not yet happened at this
  // bytecode, then use the uncommon trap mechanism, and allow for
  // a future recompilation if several traps occur here.
  // If the throw is hot（表示在代码某个位置重复抛出异常）, try to use a more complicated inline mechanism
  // which keeps execution inside the compiled code.
  bool treat_throw_as_hot = false;

  if (ProfileTraps) {
    if (too_many_traps(reason)) {
      treat_throw_as_hot = true;
    }
    // (If there is no MDO at all, assume it is early in
    // execution, and that any deopts are part of the
    // startup transient, and don't need to be remembered.)

    // Also, if there is a local exception handler, treat all throws
    // as hot if there has been at least one in this method.
    if (C->trap_count(reason) != 0
        && method()->method_data()->trap_count(reason) != 0
        && has_ex_handler()) {
        treat_throw_as_hot = true;
    }
  }

  // If this throw happens frequently, an uncommon trap might cause
  // a performance pothole.  If there is a local exception handler,
  // and if this particular bytecode appears to be deoptimizing often,
  // let us handle the throw inline, with a preconstructed instance.
  // Note:   If the deopt count has blown up, the uncommon trap
  // runtime is going to flush this nmethod, not matter what.
  // 这里要满足两个条件：1.检测到频繁抛出异常，2. OmitStackTraceInFastThrow为true，或StackTraceInThrowable为false
  if (treat_throw_as_hot
      && (!StackTraceInThrowable || OmitStackTraceInFastThrow)) {
    // If the throw is local, we use a pre-existing instance and
    // punt on the backtrace.  This would lead to a missing backtrace
    // (a repeat of 4292742) if the backtrace object is ever asked
    // for its backtrace.
    // Fixing this remaining case of 4292742 requires some flavor of
    // escape analysis.  Leave that for the future.
    ciInstance* ex_obj = NULL;
    switch (reason) {
    case Deoptimization::Reason_null_check:
      ex_obj = env()->NullPointerException_instance();
      break;
    case Deoptimization::Reason_div0_check:
      ex_obj = env()->ArithmeticException_instance();
      break;
    case Deoptimization::Reason_range_check:
      ex_obj = env()->ArrayIndexOutOfBoundsException_instance();
      break;
    case Deoptimization::Reason_class_check:
      if (java_bc() == Bytecodes::_aastore) {
        ex_obj = env()->ArrayStoreException_instance();
      } else {
        ex_obj = env()->ClassCastException_instance();
      }
      break;
    }
    ... ...
}
```

> 说明：**OmitStackTraceInFastThrow**和**StackTraceInThrowable**都默认为true，所以条件`(!StackTraceInThrowable || OmitStackTraceInFastThrow)`为true，即JVM默认开启了**Fast Throw**优化。如果想关闭这个优化，很简单，配置`-XX:-OmitStackTraceInFastThrow`，**StackTraceInThrowable**保持默认配置`-XX:+OmitStackTraceInFastThrow`即可。

另外，根据这段源码的`switch .. case ..`部分可知，JVM只对几个特定类型异常开启了**Fast Throw**优化，这些异常包括：

- **NullPointerException**
- **ArithmeticException**
- **ArrayIndexOutOfBoundsException**
- **ArrayStoreException**
- **ClassCastException**

# 问题验证

为了验证这个问题，笔者写了下面这段代码：

```java
/**
 * @author afei
 * @version 1.0.0
 * @since 2018年06月08日
 */
class WithNPE extends Thread{

    private static int count = 0;

    @Override
    public void run() {
        try{
            System.out.println(this.getClass().getSimpleName()+"--"+(++count));
            String str = null;
            // 制造空指针NPE
            System.out.println(str.length());
        }catch (Throwable e){
            e.printStackTrace();
        }
    }
}

public class FastThrowMain {
    public static void main(String[] args) throws InterruptedException {
        WithNPE withNPE = new WithNPE();
        ExecutorService executorService = Executors.newFixedThreadPool(20);
        for (int i=0; i<Integer.MAX_VALUE; i++) {
            executorService.execute(withNPE);
            // 稍微sleep一下, 是为了不要让异常抛出太快, 导致控制台输出太快, 把有异常栈信息冲掉, 只留下fast throw方式抛出的异常
            Thread.sleep(2);
        }
    }
}
```

运行部分日志如下：

```bash
WithNPE--6686
... ...
java.lang.NullPointerException
    at com.afei.juc.WithNPE.run(FastThrowMain.java:21)
    at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
    at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
    at java.lang.Thread.run(Thread.java:745)
java.lang.NullPointerException
    at com.afei.juc.WithNPE.run(FastThrowMain.java:21)
    at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
    at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
    at java.lang.Thread.run(Thread.java:745)
java.lang.NullPointerException
    at com.afei.juc.WithNPE.run(FastThrowMain.java:21)
    at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
    at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
    at java.lang.Thread.run(Thread.java:745)
WithNPE--6687
WithNPE--6688
WithNPE--6689
java.lang.NullPointerException
java.lang.NullPointerException
java.lang.NullPointerException
WithNPE--6690
WithNPE--6691
WithNPE--6692
java.lang.NullPointerException
java.lang.NullPointerException
java.lang.NullPointerException
... ...
```

> 从这段日志可知，抛出了几千次带有详细异常栈信息的异常后，只会抛出**java.lang.NullPointerException**这种没有详细异常栈信息只有异常类型的异常信息。这就是**Fast Throw**优化后抛出的异常。如果我们配置了`-XX:-OmitStackTraceInFastThrow`，再次运行，就不会看到**Fast Throw**优化后抛出的异常，全是包含了详细异常栈的异常信息。

配置JVM参数关闭**Fast Throw**后，即使抛出了2w+次异常，依然全是包含了详细异常栈的异常信息，日志如下：

```bash
WithNPE--20719
WithNPE--20720
java.lang.NullPointerException
    at com.afei.juc.WithNPE.run(FastThrowMain.java:21)
    at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
    at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
    at java.lang.Thread.run(Thread.java:745)
java.lang.NullPointerException
    at com.afei.juc.WithNPE.run(FastThrowMain.java:21)
    at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
    at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
    at java.lang.Thread.run(Thread.java:745)
```
