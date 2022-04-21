#### 在实现之前需要了解的东西

1. 什么是LRU替换算法

   给定一个按照时间戳降序排序的页框链表，获取最后一个页框号，将其页框号对应的资源释放

   所以在设计的时候需要引入两个负责的成员变量

   ```c++
   std::list<frame_id_t> pages_;
   std::unordered_map<frame_id_t, frame_id_t> location_;
   ```

2. Pin/Unpin的概念

   对应缓冲池来说，Pin一个页框，这个页框的PinCount就会加一，在这个值不是零的时候，这个数据页不能被刷写到磁盘中。也就对缓冲池来说，这个页框必定在缓冲池中。而对于LRU来说，pin一个页框，页框必定在缓冲池中，所以一定不在LRU中，所以需要将其从LRU的页框列表中移除。详细逻辑参考下述代码。

#### 实现代码

CMU 给我们提供的 Replacer接口提供了下述三个关键的逻辑:

```c++
// 获取需要被淘汰的页框号, 输出到frame_id中
bool Victim(frame_id_t *frame_id);

// 将页框pin到缓冲池中(从LRU中移除)
void Pin(frame_id_t frame_id);

// 将页框从缓冲池中解除pin(加入到LRU中)
void Unpin(frame_id_t frame_id);
```



1. 首先实现LRU替换逻辑

   ```c++
   bool LRUReplacer::Victim(frame_id_t *frame_id) {
     std::lock_guard<std::mutex> guard(mutex_);
     if (this->pages_.empty()) {
       return false;
     }
     // if frame is found, remove it from lru cache and update page numbers
     int removal = pages_.back();
     location_.erase(removal);
     pages_.pop_back();
     *frame_id = removal;
     return true;
   }
   ```

   逻辑很简单，就是从链表末尾取出元素，并从记录表中删除掉即可。

2. Pin/Unpin 逻辑的实现

   ```cpp
   // pin to lru replacer
   void LRUReplacer::Pin(frame_id_t frame_id) {
     std::lock_guard<std::mutex> guard(mutex_);
     if (location_.count(frame_id) == 0) {
       return;
     }
     pages_.remove(frame_id);
     location_.erase(frame_id);
   }
   
   // unpin to lru replacer
   void LRUReplacer::Unpin(frame_id_t frame_id) {
     std::lock_guard<std::mutex> guard(mutex_);
     if (location_.count(frame_id) > 0) {
       return;
     }
     pages_.push_front(frame_id);
     location_[frame_id] = frame_id;
   }
   ```

   注意unpin的时候，数据页是最新的，需要防止在链表头部，防止下次被淘汰掉。

