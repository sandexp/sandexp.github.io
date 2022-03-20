在处理集合时，我们通常会迭代遍历它的元素，并在每个元素上执行某项操作。这种情况下，通常需要使用循环逻辑控制迭代器的行为，而流操作为为我们屏蔽了底层循环的实现，专注于循环逻辑的处理。例如:

```java
contents.stream().map(content -> { System.out.println(content);} );
```

如果在最后添加`parallerStream`，还可以将其转换为并行流进行处理，即多个线程同时处理，例如:

```java
long count=words.parrallerStream().filter(w -> w.length()>12).count();
```

总的来说，流的操作具有如下特征:

1. 流并不存储其元素。这些元素可能存储在底层的集合中，或者是按需生成的。
2. 流的操作不会修改其数据源。
3. 流的操作是尽可能惰性执行的。这意味着直至需要其结果时，操作才会执行。

#### 流的基础API

1. 流派生算子

   流的转换会产生一个新的流，它的元素派生自另一个流中的元素。我们已经看到了filter转换会产生一个流，它的元素与某种条件相匹配。

   常用的流派生算子有`filter`,`map`,`flatMap`

2. 抽取子流和连接流

   调用`stream.limit（n）`会返回一个新的流，它在n个元素之后结束（如果原来的流更短，那么就会在流结束时结束）。这个方法对于裁剪无限流的尺寸会显得特别有用。

   调用`stream.skip（n）`正好相反：它会丢弃前n个元素。

3. 其他转换类型

   distinct方法会返回一个流，它的元素是从原有流中产生的，即原来的元素按照同样的顺序剔除重复元素后产生的。

   对于流的排序，有多种sorted方法的变体可用。其中一种用于操作Comparable元素的流，而另一种可以接受一个Comparator。

#### Optional

Optional<T>对象是一种包装器对象，要么包装了类型T的对象，要么没有包装任何对象。Optional<T>类型被当作一种更安全的方式，用来替代类型T的引用，这种引用要么引用某个对象，要么为null。

在获取不到T的实体元素的时候，会执行默认回调函数。

```java
T orElseGet(Supplier<？extends T>other);

T orElseThrow(Supplier<？extends X>exceptionSupplier);

void ifPresent(Consumer<？super T>consumer);
```

##### 创建方式

`Optional.of(result)`和`Optional.empty()`

