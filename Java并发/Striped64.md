JDK 8 开始, Java开始支持了对Long,Double类型的原子操作.

 分别是`LongAddr,LongAccumulator,DoubleAddr,DoubleAccumulator`. 这些类都是继承与`Striped64`类. 是对64位数据类型的原子操作的扩展。

#### LongAddr

AtomicLong内部是一个volatile long型变量，由多个线程对这个变量进行CAS操作。

> 多个线程同时对一个变量进行CAS操作，在高并发的场景下仍不够快，如果再要提高性能，该怎么做呢？

把一个变量拆成多份，变为多个变量，有些类似于ConcurrentHashMap 的分段锁的例子。

把一个Long型拆成一个base变量外加多个Cell, 每个Cell包装了一个Long型变量. 当多个线程并发累加的时候, 如果并发度低, 就直接加到base变量上; 如果并发度高, 冲突大, 平摊到这些Cell上. 在最后取值的时候, 再把base和这些Cell求sum运算.

由于无论是long，还是double，都是64位的。但因为没有double型的CAS操作，所以是通过把double型转化成long型来实现的。

> 上面的base和cell[]变量，是位于基类Striped64当中的。英文Striped意为“条带”，也就是分片。

##### 最终一致性问题

在sum求和函数中，并没有对cells[]数组加锁。也就是说，一边有线程对其执行求和操作，一边还有线程修改数组里的值，也就是最终一致性，而不是强一致性。它适合高并发的统计场景，而不适合要对某个Long 型变量进行严格同步的场景。

##### 伪共享与缓存行填充

在Cell类的定义中，用了一个独特的注解`@sun.misc.Contended`，这是JDK 8之后才有的，背后涉及一个很重要的优化原理：伪共享与缓存行填充。

缓存与主内存进行数据交换的基本单位叫Cache Line（缓存行）.

> Eg: 主内存中有变量X、Y、Z（假设每个变量都是一个Long型），被CPU1和CPU2分别读入自己的缓存，放在了同一行Cache Line里面。
>
> 当CPU1修改了X变量，它要失效整行Cache Line，也就是往总线上发消息，通知CPU 2对应的Cache Line失效。由于Cache Line是数据交换的基本单位，无法只失效X，要失效就会失效整行的Cache Line，这会导致Y、Z变量的缓存也失效。
>
> 虽然只修改了X变量，本应该只失效X变量的缓存，但Y、Z变量也随之失效。Y、Z变量的数据没有修改，本应该很好地被CPU1和CPU2共享，却没做到，这就是所谓的“伪共享问题”。

解决方案:

Y、Z和X变量处在了同一行Cache Line里面。要解决这个问题，需要用到所谓的“缓存行填充”，分别在X、Y、Z后面加上7个无用的Long型，填充整个缓存行，让X、Y、Z处在三行不同的缓存行中.

##### LongAddr的实现

LongAdder最核心的累加函数add（long x），自增、自减操作都是通过调用该函数实现的。

**执行过程**: 当一个线程调用add（x）的时候，首先会尝试使用casBase把x加到base变量上。如果不成功，则再用a.cas（..）函数尝试把x 加到Cell数组的某个元素上。如果还不成功，最后再调用longAccumulate（..）函数。

#### LongAccumulator

LongAccumulator的原理和LongAdder类似，只是功能更强大. LongAdder只能进行累加操作, 并且初始值默认为0；LongAccumulator可以自己定义一个二元操作符, 并且可以传入一个初始值. 

#### DoubleAddr和DoubleAccumulator

DoubleAdder 其实也是用long 型实现的，因为没有double类型的CAS 函数。下面是DoubleAdder的add（x）函数，和LongAdder的add（x）函数基本一样，只是多了long和double类型的相互转换。