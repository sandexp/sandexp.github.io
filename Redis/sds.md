### SDS

`SDS`是`Redis`对于动态字符串的一种实现, 说起动态扩展且二进制安全的字符串类型.

**动态扩展: **

底层`API`会处理好字符串动态扩展的逻辑, 对于上层是透明的.

**二进制安全:**

假设存在一个分隔符, 用于分割字符串(C语言中的`\0`字符), 如果字符串本身就包含这个字符, 那么这个字符串就会被阶段, 这里就是二进制不安全的一种典型案例.

#### 动态扩展的实现

如何设计`SDS`的结构体类型, 从而支持动态扩展, 就是关键的一步.

最简单的, 如果需要无限的扩展, 直接向操作系统申请一块无限大小的数组, 用于存储`char`类型即可, 显然, 这种做法是行不通的. 所以每次只能像操作系统申请指定大小的内存空间. 

同时, 为了降低每次分配内存的开销, 不应当在到达空间最大值的时候进行分配, 而是进行到中间某个位置的时候进行分配. 所以这里需要设计一个位置指针, 用于指示当前字符串的长度. 可以使用如下方式进行描述:

```c++
struct sds{
    // buf已经占用的字节数, 也可以描述为位置指针
    int len;
    
    // buf中剩余可用的字节数
    int free;
    
    // 数据存储空间, 其长度为当前sds的存储上限
    char buf[];
}
```

从上面的描述中可以看出, 使用位置指针`len`可以规避`\0`终止符对字符串的截断问题, 保证了二进制安全.

#### `sds`对于短字符串存储优化

基于上述结构体描述, 实际上描述一个动态字符串需要8个字节的长度, 在某些场景下显得有些太大. 对于一些长度短的字符串, 显然可以缩短头部的大小, 进而压缩存储大小.

对于存储长度小于32位的字符串, `Redis 5`做出了如下的优化:

```c++
struct __attribute__ ((__packed__)) sdshdr5 {
    // 低三位表示存储类型, 高5位表示存储长度
    unsigned char flags;
   
    // 数据存储空间
    char buf[];
}
```

对于不同长度的字符串, `Redis 5.0` 划分成了不同种类粒度, 包括`sdshdr8`,`sdshdr16`,`sdshdr32`,`sdshdr64`.  详细设计参考附录, 设计原理与`sdshdr5`类似.

#### 常用`API`

##### 字符串的创建

`Redis`通过`sdsnewlen`函数创建`SDS`. 函数中会通过字符串的长度 选择 合适的存储类型. 初始化完成之后, 返回指向字符串类型的指针. (指向的是`buf`的首地址)

##### 字符串的释放

`sds`提供了直接释放内存的方法`sdsfree`. 首先通过`s`的偏移, 定位到`sds`结构体的首部, 然后调用`s_free`释放内存.

```c++
void sdsfree(sds s){
    if(s==NULL) return;
    s_free((char *)s-sdsHdrSize(s[-1]));// 在这里释放内存
}
```

为了降低频繁分配和释放内存的开销，`sds`提供了不直接释放内存的方法, 通过重置统计信息的方式达到清空的目的(`sdsclear`)。该方法将`sds`的`len`清零, 此处存在的`buf`没有被释放, 可以被重新写入.

```c++
void sdsclear(sds s){
    sdssetlen(s,0);// 重置统计值
    s[0]='\0';// 从内容上清空buf
}
```

##### 字符串的拼接

拼接字符串的方式, 各类语言都提供了对于它的实现, 但是`redis 5.0`中设计了多种字符串的种类, 此外, 两个大字符串的拼接. 都对拼接完成的字符串的容量有所要求.

`Redis`中调用的事情是`sdscatlen`. 调用`sdsMakeRoomFor`方法进行容量检测, 如果容量检测没有问题则直接返回s, 否则则进行扩容操作.

**扩容策略**

1. 如果`sds`中剩余空间可以容纳拼接的字符串, 则直接向`buf`数组追加即可, 不需要扩容.
2. 假设新的总长度`newLen=len+addlen`，如果这个长度小于`1MB`，那么容量为当前容量的2倍, 否则每次增加`1MB`.



#### 附录

1.  不同类型`sds`设计

   ```c++
   struct __attribute__ ((__packed__)) sdshdr8 {
   	
       uint8_t len;
       
       uint8_t alloc;
       
       unsigned char flags;
      
       // 数据存储空间
       char buf[];
   }
   
   struct __attribute__ ((__packed__)) sdshdr16 {
   	
       uint16_t len;
       
       uint16_t alloc;
       
       unsigned char flags;
      
       // 数据存储空间
       char buf[];
   }
   
   struct __attribute__ ((__packed__)) sdshdr32 {
   	
       uint32_t len;
       
       uint32_t alloc;
       
       unsigned char flags;
      
       // 数据存储空间
       char buf[];
   }
   
   struct __attribute__ ((__packed__)) sdshdr64 {
   	
       uint64_t len;
       
       uint64_t alloc;
       
       unsigned char flags;
      
       // 数据存储空间
       char buf[];
   }
   ```

   其中`len`表示`buf`中的位置指针, `alloc`表示`buf`分配的总长度, `flags`表示当前结构体类型.