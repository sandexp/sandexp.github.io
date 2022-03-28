#### TheadLocal简介

线程本地变量: 每个线程都有同一个变量的独有拷贝。

对于同一个变量local，每个线程都维护一个自己独立的值，这就是线程本地变量的含义。

#### 常用API

+ initialValue: 设置默认值，如果get不到key对应的value，则返回默认值
+ remove: 移除key所对应的entry

ThreadLocal 经常用于提供线程上下文信息。

每个线程都有一个Map，对于每个ThreadLocal对象，调用其get/set实际上就是以ThreadLocal对象为键读写当前线程的Map

