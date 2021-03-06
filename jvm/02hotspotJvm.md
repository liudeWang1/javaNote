## HotSpot虚拟机对象
### 对象的内存布局
在HotSpot虚拟机中，对象内存布局分为3快区域：   
#### 对象头
对象头记录了对象在运行过程中需要使用的一些数据：  
+ 哈希码
+ GC分代年龄
+ 锁状态标志
+ 线程持有的锁
+ 偏向线程ID
+ 偏向时间戳   
可能包含类型指针，通过该指针能确定对象属于哪个类。如果对象是个数组，那么对象头还会包含数组长度。   
#### 实例数据
实例数据部分就是成员变量的值，其中包含父类成员变量和本类成员变量。  
#### 对齐填充
用于确保对象的总长度为8字节的整数倍。   
HotSpot虚拟机的自动内存管理系统要求对象的大小必须是8字节的整数倍。而对象头部分正好是8字节的倍数（1倍或2倍），因此，当对象实例数据部分没有对齐时，就要通过对齐填充来补全。   
**对齐填充不是必然存在，也没有特别含义，它仅仅起着占位符的作用。**  
### 对象的创建过程 
#### 类加载检查
虚拟机在解析.class文件时，若遇到一条new指令，首先他会去常量池中检查是否有这个类的符号引用，并且检查这个符号引用所代表的类是否已被加载、解析和初始化过，如果没有，则必须执行相应的类加载过程。   
#### 为新生对象分配内存
对象所需内存的大小在类加载完全后便可完全确定，接下来从堆中划分一块对应大小的内存空间给新对象。分配堆中内存有两种方式：  
+ 指针碰撞：如果Java堆中内存绝对规整（说明采用的是“复制算法”或“标记整理法”），空闲内存和已使用内存中间放着一个指针作为分界点指示器，那么分配内存时只用将指针向空闲内存挪动一段与对象大小一样的距离，这就是指针碰撞。
+ 空闲列表：已使用的内存和空闲内存交错（说明采用的是“标记清除法”），此时没法简单进行指针碰撞，vm必须维护一个列表，记录其中哪些内存块空闲可用。分配之时从空闲列表中找到一块足够大的内存空间划分给对象实例。   
#### 初始化
分配完内存后，为对象中的成员变量赋上初始值，设置对象头信息，调用对象的构造方法进行初始化。
### 对象的访问方式
所有对象的存储空间都是在堆中分配的，但是这个对象的引用确是在堆栈中分配的，也就是说在创建一个对象时，两个地方都分配内存，在堆中分配的内存实际创建这个对象，在堆栈中分配的内存只是一个指向这个堆对象的指针（引用），那么根据引用存放的地址类型的不同，对象有不同的访问方式。    
#### 句柄访问方式
堆中需要有一块叫做“句柄池”的内存空间，句柄中包含了实例对象数据和类型数据各自的具体信息。   
引用类型的变量存放的是该对象的句柄地址，访问对象时，首先需要通过引用类型的变量找到该对象的句柄，然后根据句柄中对象的地址找到对象。   
#### 直接指针访问方式
引用类型的变量直接存放对象的地址，从而不需要句柄池，通过引用能够直接访问对象，但对象所在的内存空间需要额外的策略存储对象所属的类信息的地址。  
**HotSpot采用第二种方式**，直接指针方式来访问对象，只需要一次寻址操作，所以在性能上比句柄访问方式快一倍，但是它需要而外的策略来存储对象在方法区中类信息的地址。