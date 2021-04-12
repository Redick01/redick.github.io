# Arthas thred命令

## thread命令参数

&emsp;&emsp; 有时候我们发现应用卡住了，通常是由于某个线程拿住了某个锁， 并且其他线程都在等待这把锁造成的。 为了排查这类问题，arthas提供了thread命令，协助我们快速定位。

&emsp;&emsp; 下面我们用个案例去演示一下这个死锁场景，然后用arthas去定位出这个问题。

## 模拟死锁场景

- **模拟死锁代码**

```
public class ArthasThreadTest {

    private static final Object object1 = new Object();
    private static final Object object2 = new Object();

    public static void main(String[] args) {
        new Thread(() -> {
            synchronized (object1) {
                try {
                    Thread.sleep(5000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                synchronized (object2) {
                    System.out.println("[dead]执行内容111...");
                }
            }
        }).start();

        new Thread(() -> {
            synchronized (object2) {
                synchronized (object1) {
                    System.out.println("[dead]执行内容222...");
                }
            }
        }).start();
    }
}
```

- **Thread命令**

```
[arthas@67089]$ thread
Threads Total: 31, NEW: 0, RUNNABLE: 8, BLOCKED: 2, WAITING: 4, TIMED_WAITING: 2, TERMINATED: 0, Internal threads: 15                                                                                                                   
ID        NAME                                                      GROUP                        PRIORITY           STATE              %CPU                DELTA_TIME         TIME               INTERRUPTED         DAEMON             
-1        C1 CompilerThread3                                        -                            -1                 -                  0.42                0.000              0:0.573            false               true               
29        arthas-command-execute                                    system                       5                  RUNNABLE           0.32                0.000              0:0.050            false               true               
-1        VM Periodic Task Thread                                   -                            -1                 -                  0.05                0.000              0:0.156            false               true               
-1        C2 CompilerThread0                                        -                            -1                 -                  0.02                0.000              0:0.271            false               true               
-1        C2 CompilerThread1                                        -                            -1                 -                  0.01                0.000              0:0.241            false               true               
-1        C2 CompilerThread2                                        -                            -1                 -                  0.0                 0.000              0:0.258            false               true               
13        Thread-0                                                  main                         5                  BLOCKED            0.0                 0.000              0:0.006            false               false              
14        Thread-1                                                  main                         5                  BLOCKED            0.0                 0.000              0:0.007            false               false              
2         Reference Handler                                         system                       10                 WAITING            0.0                 0.000              0:0.002            false               true               
3         Finalizer                                                 system                       8                  WAITING            0.0                 0.000              0:0.004            false               true               
4         Signal Dispatcher                                         system                       9                  RUNNABLE           0.0                 0.000              0:0.000            false               true               
16        Attach Listener                                           system                       9                  RUNNABLE           0.0                 0.000              0:0.035            false               true               
18        arthas-timer                                              system                       9                  WAITING            0.0                 0.000              0:0.000            false               true               
21        arthas-NettyHttpTelnetBootstrap-3-1                       system                       5                  RUNNABLE           0.0                 0.000              0:0.037            false               true               
22        arthas-NettyWebsocketTtyBootstrap-4-1                     system                       5                  RUNNABLE           0.0                 0.000              0:0.002            false               true               
23        arthas-NettyWebsocketTtyBootstrap-4-2                     system                       5                  RUNNABLE           0.0                 0.000              0:0.001            false               true               
24        arthas-shell-server                                       system                       9                  TIMED_WAITING      0.0                 0.000              0:0.001            false               true               
25        arthas-session-manager                                    system                       9                  TIMED_WAITING      0.0                 0.000              0:0.000            false               true               
26        arthas-UserStat                                           system                       9                  WAITING            0.0                 0.000              0:0.000            false               true               
28        arthas-NettyHttpTelnetBootstrap-3-2                       system                       5                  RUNNABLE           0.0                 0.000              0:0.150            false               true               
15        DestroyJavaVM                                             main                         5                  RUNNABLE           0.0                 0.000              0:0.240            false               false              
-1        GC task thread#7 (ParallelGC)                             -                            -1                 -                  0.0                 0.000              0:0.002            false               true               
-1        GC task thread#6 (ParallelGC)                             -                            -1                 -                  0.0                 0.000              0:0.003            false               true               
-1        GC task thread#0 (ParallelGC)                             -                            -1                 -                  0.0                 0.000              0:0.003            false               true               
-1        Service Thread                                            -                            -1                 -                  0.0                 0.000              0:0.000            false               true               
-1        GC task thread#1 (ParallelGC)                             -                            -1                 -                  0.0                 0.000              0:0.003            false               true               
-1        VM Thread                                                 -                            -1                 -                  0.0                 0.000              0:0.121            false               true               
-1        GC task thread#2 (ParallelGC)                             -                            -1                 -                  0.0                 0.000              0:0.003            false               true               
-1        GC task thread#3 (ParallelGC)                             -                            -1                 -                  0.0                 0.000              0:0.003            false               true               
-1        GC task thread#5 (ParallelGC)                             -                            -1                 -                  0.0                 0.000              0:0.003            false               true               
-1        GC task thread#4 (ParallelGC)                             -                            -1                 -                  0.0                 0.000              0:0.002            false               true               
                                            
```

可以看到线程13和14状态是BLOCKED，查看线程ID为13和14的线程情况，可以快速到定位到发生死锁的问题代码行。

```
[arthas@67089]$ thread 13
"Thread-0" Id=13 BLOCKED on java.lang.Object@393742e8 owned by "Thread-1" Id=14
    at com.test.arthastest.ArthasThreadTest.lambda$main$0(ArthasThreadTest.java:21)
    -  blocked on java.lang.Object@393742e8
    at com.test.arthastest.ArthasThreadTest$$Lambda$1/659748578.run(Unknown Source)
    at java.lang.Thread.run(Thread.java:748)

[arthas@67089]$ thread 14
"Thread-1" Id=14 BLOCKED on java.lang.Object@356590d0 owned by "Thread-0" Id=13
    at com.test.arthastest.ArthasThreadTest.lambda$main$1(ArthasThreadTest.java:29)
    -  blocked on java.lang.Object@356590d0
    at com.test.arthastest.ArthasThreadTest$$Lambda$2/41903949.run(Unknown Source)
    at java.lang.Thread.run(Thread.java:748)

```

- **thread -b可以直接查看发生BLOCKED的线程**

```
[arthas@67089]$ thread -b
"Thread-0" Id=13 BLOCKED on java.lang.Object@393742e8 owned by "Thread-1" Id=14
    at com.test.arthastest.ArthasThreadTest.lambda$main$0(ArthasThreadTest.java:21)
    -  blocked on java.lang.Object@393742e8
    -  locked java.lang.Object@356590d0 <---- but blocks 1 other threads!
    at com.test.arthastest.ArthasThreadTest$$Lambda$1/659748578.run(Unknown Source)
    at java.lang.Thread.run(Thread.java:748)
```

注：上面这个命令直接输出了 造成死锁的线程ID，和具体的代码位置，以及当前线程一共阻塞的线程数量：“<—- but blocks 1 other threads!“。太秀了，现在，我们只需要去修改掉这的代码问题即可。

- **thread –state：查看指定状态的线程，如：thread –state BLOCKED**

```
Threads Total: 16, NEW: 0, RUNNABLE: 8, BLOCKED: 2, WAITING: 4, TIMED_WAITING: 2, TERMINATED: 0                                                                                                                                         
ID        NAME                                                      GROUP                        PRIORITY           STATE              %CPU                DELTA_TIME         TIME               INTERRUPTED         DAEMON             
13        Thread-0                                                  main                         5                  BLOCKED            0.0                 0.000              0:0.047            false               false              
14        Thread-1                                                  main                         5                  BLOCKED            0.0                 0.000              0:0.047            false               false              
 
```

- **其他命令**

```
thread –all, 显示所有的线程；
thread id, 显示指定线程的运行堆栈；
thread –state：查看指定状态的线程，如：thread –state BLOCKED；
thread -n 3：展示当前最忙的前N个线程并打印堆栈；
thread -i value：指定cpu使用率统计的采样间隔，单位为毫秒，默认值为200
```