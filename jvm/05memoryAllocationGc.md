## 内存分配与回收策略
对象的内存分配，就是在堆上分配（也可能经过JIT编译后被拆散为标量类型并间接在栈上分配），对象主要分配在新生代的Eden区，少数情况可能直接分配在老年区，分配规则不固定，取决于当前使用的垃圾收集器组合以及相关的参数配置。  
### 对象优先在Eden区分配  
在大多数情况下，对象在新生代Eden区中分配，当Eden区没有足够空间进行分配时，虚拟机将发起一次Minor GC。   
**Minor GC vs Major GC/Full GC**   
+ Minor GC：回收新生代（包括Eden和Survivor区域），因为Java对象大都具备朝生夕灭的特点，所以Minor GC特别频繁，一般回收速度也比较快。  
+ Major GC/Full GC：回收老年代，出现了Major GC经常会伴随至少一次Minor GC，但这并非绝对。Major GC的速度一般比Minor GC慢10倍以上。   
**在JVM规范中，Major GC和Full GC都没有一个正式的定义，所以也有人简单的认为Major GC清理老年代，而Full GC清理整个内存堆。**   
### 大对象直接进入老年代   
大对象是指需要大量连续内存空间的Java对象，如很长的字符串或数据。  
一个大对象能够直接存入Eden区的概率比较小，发生分配担保的概率比较大，而分配担保需要涉及大量的复制，就会造成效率低下。   
虚拟机提供了**一个-XX:PretenureSizeThreshold**参数，令大于这个设置值的对象直接在老年代分配，这样做的目的时避免在Eden区及两个Survivor区之间发生大量的内存复制。   
### 长期存活的对象将进入老年代
JVM给每个对象定义一个对象年龄计数器，当新生代发生一次MinorGC后，存活下来的对象年龄+1，当年龄超过一定值时，就将年龄超过该值的所有对象转移到老年代去。   
**使用-XX:MaxTenuringThreshold**设置新生代的最大年龄，只要超过该参数的新生代对象都会被转移到老年代去。   
### 动态对象年龄判断  
如果当前新生代的Survivor中，相同年龄所有对象大小的总和大于Survivor空间的一半，年龄>=该年龄的对象就可以直接进入老年代，无需等待MaxTenuringThreshold中要求的年龄。   
### 空间分配担保 
JDK 6 Update 24 之前的规则是这样：   
在发生Minor GC之前，虚拟机会先检查老年代最大可用的连续空间是否大于新生代所有对象总空间，如果这个条件成立，Minor GC可以确保是安全的；如果不成立，则虚拟机会查看HandlePromotionFailure 值是否设置为允许担保失败，如果是那么会继续检查老年代最大可用的连续空间是否大于历次晋升到老年对象的平均大小，如果大于，将尝试进行一次Minor GC，尽管这次Minor GC有风险；如果小于，或者HandlePromotionFailure设置不允许冒险，那此时也要改为进行一次Full GC。   
JDK 6 Update 24之后的规则变为：   
只要老年代的连续空间大于新生代对象总空间或者历次晋升的平均大小，就会进行Minor GC，否则将进行Full GC。通过清除老年代中废弃数据来扩大老年代空闲空间，以便给新生代做担保。   
### 一些可能会触发JVM进行Full GC的情况
+ System.gc()方法调用：此方法的调用是建议JVM进行Full GC，注意**只是建议**而非一定，但在很多情况下它会触发Full GC，从而增加Full GC的频率，通常情况下我们只需要让虚拟机自己去管理内存即可，我们可以**通过-XX:+ DisableExplicitGC**来禁止调用System.gc()方法。  
+ 老年代空间不足：老年代空间不足会触发Full GC操作，若进行操作后空间依然不足，则会抛出如下错误：java.lang.OutOfMemoryError: Java heap space  
+ 永久代空间不足：JVM规范中运行时数据区域中的方法区，在HotSpot虚拟机中也称为永久代（Permanet Generation），存放一些类信息、常量、静态变量等数据，当系统要加载的类、反射的类和调用的方法较多时，永久代可能会被占满，会触发Full GC。如果经过Full GC仍然回收不了，那么JVM会抛出如下错误信息：java.lang.OutOfMemoryError: PermGen space 
+ CMS时出现promotion failed 和 concurrent mode failure，promotion failed就是担保失败，而concurrent mode failure是在执行CMS GC的过程中同时有对象要放入老年代，而此时老年代空间不足造成的。   
+ 统计得到的Minor GC晋升到老年代的平均大小大于老年代剩余空间
