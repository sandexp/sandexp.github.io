#### Netty内存管理

为了提高内存的使用效率，Netty引入了`jemalloc`内存分配算法。内存区域包含三个部分，分别是本地线程缓存、分配区arena、系统内存。

总体的内存分配策略如下:

1. 为了避免线程间锁的竞争和同步，每个I/O线程都对应一个PoolThreadCache，负责当前线程使用非大内存的快速申请和释放。
2. 当从PoolThreadCache中获取不到内存时，就从PoolArena的内存池中分配。当内存使用完并释放时，会将其放到PoolThreadCache中，方便下次使用
3. 若从PoolArena的内存池中分配不到内存，则从堆内外内存中申请，申请到的内存叫PoolChunk
4. 当申请大内存时（超过了PoolChunk的默认内存大小12 MB），直接在堆外或堆内内存中创建（不归PoolArea管理），用完后直接回收

#### PoolChunk内存分配

##### 大于8 k内存的分配

Netty底层的内存分配和管理主要由PoolChunk实现，可以把Chunk看作一块大的内存，这块内存被分成了很多小块的内存，Netty在使用内存时，会用到其中一块或多块小内存。

内存池在分配内存时，只会预先准备固定大小和数量的内存块，不会请求多少内存就恰好给多少内存，因此会有一定的内存被浪费。使用完后交还给PoolChunk并还原，以便重复使用。

PoolChunk内部维护了一棵平衡二叉树，默认由2048个page组成，一个page默认为8 KB，整个Chunk默认为16 MB. 在

PoolChunk中，用一个数组`memoryMap`维护了所有节点及其对应的高度值. 另外，还有一个同样的数组---`depthMap`。两者的区别是：`depthMap`一直不会改变，通过`depthMap`可以获取节点的内存大小，还可以获取节点的初始高度值.

##### 小于8 k内存的分配

PoolChunk有一个PoolSubpage[]组数，叫`subpages`，它的数组长度与PoolChunk的page节点数一致，都是2048.

PoolChunk分配PoolSubpage的步骤如下：

1. 在PoolChunk的二叉树上找到匹配的节点，由于小于8 KB，因此只匹配page节点。因为page节点从2048开始，所以page将节点与2048进行异或操作就可以得到PoolSubpage[]数组`subpages`的下标`subpageIdx`
2. 取出`subpages[subpageIdx]`的值。若值为空，则新建一个PoolSubpage并初始化。
3. PoolSubpage根据内存大小分两种，即小于512 B为tiny，大于或等于512 B且小于8 KB为small，此时PoolSubpage还未区分是small还是tiny。然后把PoolSubpage加入PoolArena的PoolSubpage缓存池中，以便后续直接从缓存池中获取。
4. PoolArena的PoolSubpage[]数组缓存池有两种，分别是存储(0,512)个字节的`tinySubpagePools`和存储[512,8192)个字节的`smallSubpagePools`。由于所有内存分配大小`elemSize`都会经过`normalizeCapacity`的处理。当`elemSize`<512时，`elemSize`从16开始，每次加16 B.当`elemSize`>=512时，`elemSize`成倍增长.

#### PoolSubpage内存分配与释放

PoolSubpage是由PoolChunk的page生成的，page可以生成多种PoolSubpage，但一个page只能生成其中一种PoolSubpage。PoolSubpage可以分为很多段，每段的大小相同，且由申请的内存大小决定。

由于PoolSubpage每段的最小值为16 B，因此它的段的总数量最多为`pageSize`/16。把PoolSubpage中每段的内存使用情况用一个long[]数组来标识，long类型的存储位数最大为64 B，每一位用0表示为空闲状态，用1表示被占用，这个数组的长度为`pageSize`/16/64 B.

##### 内存分配过程

1. 当内存空间在PoolSubpage中分配成功后，可以得到一个指针handle。
2. 通过handler可以计算page的偏移量，也可以计算subpage的段在page中的相对偏移量，两者加起来就是该段分配的内存在chunk中的相对位置偏移量
3. 指针handle的高32位存储PoolSubpage中分配的内存段在page中的相对偏移量；低32位存储page在PoolChunk的二叉树中的位置`memoryMapIdx`
4. 通过handler去寻找对于`SubPage`所处的内存位置，进行访问.

##### 内存释放过程

1. 若在PoolSubpage上的偏移量大于0，则交给PoolSubpage去释放, 根据PoolSubpage内存分配段的偏移位`bitmapIdx`找到long[]数组bitmap的索引q，将bitmap[q]的具体内存占用位r置为0。同时调整Arena中的PoolSubpage缓存池，若PoolSubpage已全部释放了，且池中除了它还有其他节点，则从池中移除。
2. 若在PoolSubpage上的偏移量等于0，或者PoolSubpage释放完后返回false,则只需更新PoolChunk二叉树对应节点的高度值，并更新其所有父节点的高度值及可用字节数即可。

#### PoolArena内存分配

##### 内存分配

Netty的I/O线程在读取Channel数据时，需要内存分配器`PooledByteBufAllocator`来分配内存，而最终具体的分配工作是由PoolArena完成的，PoolArena是内存的管理入口。

由于Netty应用服务是多线程高并发系统，所以为了减少多线程同时分配同一块内存的竞争、提高内存的分配效率，在默认情况下会创建多个PoolArena并放入`PoolThreadLocalCache`中.

`PoolChunkList`是PoolChunk链表，内存在使用过程中会被重复利用。每次在分配内存时不会都去创建一个PoolChunk，而是优先选择之前的PoolChunk内存块进行分配，若创建了多个PoolChunk，则要考虑如何管理这些PoolChunk，主要考虑内存能快速得到分配，而且不能过于浪费。

##### 内存释放

PoolArena除了内存分配，还管理内存的释放，PoolThreadCache中有多个`MemoryRegionCache`数组，每种类型的内存都有一个`MemoryRegionCache`数组与之对应。`MemoryRegionCache`中有个队列，这个队列主要是用来存放内存对象的。