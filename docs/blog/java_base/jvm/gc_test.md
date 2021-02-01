# GC日志分析方法

本篇并非GC的调优，而是介绍了当我们拿到GC日志的时候如何去分析日志



[TOC]



## 待测试代码

执行 javac GCLogAnalysis.java编译代码接口
```
import java.util.Random;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.LongAdder;
/*

*/
public class GCLogAnalysis {
    private static Random random = new Random();
    public static void main(String[] args) {
        long startMillis = System.currentTimeMillis();
        long timeoutMillis = TimeUnit.SECONDS.toMillis(1);
        long endMillis = startMillis + timeoutMillis;
        LongAdder counter = new LongAdder();
        System.out.println("...");
        int cacheSize = 2000;
        Object[] cachedGarbage = new Object[cacheSize];
        while (System.currentTimeMillis() < endMillis) {
            Object garbage = generateGarbage(100*1024);
            counter.increment();
            int randomIndex = random.nextInt(2 * cacheSize);
            if (randomIndex < cacheSize) {
                cachedGarbage[randomIndex] = garbage;
            }
        }
        System.out.println(":" + counter.longValue());
    }

    private static Object generateGarbage(int max) {
        int randomSize = random.nextInt(max);
        int type = randomSize % 4;
        Object result = null;
        switch (type) {
            case 0:
                result = new int[randomSize];
                break;
            case 1:
                result = new byte[randomSize];
                break;
            case 2:
                result = new double[randomSize];
                break;
            default:
                StringBuilder builder = new StringBuilder();
                String randomString = "randomString-Anything";
                while (builder.length() < randomSize) {
                    builder.append(randomString);
                    builder.append(max);
                    builder.append(randomSize);
                }
                result = builder.toString();
                break;
        }
        return result;
    }
}
```

## 串行GC（Serial GC）

- **开启Serial GC**

```
java -Xms256m -Xmx512m -XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xloggc:gc.log -XX:+UseSerialGC GCLogAnalysis
```

- **GC日志**

```
Java HotSpot(TM) 64-Bit Server VM (25.151-b12) for bsd-amd64 JRE (1.8.0_151-b12), built on Sep  5 2017 19:37:08 by "java_re" with gcc 4.2.1 (Based on Apple Inc. build 5658) (LLVM build 2336.11.00)
Memory: 4k page, physical 8388608k(345660k free)

/proc/meminfo:

CommandLine flags: -XX:InitialHeapSize=268435456 -XX:MaxHeapSize=536870912 -XX:+PrintGC -XX:+PrintGCDateStamps -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:+UseSerialGC 
2020-10-24T13:23:23.518-0800: 0.154: [GC (Allocation Failure) 2020-10-24T13:23:23.518-0800: 0.154: [DefNew: 69543K->8704K(78656K), 0.0207395 secs] 69543K->24408K(253440K), 0.0208665 secs] [Times: user=0.02 sys=0.01, real=0.02 secs] 
2020-10-24T13:23:23.549-0800: 0.185: [GC (Allocation Failure) 2020-10-24T13:23:23.549-0800: 0.185: [DefNew: 78439K->8703K(78656K), 0.0215639 secs] 94144K->45041K(253440K), 0.0216550 secs] [Times: user=0.01 sys=0.01, real=0.03 secs] 
2020-10-24T13:23:23.585-0800: 0.221: [GC (Allocation Failure) 2020-10-24T13:23:23.585-0800: 0.221: [DefNew: 78182K->8703K(78656K), 0.0219975 secs] 114520K->68036K(253440K), 0.0221630 secs] [Times: user=0.01 sys=0.01, real=0.02 secs] 
2020-10-24T13:23:23.622-0800: 0.258: [GC (Allocation Failure) 2020-10-24T13:23:23.622-0800: 0.258: [DefNew: 78655K->8703K(78656K), 0.0193394 secs] 137988K->90354K(253440K), 0.0194270 secs] [Times: user=0.01 sys=0.01, real=0.02 secs] 
2020-10-24T13:23:23.658-0800: 0.294: [GC (Allocation Failure) 2020-10-24T13:23:23.658-0800: 0.294: [DefNew: 78655K->8702K(78656K), 0.0199063 secs] 160306K->111735K(253440K), 0.0201421 secs] [Times: user=0.01 sys=0.01, real=0.02 secs] 
2020-10-24T13:23:23.693-0800: 0.329: [GC (Allocation Failure) 2020-10-24T13:23:23.693-0800: 0.329: [DefNew: 78654K->8696K(78656K), 0.0257539 secs] 181687K->133736K(253440K), 0.0258526 secs] [Times: user=0.01 sys=0.01, real=0.02 secs] 
2020-10-24T13:23:23.730-0800: 0.366: [GC (Allocation Failure) 2020-10-24T13:23:23.730-0800: 0.366: [DefNew: 78648K->8696K(78656K), 0.0264957 secs] 203688K->158408K(253440K), 0.0267254 secs] [Times: user=0.01 sys=0.01, real=0.02 secs] 
2020-10-24T13:23:23.770-0800: 0.406: [GC (Allocation Failure) 2020-10-24T13:23:23.770-0800: 0.406: [DefNew: 78648K->8703K(78656K), 0.0215485 secs] 228360K->179933K(253440K), 0.0218227 secs] [Times: user=0.01 sys=0.00, real=0.02 secs] 
2020-10-24T13:23:23.800-0800: 0.436: [GC (Allocation Failure) 2020-10-24T13:23:23.800-0800: 0.436: [DefNew: 78433K->8703K(78656K), 0.0163760 secs]2020-10-24T13:23:23.816-0800: 0.452: [Tenured: 189875K->167264K(190052K), 0.0378398 secs] 249663K->167264K(268708K), [Metaspace: 2691K->2691K(1056768K)], 0.0545653 secs] [Times: user=0.05 sys=0.00, real=0.05 secs] 
2020-10-24T13:23:23.889-0800: 0.525: [GC (Allocation Failure) 2020-10-24T13:23:23.889-0800: 0.525: [DefNew: 111074K->13887K(125568K), 0.0190206 secs] 278338K->207283K(404344K), 0.0191827 secs] [Times: user=0.01 sys=0.01, real=0.02 secs] 
2020-10-24T13:23:23.937-0800: 0.573: [GC (Allocation Failure) 2020-10-24T13:23:23.937-0800: 0.573: [DefNew: 125567K->13885K(125568K), 0.0994056 secs] 318963K->245682K(404344K), 0.0996520 secs] [Times: user=0.02 sys=0.03, real=0.10 secs] 
2020-10-24T13:23:24.063-0800: 0.699: [GC (Allocation Failure) 2020-10-24T13:23:24.063-0800: 0.699: [DefNew: 125565K->13886K(125568K), 0.0732244 secs] 357362K->284799K(404344K), 0.0734028 secs] [Times: user=0.02 sys=0.02, real=0.07 secs] 
2020-10-24T13:23:24.160-0800: 0.796: [GC (Allocation Failure) 2020-10-24T13:23:24.160-0800: 0.796: [DefNew: 125566K->13887K(125568K), 0.0631124 secs]2020-10-24T13:23:24.223-0800: 0.859: [Tenured: 306337K->258008K(306488K), 0.0552652 secs] 396479K->258008K(432056K), [Metaspace: 2691K->2691K(1056768K)], 0.1187092 secs] [Times: user=0.08 sys=0.02, real=0.11 secs] 
2020-10-24T13:23:24.295-0800: 0.931: [GC (Allocation Failure) 2020-10-24T13:23:24.295-0800: 0.931: [DefNew: 139776K->17471K(157248K), 0.0282408 secs] 397784K->300076K(506816K), 0.0283933 secs] [Times: user=0.02 sys=0.01, real=0.03 secs] 
2020-10-24T13:23:24.359-0800: 0.995: [GC (Allocation Failure) 2020-10-24T13:23:24.359-0800: 0.995: [DefNew: 157247K->17471K(157248K), 0.0495996 secs] 439852K->344326K(506816K), 0.0497317 secs] [Times: user=0.02 sys=0.02, real=0.05 secs] 
2020-10-24T13:23:24.449-0800: 1.085: [GC (Allocation Failure) 2020-10-24T13:23:24.449-0800: 1.085: [DefNew: 157247K->157247K(157248K), 0.0001119 secs]2020-10-24T13:23:24.449-0800: 1.085: [Tenured: 326855K->300168K(349568K), 0.0520648 secs] 484102K->300168K(506816K), [Metaspace: 2691K->2691K(1056768K)], 0.0524246 secs] [Times: user=0.04 sys=0.00, real=0.06 secs] 
Heap
 def new generation   total 157248K, used 5650K [0x00000007a0000000, 0x00000007aaaa0000, 0x00000007aaaa0000)
  eden space 139776K,   4% used [0x00000007a0000000, 0x00000007a05849b8, 0x00000007a8880000)
  from space 17472K,   0% used [0x00000007a8880000, 0x00000007a8880000, 0x00000007a9990000)
  to   space 17472K,   0% used [0x00000007a9990000, 0x00000007a9990000, 0x00000007aaaa0000)
 tenured generation   total 349568K, used 300168K [0x00000007aaaa0000, 0x00000007c0000000, 0x00000007c0000000)
   the space 349568K,  85% used [0x00000007aaaa0000, 0x00000007bcfc2338, 0x00000007bcfc2400, 0x00000007c0000000)
 Metaspace       used 2697K, capacity 4486K, committed 4864K, reserved 1056768K
  class space    used 295K, capacity 386K, committed 512K, reserved 1048576K
```

- **日志分析**

- - **命令行参数：**

    ```
    -XX:InitialHeapSize=268435456 -XX:MaxHeapSize=536870912 -XX:+PrintGC -XX:+PrintGCDateStamps -XX:+PrintGCDetails
    -XX:+PrintGCTimeStamps -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:+UseSerialGC
    ```

- - **GC总次数为16次**

- - **Minor GC13次，Full GC3次**

- - **某一次Young GC解读**
    ```
    2020-10-24T13:23:23.518-0800: 0.154: [GC (Allocation Failure) 
    2020-10-24T13:23:23.518-0800: 0.154: [DefNew: 69543K->8704K(78656K), 0.0207395 secs]
    69543K->24408K(253440K), 0.0208665 secs] 
    [Times: user=0.02 sys=0.01, real=0.02 secs]
    ```
1. 程序在启动的第154ms时发生了第一次GC，是年轻代GC；
2. 产生年轻代GC的原因是为对象分配内存失败（Allocation Failure）；
3. 年轻代使用的收集器是DefNew，年轻代的空间从69534K降低到8704K降低了60830K，整个堆内存由69534K（一开始还没有老年代，年轻代代表整个堆内存）降低到
    24408K降低了45126K，可以推算出，晋升到老年代的对象占了24408K-8704K=15704K。

4. GC执行时间：串行GC的执行时间也就是STW时间，该值等于real（GC耗时）20ms user（CG线程耗时）20ms sys（系统掉切换耗时）10ms

&emsp;
- - **某一次Full GC解读**
    ```
    2020-10-24T13:23:23.800-0800: 0.436: [GC (Allocation Failure)
    2020-10-24T13:23:23.800-0800: 0.436: [DefNew: 78433K->8703K(78656K), 0.0163760 secs]
    2020-10-24T13:23:23.816-0800: 0.452: [Tenured: 189875K->167264K(190052K), 0.0378398 secs] 249663K->167264K(268708K), [Metaspace: 2691K->2691K(1056768K)], 
    0.0545653 secs] 
    ```
1. 程序在启动的第154ms时发生了第一次Full GC，发生Full GC的原因是为对象分配内存失败（Allocation Failure）；
2. 年轻代使用的收集器是DefNew，年轻代的内存从78433K降低到8703K，降低了69730K，耗时16.3毫秒；
3. 老年代使用的收集器是Tenured，老年代的内存从189875K降低到167264K，降低了22621K，耗时37.8ms，整个堆内存由249663K降低到167264K降低了82399K，Metaspace区没有变化；
4. Full GC耗时：50毫秒，这里的时间是年轻代GC和老年代GC的时间总和也等于user+sys；
5. 通过计算可以算出老年代的使用率 167264/190052=88%，老年代的占用率已经很高

&emsp;
- - **第一次Full GC后续的GC概要分析**：通过日志可以看出，堆内存在一直扩充，程序执行结束之前已经快要达到最大堆内存了（在加点时间估计就要OOM了），后续的几次Young GC和Full GC可以算出，
   老年代内存的使用率其实是在减少，分别是85%和84%，最后一次Full GC时可以看到年轻代已经没有任何垃圾回收了。

&emsp;
- - **程序执行完成后**：年轻代总共157248K, 已经使用了5650K，eden区已经使用了4%，from to区空的；老年代总共349568K, 已经使用300168K，使用率达到了85%；metaspace容量4486K已经使用2697K


## 并行GC

- **开启串行GC**

```
java -Xms256m -Xmx512m -XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xloggc:gc.log -XX:+UseParallelGC GCLogAnalysis
```

- **GC日志**

```
Java HotSpot(TM) 64-Bit Server VM (25.151-b12) for bsd-amd64 JRE (1.8.0_151-b12), built on Sep  5 2017 19:37:08 by "java_re" with gcc 4.2.1 (Based on Apple Inc. build 5658) (LLVM build 2336.11.00)
Memory: 4k page, physical 8388608k(233752k free)

/proc/meminfo:

CommandLine flags: -XX:InitialHeapSize=268435456 -XX:MaxHeapSize=536870912 -XX:+PrintGC -XX:+PrintGCDateStamps -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:+UseParallelGC 
2020-10-24T14:01:40.134-0800: 0.317: [GC (Allocation Failure) [PSYoungGen: 65536K->10740K(76288K)] 65536K->23784K(251392K), 0.0104645 secs] [Times: user=0.01 sys=0.02, real=0.01 secs] 
2020-10-24T14:01:40.164-0800: 0.347: [GC (Allocation Failure) [PSYoungGen: 76222K->10751K(141824K)] 89265K->47524K(316928K), 0.0205330 secs] [Times: user=0.01 sys=0.04, real=0.02 secs] 
2020-10-24T14:01:40.233-0800: 0.415: [GC (Allocation Failure) [PSYoungGen: 141709K->10739K(141824K)] 178483K->95792K(316928K), 0.0410582 secs] [Times: user=0.04 sys=0.07, real=0.04 secs] 
2020-10-24T14:01:40.303-0800: 0.486: [GC (Allocation Failure) [PSYoungGen: 141811K->10734K(163840K)] 226864K->135667K(338944K), 0.0342940 secs] [Times: user=0.03 sys=0.05, real=0.03 secs] 
2020-10-24T14:01:40.338-0800: 0.521: [Full GC (Ergonomics) [PSYoungGen: 10734K->0K(163840K)] [ParOldGen: 124932K->121164K(251904K)] 135667K->121164K(415744K), [Metaspace: 2691K->2691K(1056768K)], 0.0277414 secs] [Times: user=0.06 sys=0.00, real=0.03 secs] 
2020-10-24T14:01:40.392-0800: 0.574: [GC (Allocation Failure) [PSYoungGen: 153088K->10743K(163840K)] 274252K->169011K(415744K), 0.0347534 secs] [Times: user=0.02 sys=0.04, real=0.03 secs] 
2020-10-24T14:01:40.459-0800: 0.642: [GC (Allocation Failure) [PSYoungGen: 163831K->10745K(69632K)] 322099K->220650K(321536K), 0.0403120 secs] [Times: user=0.03 sys=0.04, real=0.04 secs] 
2020-10-24T14:01:40.499-0800: 0.682: [Full GC (Ergonomics) [PSYoungGen: 10745K->0K(69632K)] [ParOldGen: 209904K->181374K(334336K)] 220650K->181374K(403968K), [Metaspace: 2691K->2691K(1056768K)], 0.0311973 secs] [Times: user=0.07 sys=0.00, real=0.04 secs] 
2020-10-24T14:01:40.545-0800: 0.728: [GC (Allocation Failure) [PSYoungGen: 58852K->22243K(116736K)] 240227K->203618K(451072K), 0.0025515 secs] [Times: user=0.01 sys=0.00, real=0.00 secs] 
2020-10-24T14:01:40.555-0800: 0.738: [GC (Allocation Failure) [PSYoungGen: 81111K->37568K(116736K)] 262485K->218943K(451072K), 0.0051284 secs] [Times: user=0.02 sys=0.00, real=0.01 secs] 
2020-10-24T14:01:40.573-0800: 0.756: [GC (Allocation Failure) [PSYoungGen: 96448K->54649K(116736K)] 277823K->236023K(451072K), 0.0093564 secs] [Times: user=0.03 sys=0.00, real=0.01 secs] 
2020-10-24T14:01:40.597-0800: 0.780: [GC (Allocation Failure) [PSYoungGen: 113529K->39456K(116736K)] 294903K->253815K(451072K), 0.0195215 secs] [Times: user=0.05 sys=0.01, real=0.02 secs] 
2020-10-24T14:01:40.631-0800: 0.813: [GC (Allocation Failure) [PSYoungGen: 98336K->18142K(116736K)] 312695K->269894K(451072K), 0.0178010 secs] [Times: user=0.02 sys=0.02, real=0.01 secs] 
2020-10-24T14:01:40.663-0800: 0.845: [GC (Allocation Failure) [PSYoungGen: 77021K->17765K(116736K)] 328773K->287155K(451072K), 0.0189907 secs] [Times: user=0.02 sys=0.02, real=0.02 secs] 
2020-10-24T14:01:40.696-0800: 0.879: [GC (Allocation Failure) [PSYoungGen: 76645K->17794K(116736K)] 346035K->304467K(451072K), 0.0183164 secs] [Times: user=0.01 sys=0.02, real=0.02 secs] 
2020-10-24T14:01:40.715-0800: 0.898: [Full GC (Ergonomics) [PSYoungGen: 17794K->0K(116736K)] [ParOldGen: 286673K->237251K(349696K)] 304467K->237251K(466432K), [Metaspace: 2691K->2691K(1056768K)], 0.0346342 secs] [Times: user=0.10 sys=0.00, real=0.04 secs] 
2020-10-24T14:01:40.765-0800: 0.948: [GC (Allocation Failure) [PSYoungGen: 58880K->23806K(116736K)] 296131K->261057K(466432K), 0.0116023 secs] [Times: user=0.03 sys=0.00, real=0.01 secs] 
2020-10-24T14:01:40.790-0800: 0.973: [GC (Allocation Failure) [PSYoungGen: 82686K->20115K(116736K)] 319937K->280435K(466432K), 0.0075864 secs] [Times: user=0.02 sys=0.00, real=0.00 secs] 
2020-10-24T14:01:40.813-0800: 0.996: [GC (Allocation Failure) [PSYoungGen: 78801K->17467K(116736K)] 339121K->296700K(466432K), 0.0060933 secs] [Times: user=0.01 sys=0.00, real=0.00 secs] 
2020-10-24T14:01:40.834-0800: 1.016: [GC (Allocation Failure) [PSYoungGen: 76092K->19709K(116736K)] 355324K->315123K(466432K), 0.0162567 secs] [Times: user=0.03 sys=0.01, real=0.02 secs] 
2020-10-24T14:01:40.864-0800: 1.047: [GC (Allocation Failure) [PSYoungGen: 78530K->19528K(116736K)] 373943K->333871K(466432K), 0.0187814 secs] [Times: user=0.02 sys=0.02, real=0.02 secs] 
2020-10-24T14:01:40.883-0800: 1.066: [Full GC (Ergonomics) [PSYoungGen: 19528K->0K(116736K)] [ParOldGen: 314342K->270662K(349696K)] 333871K->270662K(466432K), [Metaspace: 2691K->2691K(1056768K)], 0.0534893 secs] [Times: user=0.14 sys=0.01, real=0.05 secs] 
2020-10-24T14:01:40.945-0800: 1.128: [GC (Allocation Failure) [PSYoungGen: 58880K->19681K(116736K)] 329542K->290344K(466432K), 0.0034099 secs] [Times: user=0.01 sys=0.00, real=0.00 secs] 
2020-10-24T14:01:40.964-0800: 1.146: [GC (Allocation Failure) [PSYoungGen: 78480K->21433K(116736K)] 349143K->310775K(466432K), 0.0105855 secs] [Times: user=0.02 sys=0.00, real=0.01 secs] 
2020-10-24T14:01:40.991-0800: 1.174: [GC (Allocation Failure) [PSYoungGen: 80313K->16819K(116736K)] 369655K->326358K(466432K), 0.0054764 secs] [Times: user=0.01 sys=0.00, real=0.00 secs] 
2020-10-24T14:01:41.004-0800: 1.187: [GC (Allocation Failure) [PSYoungGen: 75699K->16855K(116736K)] 385238K->342626K(466432K), 0.0084168 secs] [Times: user=0.01 sys=0.01, real=0.01 secs] 
2020-10-24T14:01:41.012-0800: 1.195: [Full GC (Ergonomics) [PSYoungGen: 16855K->0K(116736K)] [ParOldGen: 325770K->291322K(349696K)] 342626K->291322K(466432K), [Metaspace: 2691K->2691K(1056768K)], 0.0583280 secs] [Times: user=0.14 sys=0.00, real=0.06 secs] 
Heap
 PSYoungGen      total 116736K, used 6197K [0x00000007b5580000, 0x00000007c0000000, 0x00000007c0000000)
  eden space 58880K, 10% used [0x00000007b5580000,0x00000007b5b8d6c8,0x00000007b8f00000)
  from space 57856K, 0% used [0x00000007bc780000,0x00000007bc780000,0x00000007c0000000)
  to   space 57856K, 0% used [0x00000007b8f00000,0x00000007b8f00000,0x00000007bc780000)
 ParOldGen       total 349696K, used 291322K [0x00000007a0000000, 0x00000007b5580000, 0x00000007b5580000)
  object space 349696K, 83% used [0x00000007a0000000,0x00000007b1c7ea00,0x00000007b5580000)
 Metaspace       used 2697K, capacity 4486K, committed 4864K, reserved 1056768K
  class space    used 295K, capacity 386K, committed 512K, reserved 1048576K
```

- **日志分析**

- - **命令行参数**

    ```
    -XX:InitialHeapSize=268435456 -XX:MaxHeapSize=536870912 -XX:+PrintGC -XX:+PrintGCDateStamps -XX:+PrintGCDetails
    -XX:+PrintGCTimeStamps -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:+UseParallelGC 
    ```

- - **GC总次数：17次**

- - **Young GC次数22次，Full GC次数5次**

- - **某一次Young GC解读，这里选第一次**
    ```
    2020-10-24T14:01:40.134-0800: 0.317: [GC (Allocation Failure) [PSYoungGen: 65536K->10740K(76288K)] 
    65536K->23784K(251392K), 0.0104645 secs] [Times: user=0.01 sys=0.02, real=0.01 secs] 
    ```
1. 程序在启动317毫秒时发生了第一次年轻代GC，发生GC的原因是为对象分配内存失败（Allocation Failure）；
2. 年轻代使用的收集器是PSYoungGen，年轻代内存从65536K降低到10740K，降低了54796K，年轻代的容量为76288K；
3. 整个堆内存从65536K降低到23784K，降低了41752K，整个堆容量为251392K；
4. 通过数据可以计算出有54796K-41752K=13044K对象晋升到老年代，进而可以算出老年代的使用率13044/(251392-76288)=7.4%，可以看出使用率很低；
5. Young GC耗时：由于是并行的GC，不能像串行GC那样计算耗时，只能是采用约等于的方式，即：user+sys/GC线程数，这里算出来大约是11.2毫秒。

&emsp;

- - **某一次Full GC解读，这里选第一次Full GC**

    ```
    2020-10-24T14:01:40.338-0800: 0.521: [Full GC (Ergonomics) [PSYoungGen: 10734K->0K(163840K)]
    [ParOldGen: 124932K->121164K(251904K)] 135667K->121164K(415744K), 
    [Metaspace: 2691K->2691K(1056768K)], 0.0277414 secs] 
    [Times: user=0.06 sys=0.00, real=0.03 secs] 
    ```
  
1. 程序在启动的第521ms时发生了第一次Full GC，发生Full GC的原因是为对象分配内存失败；
2. 年轻代使用的收集器是PSYoungGen，年轻代的内存从10734K降低到0K，降低了10734K，年轻代容量163840K；
3. 老年代使用的收集器是ParOldGen（标记-清楚-整理），老年代的内存从124932K降低到121164K，降低了3768K，老年代容量251904K，
        整个堆内存由135667K降低到121164K降低了14503K，整个堆容量415744K，Metaspace区没有变化；
4. Full GC耗时：0.0277414秒；
5. 通过计算可以算出GC后老年代的使用率 121164/251904=48%，GC前老年代使用率124932/251904=49%，可以看出老年代回收率很低；
  
6. 第一次Full GC后续的GC概要分析：从GC日志中可以看出，年轻代的大小一直没有变化，老年代的大小会随着Full GC的触发而增加，增加的幅度也不是很大；
   此外我们可以算出后四次Full GC后老年代的使用率，分别是第二次GC后54%，第三次67%，第四次77%，第五次83%，结合第一次Full GC可以看出老年代的
   使用率逐渐增加。

## CMS GC


- **第一次CMS GC分析**

```
java -Xms256m -Xmx512m -XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xloggc:gc.log -XX:+UseConcMarkSweepGC GCLogAnalysis
```

```
Java HotSpot(TM) 64-Bit Server VM (25.151-b12) for bsd-amd64 JRE (1.8.0_151-b12), built on Sep  5 2017 19:37:08 by "java_re" with gcc 4.2.1 (Based on Apple Inc. build 5658) (LLVM build 2336.11.00)
Memory: 4k page, physical 8388608k(946228k free)

/proc/meminfo:

CommandLine flags: -XX:InitialHeapSize=268435456 -XX:MaxHeapSize=536870912 -XX:MaxNewSize=178958336 -XX:MaxTenuringThreshold=6 -XX:NewSize=178958336 -XX:OldPLABSize=16 -XX:OldSize=357912576 -XX:+PrintGC -XX:+PrintGCDateStamps -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:+UseConcMarkSweepGC -XX:+UseParNewGC 
2020-10-27T23:05:04.012-0800: 0.162: [GC (Allocation Failure) 2020-10-27T23:05:04.012-0800: 0.162: [ParNew: 139776K->17471K(157248K), 0.0244108 secs] 139776K->49466K(244672K), 0.0245300 secs] [Times: user=0.02 sys=0.04, real=0.02 secs] 
2020-10-27T23:05:04.055-0800: 0.205: [GC (Allocation Failure) 2020-10-27T23:05:04.055-0800: 0.205: [ParNew: 157247K->17471K(157248K), 0.0157752 secs] 189242K->95015K(244672K), 0.0158560 secs] [Times: user=0.03 sys=0.03, real=0.02 secs] 
2020-10-27T23:05:04.071-0800: 0.221: [GC (CMS Initial Mark) [1 CMS-initial-mark: 77544K(87424K)] 97913K(244672K), 0.0010923 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
2020-10-27T23:05:04.073-0800: 0.222: [CMS-concurrent-mark-start]
2020-10-27T23:05:04.077-0800: 0.226: [CMS-concurrent-mark: 0.004/0.004 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
2020-10-27T23:05:04.077-0800: 0.226: [CMS-concurrent-preclean-start]
2020-10-27T23:05:04.077-0800: 0.226: [CMS-concurrent-preclean: 0.000/0.000 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
2020-10-27T23:05:04.077-0800: 0.226: [CMS-concurrent-abortable-preclean-start]
2020-10-27T23:05:04.088-0800: 0.238: [GC (Allocation Failure) 2020-10-27T23:05:04.088-0800: 0.238: [ParNew: 157247K->17472K(157248K), 0.0257581 secs] 234791K->141377K(281800K), 0.0258409 secs] [Times: user=0.09 sys=0.02, real=0.03 secs] 
2020-10-27T23:05:04.130-0800: 0.280: [GC (Allocation Failure) 2020-10-27T23:05:04.130-0800: 0.280: [ParNew: 156998K->17472K(157248K), 0.0228296 secs] 280904K->183141K(323464K), 0.0229105 secs] [Times: user=0.08 sys=0.01, real=0.02 secs] 
2020-10-27T23:05:04.169-0800: 0.318: [GC (Allocation Failure) 2020-10-27T23:05:04.169-0800: 0.318: [ParNew: 157140K->17472K(157248K), 0.0253436 secs] 322809K->231624K(372104K), 0.0254245 secs] [Times: user=0.08 sys=0.01, real=0.03 secs] 
2020-10-27T23:05:04.209-0800: 0.359: [GC (Allocation Failure) 2020-10-27T23:05:04.209-0800: 0.359: [ParNew: 157248K->17472K(157248K), 0.0263273 secs] 371400K->279590K(420128K), 0.0264075 secs] [Times: user=0.09 sys=0.02, real=0.03 secs] 
2020-10-27T23:05:04.252-0800: 0.402: [GC (Allocation Failure) 2020-10-27T23:05:04.253-0800: 0.402: [ParNew: 156837K->17470K(157248K), 0.0217489 secs] 418956K->318930K(459336K), 0.0218274 secs] [Times: user=0.07 sys=0.01, real=0.02 secs] 
2020-10-27T23:05:04.290-0800: 0.439: [GC (Allocation Failure) 2020-10-27T23:05:04.290-0800: 0.439: [ParNew: 157246K->157246K(157248K), 0.0000199 secs]2020-10-27T23:05:04.290-0800: 0.439: [CMS2020-10-27T23:05:04.290-0800: 0.440: [CMS-concurrent-abortable-preclean: 0.004/0.213 secs] [Times: user=0.52 sys=0.07, real=0.22 secs] 
 (concurrent mode failure): 301460K->248144K(302088K), 0.0534466 secs] 458706K->248144K(459336K), [Metaspace: 2691K->2691K(1056768K)], 0.0536865 secs] [Times: user=0.04 sys=0.00, real=0.05 secs] 
2020-10-27T23:05:04.361-0800: 0.510: [GC (Allocation Failure) 2020-10-27T23:05:04.361-0800: 0.510: [ParNew: 139776K->17470K(157248K), 0.0067239 secs] 387920K->293121K(506816K), 0.0068100 secs] [Times: user=0.03 sys=0.00, real=0.00 secs] 
2020-10-27T23:05:04.368-0800: 0.517: [GC (CMS Initial Mark) [1 CMS-initial-mark: 275650K(349568K)] 293409K(506816K), 0.0000978 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
2020-10-27T23:05:04.368-0800: 0.517: [CMS-concurrent-mark-start]
2020-10-27T23:05:04.370-0800: 0.519: [CMS-concurrent-mark: 0.002/0.002 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
2020-10-27T23:05:04.370-0800: 0.519: [CMS-concurrent-preclean-start]
2020-10-27T23:05:04.370-0800: 0.520: [CMS-concurrent-preclean: 0.000/0.000 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
2020-10-27T23:05:04.370-0800: 0.520: [CMS-concurrent-abortable-preclean-start]
2020-10-27T23:05:04.385-0800: 0.534: [GC (Allocation Failure) 2020-10-27T23:05:04.385-0800: 0.534: [ParNew: 157246K->17471K(157248K), 0.0152972 secs] 432897K->337266K(506816K), 0.0153812 secs] [Times: user=0.05 sys=0.01, real=0.02 secs] 
2020-10-27T23:05:04.417-0800: 0.566: [GC (Allocation Failure) 2020-10-27T23:05:04.417-0800: 0.566: [ParNew: 157243K->157243K(157248K), 0.0000215 secs]2020-10-27T23:05:04.417-0800: 0.567: [CMS2020-10-27T23:05:04.417-0800: 0.567: [CMS-concurrent-abortable-preclean: 0.002/0.047 secs] [Times: user=0.09 sys=0.01, real=0.04 secs] 
 (concurrent mode failure): 319795K->285892K(349568K), 0.0377119 secs] 477038K->285892K(506816K), [Metaspace: 2691K->2691K(1056768K)], 0.0378370 secs] [Times: user=0.04 sys=0.00, real=0.04 secs] 
2020-10-27T23:05:04.472-0800: 0.621: [GC (Allocation Failure) 2020-10-27T23:05:04.472-0800: 0.621: [ParNew: 139776K->17470K(157248K), 0.0061197 secs] 425668K->327835K(506816K), 0.0062002 secs] [Times: user=0.03 sys=0.00, real=0.00 secs] 
2020-10-27T23:05:04.478-0800: 0.627: [GC (CMS Initial Mark) [1 CMS-initial-mark: 310364K(349568K)] 328034K(506816K), 0.0001819 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
2020-10-27T23:05:04.478-0800: 0.627: [CMS-concurrent-mark-start]
2020-10-27T23:05:04.480-0800: 0.629: [CMS-concurrent-mark: 0.002/0.002 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
2020-10-27T23:05:04.480-0800: 0.629: [CMS-concurrent-preclean-start]
2020-10-27T23:05:04.481-0800: 0.630: [CMS-concurrent-preclean: 0.000/0.000 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
2020-10-27T23:05:04.481-0800: 0.630: [CMS-concurrent-abortable-preclean-start]
2020-10-27T23:05:04.495-0800: 0.644: [GC (Allocation Failure) 2020-10-27T23:05:04.495-0800: 0.644: [ParNew (promotion failed): 157246K->157247K(157248K), 0.0189659 secs]2020-10-27T23:05:04.514-0800: 0.663: [CMS2020-10-27T23:05:04.514-0800: 0.663: [CMS-concurrent-abortable-preclean: 0.000/0.034 secs] [Times: user=0.08 sys=0.01, real=0.03 secs] 
 (concurrent mode failure): 349196K->306785K(349568K), 0.0389500 secs] 467611K->306785K(506816K), [Metaspace: 2691K->2691K(1056768K)], 0.0580328 secs] [Times: user=0.09 sys=0.01, real=0.06 secs] 
2020-10-27T23:05:04.570-0800: 0.719: [GC (Allocation Failure) 2020-10-27T23:05:04.570-0800: 0.719: [ParNew: 139484K->139484K(157248K), 0.0000199 secs]2020-10-27T23:05:04.570-0800: 0.719: [CMS: 306785K->311186K(349568K), 0.0382646 secs] 446270K->311186K(506816K), [Metaspace: 2691K->2691K(1056768K)], 0.0383869 secs] [Times: user=0.04 sys=0.00, real=0.03 secs] 
...
Heap
 par new generation   total 157248K, used 16579K [0x00000007a0000000, 0x00000007aaaa0000, 0x00000007aaaa0000)
  eden space 139776K,  11% used [0x00000007a0000000, 0x00000007a1030df0, 0x00000007a8880000)
  from space 17472K,   0% used [0x00000007a8880000, 0x00000007a8880000, 0x00000007a9990000)
  to   space 17472K,   0% used [0x00000007a9990000, 0x00000007a9990000, 0x00000007aaaa0000)
 concurrent mark-sweep generation total 349568K, used 349439K [0x00000007aaaa0000, 0x00000007c0000000, 0x00000007c0000000)
 Metaspace       used 2698K, capacity 4486K, committed 4864K, reserved 1056768K
  class space    used 295K, capacity 386K, committed 512K, reserved 1048576K
```
- - **第一次CMS GC日志分析**

1. 程序启动命令：java -Xms256m -Xmx512m -XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xloggc:gc.log -XX:+UseConcMarkSweepGC GCLogAnalysis
2. 虚拟机版本：25.151-b12
3. 命令行参数： -XX:InitialHeapSize=268435456 -XX:MaxHeapSize=536870912 -XX:MaxNewSize=178958336 -XX:MaxTenuringThreshold=6 -XX:NewSize=178958336
 -XX:OldPLABSize=16 -XX:OldSize=357912576 -XX:+PrintGC -XX:+PrintGCDateStamps -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+UseCompressedClassPointers
    -XX:+UseCompressedOops -XX:+UseConcMarkSweepGC -XX:+UseParNewGC 
4. 某一次Young GC解读，这里选第一次
2020-10-27T23:17:17.656-0800: 0.310: [GC (Allocation Failure) 2020-10-27T23:17:17.656-0800: 0.311: 
[ParNew: 139677K->17470K(157248K), 0.0139546 secs] 139677K->42932K(506816K), 0.0140845 secs] 
[Times: user=0.02 sys=0.03, real=0.02 secs] 
    （1）程序在启动310毫秒时发生了第一次年轻代GC，发生GC的原因是为对象分配内存失败（Allocation Failure）；
    （2）年轻代使用的收集器是ParNew，年轻代内存从139677K降低到17470K，降低了122207K，年轻代的容量为157248K；
    （3）整个堆内存从139677K降低到42932K，降低了96745K，整个堆容量为506816K；
    （4）通过数据可以计算出有122207K-96745K=25462K对象晋升到老年代，进而可以算出老年代的使用率25462/(506816-157248)=7.3%，可以看出使用率很低；
    （5）Young GC耗时：大约是27毫秒，这里计算规则类似于并行GC
  
5. 某一次Full GC解读，这里选第一次Full GC
```
2020-10-27T23:17:17.836-0800: 0.490: [GC (CMS Initial Mark) [1 CMS-initial-mark: 202233K(349568K)] 219850K(506816K), 0.0001629 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
2020-10-27T23:17:17.836-0800: 0.490: [CMS-concurrent-mark-start]
2020-10-27T23:17:17.840-0800: 0.495: [CMS-concurrent-mark: 0.004/0.004 secs] [Times: user=0.01 sys=0.00, real=0.01 secs] 
2020-10-27T23:17:17.840-0800: 0.495: [CMS-concurrent-preclean-start]
2020-10-27T23:17:17.841-0800: 0.495: [CMS-concurrent-preclean: 0.001/0.001 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
2020-10-27T23:17:17.841-0800: 0.495: [CMS-concurrent-abortable-preclean-start]
2020-10-27T23:17:17.974-0800: 0.628: [CMS2020-10-27T23:17:17.974-0800: 0.628: [CMS-concurrent-abortable-preclean: 0.004/0.133 secs] [Times: user=0.28 sys=0.04, real=0.13 secs] 
 (concurrent mode failure): 323971K->236801K(349568K), 0.0488632 secs] 481219K->236801K(506816K), [Metaspace: 2691K->2691K(1056768K)], 0.0489962 secs] [Times: user=0.05 sys=0.00, real=0.05 secs] 
```

（1）程序在启动的第490ms时发生了第一次CMS GC

（2）从日志中可以看出CMS GC执行了初始标记、并发标记、并发预清理后就结束了，并没有进行最终标记、并发清除和并发重置，可以在日志发现concurrent mode failure，该错误导致的原因是在CMS并发执行GC过程中同时业务线程将对象放入老年代，而老年代空间不足，
   或者在做Minor GC时，Survivor放不下，需要进入老年代，而老年代空间不足造成的；仔细观察日志，在后续的第三次CMS GC时又报了一个promotion failed，此时就是因为Survivor放不下，需要进入老年代，这时候CMS GC会降级成Serial GC。

- **第二次CMS GC分析**

调整JVM参数进行第二次CMS GC 分析

```
java -Xms2g -Xmx2g -XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xloggc:gc.log -XX:+UseConcMarkSweepGC GCLogAnalysis，这一次我们将堆调成了初始化和最大都为2g
```

```
Java HotSpot(TM) 64-Bit Server VM (25.151-b12) for bsd-amd64 JRE (1.8.0_151-b12), built on Sep  5 2017 19:37:08 by "java_re" with gcc 4.2.1 (Based on Apple Inc. build 5658) (LLVM build 2336.11.00)
Memory: 4k page, physical 8388608k(4075116k free)

/proc/meminfo:

CommandLine flags: -XX:InitialHeapSize=2147483648 -XX:MaxHeapSize=2147483648 -XX:MaxNewSize=348966912 -XX:MaxTenuringThreshold=6 -XX:NewSize=348966912 -XX:OldPLABSize=16 -XX:OldSize=697933824 -XX:+PrintGC -XX:+PrintGCDateStamps -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:+UseConcMarkSweepGC -XX:+UseParNewGC 
2020-10-27T23:21:31.129-0800: 0.212: [GC (Allocation Failure) 2020-10-27T23:21:31.129-0800: 0.212: [ParNew: 272640K->34047K(306688K), 0.0342931 secs] 272640K->81571K(2063104K), 0.0344175 secs] [Times: user=0.05 sys=0.06, real=0.04 secs] 
2020-10-27T23:21:31.197-0800: 0.281: [GC (Allocation Failure) 2020-10-27T23:21:31.197-0800: 0.281: [ParNew: 306687K->34048K(306688K), 0.0296918 secs] 354211K->156584K(2063104K), 0.0297917 secs] [Times: user=0.06 sys=0.06, real=0.03 secs] 
2020-10-27T23:21:31.258-0800: 0.341: [GC (Allocation Failure) 2020-10-27T23:21:31.258-0800: 0.341: [ParNew: 306688K->34048K(306688K), 0.0409308 secs] 429224K->234993K(2063104K), 0.0410245 secs] [Times: user=0.14 sys=0.02, real=0.04 secs] 
2020-10-27T23:21:31.328-0800: 0.411: [GC (Allocation Failure) 2020-10-27T23:21:31.328-0800: 0.411: [ParNew: 306688K->34048K(306688K), 0.0419711 secs] 507633K->313577K(2063104K), 0.0420661 secs] [Times: user=0.14 sys=0.02, real=0.05 secs] 
2020-10-27T23:21:31.399-0800: 0.483: [GC (Allocation Failure) 2020-10-27T23:21:31.399-0800: 0.483: [ParNew: 306688K->34048K(306688K), 0.0410468 secs] 586217K->390650K(2063104K), 0.0411321 secs] [Times: user=0.14 sys=0.02, real=0.05 secs] 
2020-10-27T23:21:31.471-0800: 0.554: [GC (Allocation Failure) 2020-10-27T23:21:31.471-0800: 0.554: [ParNew: 306688K->34048K(306688K), 0.0392424 secs] 663290K->463061K(2063104K), 0.0393437 secs] [Times: user=0.13 sys=0.02, real=0.04 secs] 
2020-10-27T23:21:31.540-0800: 0.623: [GC (Allocation Failure) 2020-10-27T23:21:31.540-0800: 0.623: [ParNew: 306688K->34048K(306688K), 0.0423618 secs] 735701K->532575K(2063104K), 0.0424609 secs] [Times: user=0.12 sys=0.02, real=0.04 secs] 
2020-10-27T23:21:31.612-0800: 0.695: [GC (Allocation Failure) 2020-10-27T23:21:31.612-0800: 0.695: [ParNew: 306688K->34048K(306688K), 0.0413909 secs] 805215K->609074K(2063104K), 0.0414911 secs] [Times: user=0.15 sys=0.02, real=0.04 secs] 
2020-10-27T23:21:31.684-0800: 0.767: [GC (Allocation Failure) 2020-10-27T23:21:31.684-0800: 0.767: [ParNew: 306688K->34048K(306688K), 0.0378931 secs] 881714K->677658K(2063104K), 0.0379909 secs] [Times: user=0.13 sys=0.02, real=0.04 secs] 
2020-10-27T23:21:31.751-0800: 0.835: [GC (Allocation Failure) 2020-10-27T23:21:31.751-0800: 0.835: [ParNew: 306688K->34048K(306688K), 0.0404181 secs] 950298K->751633K(2063104K), 0.0405174 secs] [Times: user=0.14 sys=0.02, real=0.04 secs] 
2020-10-27T23:21:31.822-0800: 0.906: [GC (Allocation Failure) 2020-10-27T23:21:31.822-0800: 0.906: [ParNew: 306688K->34048K(306688K), 0.0413815 secs] 1024273K->832225K(2063104K), 0.0414843 secs] [Times: user=0.14 sys=0.02, real=0.04 secs] 
2020-10-27T23:21:31.892-0800: 0.976: [GC (Allocation Failure) 2020-10-27T23:21:31.892-0800: 0.976: [ParNew: 306688K->34048K(306688K), 0.0411901 secs] 1104865K->912519K(2063104K), 0.0412827 secs] [Times: user=0.14 sys=0.02, real=0.04 secs] 
2020-10-27T23:21:31.934-0800: 1.017: [GC (CMS Initial Mark) [1 CMS-initial-mark: 878471K(1756416K)] 918167K(2063104K), 0.0062490 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
2020-10-27T23:21:31.940-0800: 1.024: [CMS-concurrent-mark-start]
2020-10-27T23:21:31.962-0800: 1.045: [CMS-concurrent-mark: 0.022/0.022 secs] [Times: user=0.03 sys=0.01, real=0.02 secs] 
2020-10-27T23:21:31.962-0800: 1.045: [CMS-concurrent-preclean-start]
2020-10-27T23:21:31.964-0800: 1.048: [CMS-concurrent-preclean: 0.002/0.002 secs] [Times: user=0.01 sys=0.00, real=0.00 secs] 
2020-10-27T23:21:31.964-0800: 1.048: [CMS-concurrent-abortable-preclean-start]
2020-10-27T23:21:31.973-0800: 1.056: [GC (Allocation Failure) 2020-10-27T23:21:31.973-0800: 1.056: [ParNew: 306688K->34048K(306688K), 0.0460958 secs] 1185159K->1004554K(2063104K), 0.0462124 secs] [Times: user=0.16 sys=0.02, real=0.04 secs] 
Heap
 par new generation   total 306688K, used 45266K [0x0000000740000000, 0x0000000754cc0000, 0x0000000754cc0000)
  eden space 272640K,   4% used [0x0000000740000000, 0x0000000740af4b00, 0x0000000750a40000)
  from space 34048K, 100% used [0x0000000752b80000, 0x0000000754cc0000, 0x0000000754cc0000)
  to   space 34048K,   0% used [0x0000000750a40000, 0x0000000750a40000, 0x0000000752b80000)
 concurrent mark-sweep generation total 1756416K, used 970506K [0x0000000754cc0000, 0x00000007c0000000, 0x00000007c0000000)
 Metaspace       used 2697K, capacity 4486K, committed 4864K, reserved 1056768K
  class space    used 295K, capacity 386K, committed 512K, reserved 1048576K
```
- **第二次CMS GC日志分析**

1. 程序启动命令：java -Xms2g -Xmx2g -XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xloggc:gc.log -XX:+UseConcMarkSweepGC GCLogAnalysis，这一次我们将堆调成了初始化和最大都为2g
2. 虚拟机版本：25.151-b12
3. 命令行参数: -XX:InitialHeapSize=2147483648 -XX:MaxHeapSize=2147483648 -XX:MaxNewSize=348966912 -XX:MaxTenuringThreshold=6 -XX:NewSize=348966912
 -XX:OldPLABSize=16 -XX:OldSize=697933824 -XX:+PrintGC -XX:+PrintGCDateStamps -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+UseCompressedClassPointers
    -XX:+UseCompressedOops -XX:+UseConcMarkSweepGC -XX:+UseParNewGC 

4、CMS GC日志解读：当堆内存调大为2g后，可以看到并没有产生CMS GC失败或者退化到串行GC的情况，可以看到，一共进行了13次Minor GC和一次CMS GC，并且没等CMS GC完成程序就执行完成了

   （1）年轻代收集器：ParNew，并且通过分析GC 日志中的耗时可以看出，GC的耗时在增加，这是因为堆较大，当发生GC时占用内存大，执行GC的时间耗时就增加了；

   （2）这里没有一个完整的CMS GC产生，下面我跟我将Xms和Xmx设置成1g，记录了一次完整的CMS GC
```
       2020-10-28T00:07:42.097-0800: 0.779: [GC (CMS Initial Mark) [1 CMS-initial-mark: 354992K(707840K)] 389423K(1014528K), 0.0001729 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
       2020-10-28T00:07:42.098-0800: 0.779: [CMS-concurrent-mark-start]
       2020-10-28T00:07:42.107-0800: 0.789: [CMS-concurrent-mark: 0.009/0.009 secs] [Times: user=0.01 sys=0.01, real=0.01 secs] 
       2020-10-28T00:07:42.107-0800: 0.789: [CMS-concurrent-preclean-start]
       2020-10-28T00:07:42.109-0800: 0.791: [CMS-concurrent-preclean: 0.002/0.002 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
       2020-10-28T00:07:42.109-0800: 0.791: [CMS-concurrent-abortable-preclean-start]
       2020-10-28T00:07:42.298-0800: 0.980: [CMS-concurrent-abortable-preclean: 0.004/0.190 secs] [Times: user=0.34 sys=0.04, real=0.19 secs] 
       
       2020-10-28T00:07:42.343-0800: 1.024: [GC (CMS Final Remark) [YG occupancy: 34477 K (306688 K)]
       2020-10-28T00:07:42.343-0800: 1.024: [Rescan (parallel) , 0.0021714 secs]
       2020-10-28T00:07:42.345-0800: 1.027: [weak refs processing, 0.0000238 secs]
       2020-10-28T00:07:42.345-0800: 1.027: [class unloading, 0.0002818 secs]
       2020-10-28T00:07:42.345-0800: 1.027: [scrub symbol table, 0.0002872 secs]
       2020-10-28T00:07:42.345-0800: 1.027: [scrub string table, 0.0001214 secs]
       [1 CMS-remark: 508915K(707840K)] 543392K(1014528K), 0.0029799 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
       
       2020-10-28T00:07:42.346-0800: 1.027: [CMS-concurrent-sweep-start]
       2020-10-28T00:07:42.347-0800: 1.029: [CMS-concurrent-sweep: 0.001/0.001 secs] [Times: user=0.01 sys=0.00, real=0.00 secs] 
       2020-10-28T00:07:42.347-0800: 1.029: [CMS-concurrent-reset-start]
       2020-10-28T00:07:42.348-0800: 1.030: [CMS-concurrent-reset: 0.001/0.001 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
       
       a.初始标记（STW）：2020-10-28T00:07:42.097-0800: 0.779: [GC (CMS Initial Mark) [1 CMS-initial-mark: 354992K(707840K)] 389423K(1014528K), 0.0001729 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
       b.并发标记：2020-10-28T00:07:42.098-0800: 0.779: [CMS-concurrent-mark-start]
                 2020-10-28T00:07:42.107-0800: 0.789: [CMS-concurrent-mark: 0.009/0.009 secs] [Times: user=0.01 sys=0.01, real=0.01 secs] 
       c.并发预清理：2020-10-28T00:07:42.107-0800: 0.789: [CMS-concurrent-preclean-start]
               2020-10-28T00:07:42.109-0800: 0.791: [CMS-concurrent-preclean: 0.002/0.002 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
       d.并发可终止预清理阶段：2020-10-28T00:07:42.109-0800: 0.791: [CMS-concurrent-abortable-preclean-start]
         增加这一阶段是为了让我们可以控制这个阶段的结束时机，比如扫描多长时间（默认5秒）或者Eden区使用占比达到期望比例（默认50%）就结束本阶段。
         
       e.最终标记步骤：
                     i.年轻代当前占用情况和容量：2020-10-28T00:07:42.343-0800: 1.024: [GC (CMS Final Remark) [YG occupancy: 34477 K (306688 K)]2020-10-28T00:07:42.343-0800: 1.024: 
                     ii.在应用停止的阶段完成存活对象的标记工作：[Rescan (parallel) , 0.0021714 secs]
                     iii.弱引用处理：2020-10-28T00:07:42.345-0800: 1.027: [weak refs processing, 0.0000238 secs]
                     iv.2020-10-28T00:07:42.345-0800: 1.027: [class unloading, 0.0002818 secs]
                     v.2020-10-28T00:07:42.345-0800: 1.027: [scrub symbol table, 0.0002872 secs] 2020-10-28T00:07:42.345-0800: 1.027: [scrub string table, 0.0001214 secs]
                     vi.老年代大小和老年代容量：[1 CMS-remark: 508915K(707840K)] 
                     vii.整个堆大小和容量：543392K(1014528K), 
                     viii.持续时间：0.0029799 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
       f.并发清除：2020-10-28T00:07:42.346-0800: 1.027: [CMS-concurrent-sweep-start]
              2020-10-28T00:07:42.347-0800: 1.029: [CMS-concurrent-sweep: 0.001/0.001 secs] [Times: user=0.01 sys=0.00, real=0.00 secs] 
       g.并发重置：2020-10-28T00:07:42.347-0800: 1.029: [CMS-concurrent-reset-start]
              2020-10-28T00:07:42.348-0800: 1.030: [CMS-concurrent-reset: 0.001/0.001 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
```

## G1 GC

G1 GC日志：G1 GC的GC过程有两种模式Young GC和Mixed GC，接下来根据日志分析两种模式，由于日志量过大这里只分别粘出Young GC和Mixed GC的一段日志进行分析

1. 命令行参数：-XX:InitialHeapSize=1073741824 -XX:MaxHeapSize=1073741824 -XX:+PrintGC -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:+UseG1GC

- **堆信息日志分析**

从堆信息日志中可以看出，使用的是G1（garbage-first），堆的总容量是1048576K，已经使用了668067K；每个region大小是1024K（1M），128个region是young区（131072K），7个region是survivors（7168K）；
    Metaspace 已经使用了2696K，容量是4486K，当前可使用大小4864K，元数据保留空间1056768K
    类似于Metaspace，class space与之相似。

````
Heap
 garbage-first heap   total 1048576K, used 668067K [0x0000000780000000, 0x0000000780102000, 0x00000007c0000000)
  region size 1024K, 128 young (131072K), 7 survivors (7168K)
 Metaspace       used 2696K, capacity 4486K, committed 4864K, reserved 1056768K
  class space    used 295K, capacity 386K, committed 512K, reserved 1048576K
````
- **Young GC：该种模式GC选定所有年轻代Region，通过控制年轻代Region的个数（或者说大小）来控制Young GC时间开销**

```
0.129: [GC pause (G1 Evacuation Pause) (young), 0.0088526 secs]
   [Parallel Time: 8.2 ms, GC Workers: 8]
      [GC Worker Start (ms): Min: 128.8, Avg: 128.9, Max: 128.9, Diff: 0.1]
      [Ext Root Scanning (ms): Min: 0.1, Avg: 0.2, Max: 0.3, Diff: 0.2, Sum: 1.4]
      [Update RS (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
         [Processed Buffers: Min: 0, Avg: 0.0, Max: 0, Diff: 0, Sum: 0]
      [Scan RS (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [Code Root Scanning (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [Object Copy (ms): Min: 7.3, Avg: 7.6, Max: 7.9, Diff: 0.6, Sum: 60.9]
      [Termination (ms): Min: 0.0, Avg: 0.3, Max: 0.6, Diff: 0.6, Sum: 2.3]
         [Termination Attempts: Min: 1, Avg: 1.0, Max: 1, Diff: 0, Sum: 8]
      [GC Worker Other (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.1]
      [GC Worker Total (ms): Min: 8.1, Avg: 8.1, Max: 8.1, Diff: 0.1, Sum: 64.8]
      [GC Worker End (ms): Min: 137.0, Avg: 137.0, Max: 137.0, Diff: 0.0]
   [Code Root Fixup: 0.0 ms]
   [Code Root Purge: 0.0 ms]
   [Clear CT: 0.1 ms]
   [Other: 0.5 ms]
      [Choose CSet: 0.0 ms]
      [Ref Proc: 0.2 ms]
      [Ref Enq: 0.0 ms]
      [Redirty Cards: 0.1 ms]
      [Humongous Register: 0.1 ms]
      [Humongous Reclaim: 0.0 ms]
      [Free CSet: 0.0 ms]
   [Eden: 51.0M(51.0M)->0.0B(44.0M) Survivors: 0.0B->7168.0K Heap: 63.7M(1024.0M)->24.9M(1024.0M)]
 [Times: user=0.02 sys=0.04, real=0.01 secs] 
```
**Young GC日志分析：**
1. 程序在启动的第128毫秒发生了第一次GC；
2. [GC pause (G1 Evacuation Pause) (young), 0.0088526 secs]：这是个新生代的Minor GC，该段表示将活着的对象从一个区域拷贝到另一个区域；
3. GC Workers: 8：GC的工作线程有8个；
4. [Ext Root Scanning (ms): Min: 0.1, Avg: 0.2, Max: 0.3, Diff: 0.2, Sum: 1.4] 表示扫描Roots花费的时间；
5. [Update RS (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0] [Processed Buffers: Min: 0, Avg: 0.0, Max: 0, Diff: 0, Sum: 0] 表示每个线程更新remember set花费的时间；
6. [Scan RS (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0] 扫描CSet中region对应的remeber set的时间；
7. [Code Root Scanning (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0] 扫描code roots耗时，code roots是JIT编译后的代码里引用了堆中的对象。引用关系保存在remember set中；
8. [Object Copy (ms): Min: 7.3, Avg: 7.6, Max: 7.9, Diff: 0.6, Sum: 60.9] 拷贝活的对象到别的region耗时；
9. [Termination (ms): Min: 0.0, Avg: 0.3, Max: 0.6, Diff: 0.6, Sum: 2.3] 线程结束
10. [GC Worker Other (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.1] 花费在其他工作上时间
11. [GC Worker Total (ms): Min: 8.1, Avg: 8.1, Max: 8.1, Diff: 0.1, Sum: 64.8] 每个线程耗时的和
12. [GC Worker End (ms): Min: 137.0, Avg: 137.0, Max: 137.0, Diff: 0.0] 每个线程的结束时间
13. [Code Root Fixup: 0.0 ms]
      [Code Root Purge: 0.0 ms]
      [Clear CT: 0.1 ms]
      分别是修正Code Root耗时；清理Code Root引用耗时；清理Card Table耗时
14. [Other: 0.5 ms]
            [Choose CSet: 0.0 ms]
            [Ref Proc: 0.2 ms]
            [Ref Enq: 0.0 ms]
            [Redirty Cards: 0.1 ms]
            [Humongous Register: 0.1 ms]
            [Humongous Reclaim: 0.0 ms]
            [Free CSet: 0.0 ms]
      其他耗时：选择CSet，处理已用对象，引用入ReferenceQueues，释放CSet中的region到free list。
15. [Eden: 51.0M(51.0M)->0.0B(44.0M) Survivors: 0.0B->7168.0K Heap: 63.7M(1024.0M)->24.9M(1024.0M)] 新生代被清空了，下次大小为44.0M，幸存者区增加到7168.0K，整个堆由63.7M降低到24.9M。

- **Mixed GC：顾名思义，混合型GC，该种模式选定所有年轻代Region，外加根据global concurrent marking统计得出的收益较高的若干老年代Region，在用户指定的开销目标范围内尽可能选择收益高的老年代Region**

```
0.654: [GC pause (G1 Humongous Allocation) (young) (initial-mark), 0.0088469 secs]
   [Parallel Time: 8.2 ms, GC Workers: 8]
      [GC Worker Start (ms): Min: 653.9, Avg: 654.0, Max: 654.0, Diff: 0.1]
      [Ext Root Scanning (ms): Min: 0.1, Avg: 0.1, Max: 0.2, Diff: 0.1, Sum: 0.9]
      [Update RS (ms): Min: 0.2, Avg: 0.2, Max: 0.4, Diff: 0.3, Sum: 1.8]
         [Processed Buffers: Min: 0, Avg: 2.4, Max: 3, Diff: 3, Sum: 19]
      [Scan RS (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.1]
      [Code Root Scanning (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [Object Copy (ms): Min: 7.3, Avg: 7.6, Max: 7.8, Diff: 0.6, Sum: 60.6]
      [Termination (ms): Min: 0.0, Avg: 0.2, Max: 0.5, Diff: 0.5, Sum: 1.6]
         [Termination Attempts: Min: 1, Avg: 1.8, Max: 3, Diff: 2, Sum: 14]
      [GC Worker Other (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.2]
      [GC Worker Total (ms): Min: 8.1, Avg: 8.2, Max: 8.2, Diff: 0.1, Sum: 65.2]
      [GC Worker End (ms): Min: 662.1, Avg: 662.1, Max: 662.1, Diff: 0.0]
   [Code Root Fixup: 0.0 ms]
   [Code Root Purge: 0.0 ms]
   [Clear CT: 0.1 ms]
   [Other: 0.5 ms]
      [Choose CSet: 0.0 ms]
      [Ref Proc: 0.1 ms]
      [Ref Enq: 0.0 ms]
      [Redirty Cards: 0.1 ms]
      [Humongous Register: 0.1 ms]
      [Humongous Reclaim: 0.1 ms]
      [Free CSet: 0.1 ms]
   [Eden: 80.0M(281.0M)->0.0B(332.0M) Survivors: 33.0M->24.0M Heap: 510.1M(1024.0M)->431.6M(1024.0M)]
 [Times: user=0.02 sys=0.03, real=0.01 secs] 
0.663: [GC concurrent-root-region-scan-start]
0.663: [GC concurrent-root-region-scan-end, 0.0001375 secs]
0.663: [GC concurrent-mark-start]
0.666: [GC concurrent-mark-end, 0.0026340 secs]
0.666: [GC remark 0.666: [Finalize Marking, 0.0002389 secs] 0.666: [GC ref-proc, 0.0000388 secs] 0.666: [Unloading, 0.0004490 secs], 0.0015490 secs]
 [Times: user=0.01 sys=0.00, real=0.00 secs] 
0.668: [GC cleanup 449M->430M(1024M), 0.0007843 secs]
 [Times: user=0.00 sys=0.00, real=0.00 secs] 
0.668: [GC concurrent-cleanup-start]
0.668: [GC concurrent-cleanup-end, 0.0000144 secs] 
```

**Mixed GC日志分析**

1. 程序在执行的第654毫秒发生了一次Mixed GC；
2. 发生的原因是：Humongous Allocation，巨大对象分配之前，会检测到old generation 使用占比是否超过了 initiating heap occupancy percent（45%），因所以触发了本次global concurrent marking；
3. 可以看到Mixed GC前半部分几乎和Young GC一摸一样；
4. 但是后半部分就很不一样了，可以看到后续的操作大部分都是并发操作，包括：并发的扫描roots region；并发的标记；最终标记（STW）；并发清理。