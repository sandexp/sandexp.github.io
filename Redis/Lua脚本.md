### Lua脚本

Redis 2.6.0开始官方提供了内置的Lua解释器, 可以对Lua脚本求值. 主要包括`eval`和`evalsha`两个指令.



##### Redis执行Lua命令

`eval`第一个参数是一个lua脚本程序，运行在redis服务器中. 

第二个参数表示参数的个数, 后面列举出参数列表信息, 例如:

```bash
> eval "return {KEYS[1],KEYS[2],ARGV[1],ARGV[2]}" 2 key1 key2 first second
```



##### Lua执行Redis命令

Lua脚本通过call(), pcall()执行redis, 这里个函数不同点在于对错误处理的不同. call()会直接上报错误, 而pcall()将错误按照Lua表的形式返回.



##### Lua 数据类型和Redis数据类型的转换

1. 当 Lua 通过 call() 或 pcall() 函数执行 Redis 命令的时候，命令的返回值会被转换成 Lua 数据结构。
2. 当 Lua 脚本在 Redis 内置的解释器里运行时，Lua 脚本的返回值也会被转换成 Redis 协议(protocol)，然后由 EVAL 将值返回给客户端。

所以来说Lua数据类型和Redis数据类型是一一对应的, 具体情况这里不做详述.



##### Lua脚本具有原子性

Redis 使用单个 Lua 解释器去运行所有脚本，并且， Redis 也保证脚本会以原子性(atomic)的方式执行： 当某个脚本正在运行的时候，不会有其他脚本或 Redis 命令被执行。

在其他别的客户端看来，脚本的效果要么是不可见的，要么就是已完成的。



##### 带宽和EVALSHA

`EVAL` 命令要求你在每次执行脚本的时候都发送一次脚本主体。**Redis 有一个内部的缓存机制，因此它不会每次都重新编译脚本**，不过在很多场合，付出无谓的带宽来传送脚本主体并不是最佳选择。



为了减少带宽的消耗， Redis 实现了 [EVALSHA](http://www.redis.cn/commands/evalsha.html) 命令，它的作用和 `EVAL` 一样，都用于对脚本求值，但它接受的第一个参数不是脚本，而是脚本的 SHA1 校验和(用于缓存标识).



##### 脚本缓存

Redis 保证所有被运行过的脚本都会被永久保存在脚本缓存当中，这意味着，当 `EVAL` 命令在一个 Redis 实例上成功执行某个脚本之后，随后针对这个脚本的所有 EVALSHA命令都会成功执行。

刷新脚本缓存的唯一办法是显式地调用 SCRIPT FLUSH 命令，这个命令会清空运行过的所有脚本的缓存.