# GC日志分析方法

本篇并非GC的调优，而是介绍了当我们拿到GC日志的时候如何去分析日志

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
- - - 程序在启动的第154ms时发生了第一次GC，是年轻代GC；
- - - 产生年轻代GC的原因是为对象分配内存失败（Allocation Failure）；
- - - 年轻代使用的收集器是DefNew，年轻代的空间从69534K降低到8704K降低了60830K，整个堆内存由69534K（一开始还没有老年代，年轻代代表整个堆内存）降低到
    24408K降低了45126K，可以推算出，晋升到老年代的对象占了24408K-8704K=15704K。

- - - GC执行时间：串行GC的执行时间也就是STW时间，该值等于real（GC耗时）20ms user（CG线程耗时）20ms sys（系统掉切换耗时）10ms

&emsp;
- - **某一次Full GC解读**
    ```
    2020-10-24T13:23:23.800-0800: 0.436: [GC (Allocation Failure)
    2020-10-24T13:23:23.800-0800: 0.436: [DefNew: 78433K->8703K(78656K), 0.0163760 secs]
    2020-10-24T13:23:23.816-0800: 0.452: [Tenured: 189875K->167264K(190052K), 0.0378398 secs] 249663K->167264K(268708K), [Metaspace: 2691K->2691K(1056768K)], 
    0.0545653 secs] 
    ```
- - - 程序在启动的第154ms时发生了第一次Full GC，发生Full GC的原因是为对象分配内存失败（Allocation Failure）；
- - - 年轻代使用的收集器是DefNew，年轻代的内存从78433K降低到8703K，降低了69730K，耗时16.3毫秒；
- - - 老年代使用的收集器是Tenured，老年代的内存从189875K降低到167264K，降低了22621K，耗时37.8ms，整个堆内存由249663K降低到167264K降低了82399K，Metaspace区没有变化；
- - - Full GC耗时：50毫秒，这里的时间是年轻代GC和老年代GC的时间总和也等于user+sys；
- - - 通过计算可以算出老年代的使用率 167264/190052=88%，老年代的占用率已经很高

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
- - - 程序在启动317毫秒时发生了第一次年轻代GC，发生GC的原因是为对象分配内存失败（Allocation Failure）；
- - - 年轻代使用的收集器是PSYoungGen，年轻代内存从65536K降低到10740K，降低了54796K，年轻代的容量为76288K；
- - - 整个堆内存从65536K降低到23784K，降低了41752K，整个堆容量为251392K；
- - - 通过数据可以计算出有54796K-41752K=13044K对象晋升到老年代，进而可以算出老年代的使用率13044/(251392-76288)=7.4%，可以看出使用率很低；
- - - Young GC耗时：由于是并行的GC，不能像串行GC那样计算耗时，只能是采用约等于的方式，即：user+sys/GC线程数，这里算出来大约是11.2毫秒。


&emsp;