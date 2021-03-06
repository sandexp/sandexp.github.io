#### 程序计数器

指示程序的运行位置，控制程序的执行流程，多线程情况下，每个线程执行的指令不一定相同，所以程序计数器的线程私有的。



#### 方法区

各个线程所共享的存储区域，用于存储静态变量，常量，类型信息等数据，也称作非堆(No-heap,Off-heap).

##### 运行时常量池

是方法区的一部分，Class文件中除了包含类的信息之外，还有一项信息是常量池表。用于存放编译器产生的字面量以及符号引用，在类加载完成之后存放到常量池中。

同时运行期也可以将常量存储到常量池中，例如String的intern方法。



#### 直接内存

用过native函数库，直接分配堆外内存，通过Java堆中的DirectByteBuffer作为引用进行操作。不需要额外的数据拷贝。



#### 堆

Java内存模型中最大的一块，线程共享的存储区域。垃圾回收主要发生的地点，逻辑上可以分为新生代，老年代，永久代等区域。新生代又分为Eden区域，Survivor区域等。



#### 栈

##### 本地方法栈

与虚拟机方法栈相似，用于维护native方法的栈帧信息。

##### 虚拟机方法栈

线程私有，生命周期与线程相同，启动线程的时候，虚拟机方法栈会存储局部变量表，操作数栈，动态链接和方法出口等信息。



#### 对象的创建过程

1. Java虚拟机遇到一条new指令，从常量池中查找是否有这个变量的缓存，如果有，则读取，否则进行2的步骤
2. 检查这个引用代表的类是否已经类加载完毕，如果没有，则进行类加载过程，进行步骤3
3. 为这个对象在堆中分配内存，需要分配的内存已经在类加载的过程中确定
4. 内存分配完毕之后，需要将对象中的参数初始化为零值(除了对象头中的数据)
5. 设置对象头中的数据，设置好哈希码，分代年龄，锁相关信息

执行上述步骤，可以完成对象的创建和初始化过程，即Class对象`init<>`还没有执行的逻辑。之后在构造器的初始化则是用户实现的部分。



##### 多线程情境下，分配对象内存安全性的考虑

进行分配内存的时候，两个对象同时分配在同一个内存区域，就会造成分配的内存区域重叠的问题。所以必须要保证这两个对象内存分配的串行语义，所以可以通过CAS的方式保证先分配的对象对后分配对象的可见性，这样就不会内存分配重叠。

另一种方式就是将内存区域划分为若干个不重叠的区域，每个线程分配的时候选择一个没有使用的区域即可，这样就不会重叠了。



#### 对象的布局

对象主要由对象头(MarkWord),主体数据, 和对齐留白组成.

对象头中主要包括对象的hash码，分代年龄，偏向锁信息，锁类型信息等信息组成。数组对象需要额外指定4字节的数组长度标识。同时会设置指向其元数据的指针，标记当前对象所属类的信息。

对其留白是保证32位/64位处理器能够按照8位为单位的寻址方式，补齐不满8位的数据部分。



#### 对象的访问方式

1. 使用句柄间接访问

   在栈帧中维护指向堆中的地址，对应堆地址为句柄池的地址，从句柄池中寻找对象的实际地址，相当于是间接寻址。

2. 直接访问

   栈帧中之间维护对象在堆中的地址，属于直接寻址，需要在栈帧中维护的信息比间接寻址多。

