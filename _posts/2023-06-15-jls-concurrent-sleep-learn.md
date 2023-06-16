---
layout: post
tag: "Java"
date: 2023-06-15
title: "JLS 并发 Thread.sleep 刷新共享内存探究"
desp: "Lorem ipsum dolor sit amet, consectetur adipiscing elit. Nulla massa lorem, tempor vitae libero vitae, fringilla interdum mauris. Integer volutpat aliquam orci, non eleifend diam tempor sed. Aenean eget nulla non diam maximus porttitor eu in eros. Etiam auctor odio tempus molestie hendrerit. Proin tristique tincidunt nibh, vitae viverra sem scelerisque eget. Nam porta orci ac quam blandit, nec pellentesque erat volutpat. Phasellus turpis erat, dictum non tristique vitae, vehicula eu nisl. Duis ut quam nulla. Proin in sapien ex. Nullam porta, orci in elementum cursus, quam velit viverra elit, eu tristique turpis ex et tellus. Duis molestie arcu venenatis, cursus nisi in, malesuada metus."
---

最近在学习Java Language Specification中的[Chapter 17. Threads and Locks](https://docs.oracle.com/javase/specs/jls/se8/html/jls-17.html)。
看到`17.3. Sleep and Yield`及附带的例子时，照着编了一段代码运行却发现两处不理解的点，在此记录一下。

## JLS内容
> ### 17.3. Sleep and Yield
> Thread.sleep causes the currently executing thread to sleep (temporarily cease execution) for the specified duration, subject to the precision and accuracy of system timers and schedulers. The thread does not lose ownership of any monitors, and resumption of execution will depend on scheduling and the availability of processors on which to execute the thread.
> 
> It is important to note that neither Thread.sleep nor Thread.yield have any synchronization semantics. In particular, the compiler does not have to flush writes cached in registers out to shared memory before a call to Thread.sleep or Thread.yield, nor does the compiler have to reload values cached in registers after a call to Thread.sleep or Thread.yield.
>
> > For example, in the following (broken) code fragment, assume that this.done is a non-volatile boolean field:
> > ```java
> >	while (!this.done)
> >     Thread.sleep(1000);
> > ```
> > The compiler is free to read the field this.done just once, and reuse the cached value in each execution of the loop. This would mean that the loop would never terminate, even if another thread changed the value of this.done.

## 疑点
对例子中代码完善如下：
```java
import java.util.concurrent.TimeUnit;

public class ThreadTest {
   boolean stopRequested = false;
   
   public static void main(String[] args) {
      ThreadTest t = new ThreadTest();
      try {
         t.run();
      } catch (InterruptedException e) {
         e.printStackTrace();
      }
   }

   public void run() throws InterruptedException {
      new Thread(
         () -> {
            long start = System.currentTimeMillis();
            int i = 0;
            while(!stopRequested) {
               // if (i >= Integer.MAX_VALUE) break;
               i++;
               try {
                  // System.out.println(String.format("[CHILD] PENDING -- %s -- %d -- %d", stopRequested, i, System.currentTimeMillis()));
                  // Thread.sleep(0);
                  // Thread.yield();
                  // String.format("%s", i);
                  // boolean b = i == 0;
               } catch (Exception e) {
                   
               }
            }
            System.out.println(String.format("%d times completed in %sms", i, System.currentTimeMillis() - start));
         }
      ).start();
      // Thread.sleep(10);
      // TimeUnit.MILLISECONDS.sleep(1);
      stopRequested = true;      
   }
}
```

### 疑点一
上述代码运行后，并未出现例子中所描述的子线程无限循环的情况，而是在0ms后顺利运行完成并退出。
然而，在主线程将标志位`stopRequested`置为`false`之前将主线程sleep一段时间，则会出现子线程无限循环的结果。
```java
new Thread(
// 子线程如上
).start();
Thread.sleep(10);
stopRequested=true;
```

### 疑点二
在以上基础上，按照jls例子，为子线程循环代码块方法内添加`Thread.sleep(1)`后再次运行测试，结果跳出循环并执行完毕。
```java
new Thread(
    ......
    while(!stopRequested) {
        i++;
        try {
            Thread.sleep(1);
        } catch (Excrption e) {
            
        }
    }
    ......
).start();
```
## 探究
针对疑点一，本以为是由于在启动子线程后立刻将标志位置为`false`，此时子线程还未完成启动或未执行至while代码块，故而能获取到值为`false`的标志位，从而结束循环并退出。

但当我们在保持主线程sleep 10毫秒不变的情况下，在子线程循环块中添加`Double.parseDouble("1");`后，执行结果则会结束循环并退出。
```java
new Thread(
    ......
    while (!stopRequested){
        i++;
        try {
            Double.parseDouble("1");
        } catch (Exception e) {
            
        }
    }
    System.out.println(String.format("%d times completed in %sms", i, System.currentTimeMillis() - start));
)
```
`执行结果`
```
377222 times completed in 13ms
```
由`执行结果`可知，在子线程发现标志位变化并退出前，已经循环377222次。那么，这次退出并非是由于更改标志位时子线程还未执行至while代码块，与原本的猜想相悖。而`Double.parse()`方法又并非同步方法，那么为何添加此方法后子线程能够读到主线程对标志位的更新？  
我们继续进行实验，将子线程循环块中添加的方法作如下替换：
```java
new Thread(
    ......
    while (!stopRequested){
        i++;
        try {
            // Double.parseDouble("1");
            boolean b = i == 0;
        } catch (Exception e) {
            
        }
    }
    System.out.println(String.format("%d times completed in %sms", i, System.currentTimeMillis() - start));
)
```
替换后再次执行，发现子线程进入无限循环状态，又无法读取到主线程对标志位的变化。  
对比两次实验中子线程循环块的方法`Double.parse("1");`和`boolean b = i == 0;`，后者消耗的运行时间应该更短。那么我们大胆猜测，是不是将主线程sleep时间缩短的话，又能够使子线程读取到主线程对标志位的变化？
```java
new Thread(
    ......
    while (!stopRequested){
        i++;
        try {
            // Double.parseDouble("1");
            boolean b = i == 0;
        } catch (Exception e) {
            
        }
    }
    System.out.println(String.format("%d times completed in %sms", i, System.currentTimeMillis() - start));
).start();
Thread.sleep(1);
stopRequested = true;
```
`执行结果`
```
255697 times completed in 1ms
```
果不其然，成功监测到标志位并结束循环。在此针对疑点一，得出子线程能否读取到主线程对标志位的变更受到两个因素的影响：`子线程内部循环方法消耗的运行时间`和`子线程启动至主线程更改标志位的时间间隔`。
那么具体的说，这两个因素是如何影响内存可见性的？  
结合以下两个链接：  
https://stackoverflow.com/questions/72632817/thread-sleep-behaviour-with-non-volatile-boolean-variable
https://www.zhihu.com/question/39458585/answer/81521474

个人猜测，由于JIT编译器会在程序运行时对执行频率高的热点代码进行编译，将其转化为机器码以提高执行效率。并且我们看到第二个链接中提到的hoisting优化编译的方法会将标志位提升，从而导致即使主线程将标志位变更并flush回共享内存后，子线程的循环块依然无法读取到这种变化。

而对比几次实验，在sleep 10ms的情况下，将`Double.parse("1");`切换为`boolean b = i == 0;`后，消耗时间大减。10ms的时间间隔内，第二个方法高频率运行，从而触发JIT对热点代码编译。

而将sleep时间修改为1ms后，编译器还未来得及编译子线程就已经读取到标志位的变化并退出循环。

到这里，还有一个***疑问***。即便是编译器没有优化子线程方法，那么子线程是如何读取到标志位的变化，JLS的例子中不是说`Thread.sleep`不是同步方法、不会flush缓存回共享内存吗？

看过第一个参考链接后，发现是我不小心疏忽理解错误，JLS中原文写的是：
> the compiler ***does not have to*** flush writes cached in registers out to shared memory

> The compiler ***is free to*** read the field this.done just once

看到这里，第二个疑点应该也就有了答案。JLS仅仅规定了编译器可以只读取标志位一次并缓存使用，而非强制如此。在实际的编译器实现中，很可能在调用`Thread.sleep`方法时，会将缓存中的值flush回内存。

此处仅为个人猜测，还需进一步研究及查阅资料为证。