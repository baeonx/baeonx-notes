# 1 跳表的实现

## 1.1 跳表的random_level

```cpp
// TODO: 如果插入一个元素要决定元素最终插入多少层

int SkipList::random_level() {
   // <span style="color: #6A737D;">? 通过"抛硬币"的方式随机生成层数：</span>
  // ? 通过"抛硬币"的方式随机生成层数：                                   
  // ? - 每次有50%的概率增加一层
  // ? - 确保层数分布为：第1层100%，第2层50%，第3层25%，以此类推
  // ? - 层数范围限制在[1, max_level]之间，避免浪费内存
  // TODO: Lab1.1 任务：插入时随机为这一次操作确定其最高连接的链表层数
  int level = 1;
  while(rand()% 2 == 0)
  {
    level ++;
  }
  if(level >= max_level) return max_level;
  return level;
}
```





## 1.2 跳表的put

```cpp

// 插入或更新键值对
void SkipList::put(const std::string &key, const std::string &value,
                   uint64_t tranc_id) {
  spdlog::trace("SkipList--put({}, {}, {})", key, value, tranc_id);

  // TODO: Lab1.1  任务：实现插入或更新键值对
  // ? Hint: 你需要保证不同`Level`的步长从底层到高层逐渐增加
  // ? 你可能需要使用到`random_level`函数以确定层数, 其注释中为你提供一种思路
  // ? tranc_id 为事务id, 现在你不需要关注它, 直接将其传递到 SkipListNode 的构造函数中即可
  std::shared_ptr<SkipListNode> current = head;
  std::vector<std::shared_ptr<SkipListNode>> update(max_level, nullptr);
  for(int i = current_level - 1; i >= 0; i --)
  {
    while(current->forward_[i] != nullptr && current->forward_[i]->key_ < key)
    {
      current = current->forward_[i];
    }
    update[i] = current;
  }

  current = current->forward_[0];

  // 检查key是否存在，如果存在则更新value
  if (current != nullptr && current->key_ == key) {
      // 更新值，调整大小
      size_bytes -= current->value_.size();  // 减去旧值大小
      current->value_ = value;
      size_bytes += value.size();             // 加上新值大小
      spdlog::trace("SkipList--put({}, {}) updated, size_bytes={}", key, value, size_bytes);
      return;
  }

  int newLevel = random_level();
  if(newLevel > current_level)
  {
    // 如果该元素要插入的层数，高于原来的最高处，那么高出的层数的update都是head
    // 例如 新元素4要插入4层则，高出的一层的update也就是前一个元素是head
    // head --> 1
    // head --> 1 --> 5
    // head --> 1---> 5 --> 7 
    for(int i = current_level; i < newLevel; i ++)
    {
      update[i] = head;
    }
    current_level = newLevel;
  }

  // 创建一个新节点然后依次插入
  auto new_node = std::make_shared<SkipListNode>(key, value, newLevel, tranc_id);

  // 实现双向链表
  for(int i = 0; i < newLevel; i ++)
  {
    auto next_node = update[i]->forward_[i];
    new_node->forward_[i] = next_node;
    // 需要判断下一个指针是否是空指针
    if(new_node!=nullptr) new_node->set_backward(i, new_node);
    update[i]->forward_[i] = new_node;
    new_node->set_backward(i, update[i]);
  }

  // 计算新增的内存
  size_t node_size = key.size() + value.size() + 8;
  size_bytes += node_size;
  
  spdlog::trace("SkipList--put({}, {}) inserted, added {} bytes, total size_bytes={}", key, value, node_size, size_bytes);
}

```





## 1.3 跳表的get

```cpp
// 查找键值对
SkipListIterator SkipList::get(const std::string &key, uint64_t tranc_id) {
  // spdlog::trace("SkipList--get({}) called", key);
  // ? 你可以参照上面的注释完成日志输出以便于调试
  // ? 日志为输出到你执行二进制所在目录下的log文件夹

  // TODO: Lab1.1 任务：实现查找键值对,
  // TODO: 并且你后续需要额外实现SkipListIterator中的TODO部分(Lab1.2)
  std::shared_ptr<SkipListNode> current = head;
  if (!current) return SkipListIterator{};
  for(int i = current_level - 1; i >= 0; i --)
  {
    while(current->forward_[i] != nullptr && current->forward_[i]->key_ < key)
    {
      current = current->forward_[i];
    }
  }

  current = current->forward_[0];

  if(current != nullptr && current->key_ == key)
  {
    spdlog::trace("SkipList--get({}) found", key);
    return SkipListIterator{current};
  }

  spdlog::trace("SkipList--get({}) not found", key);
  return SkipListIterator{};
}

```





## 1.4 跳表的remove

```cpp
// 删除键值对
// ! 这里的 remove 是跳表本身真实的 remove,  lsm 应该使用 put 空值表示删除,
// ! 这里只是为了实现完整的 SkipList 不会真正被上层调用
void SkipList::remove(const std::string &key) {
  // TODO: Lab1.1 任务：实现删除键值对
  std::shared_ptr<SkipListNode> current = head;
  std::vector<std::shared_ptr<SkipListNode>> update(max_level, nullptr);
  for(int i = current_level - 1; i >= 0; i --)
  {
    while(current->forward_[i] != nullptr && current->forward_[i]->key_ < key)
    {
      current = current->forward_[i];
    }
    update[i] = current;
  }

  current = current->forward_[0];

  if(current != nullptr && current->key_ == key)
  {
    // 计算要减去的字节数
    size_t removed_size = current->key_.size() + current->value_.size() + 8;


    // 获取节点的实际层数
    int nodeLevel = current->forward_.size();
        
    // 只更新到节点的实际层数
    for(int i = 0; i < nodeLevel; i++)
    {

      update[i]->forward_[i] = current->forward_[i];
      if(update[i]->forward_[i] != nullptr) update[i]->forward_[i]->set_backward(i, update[i]);
    }
    // 更新当前最大层数
    while(current_level > 0 && head->forward_[current_level - 1] == nullptr) {
      current_level--;
    }
      
    // 减少元素计数
    // 减少内存计数
    size_bytes -= removed_size;
    
    spdlog::trace("SkipList--remove({}) success, removed {} bytes, size_bytes={}", key, removed_size, size_bytes);
  } else {
    spdlog::trace("SkipList--remove({}) failed: key not found", key);
  }

}
```









## 2 迭代器

 迭代器是对skiplistnode指针的封装，能够实现对*skiplistnode的各种操作

```cpp

BaseIterator &SkipListIterator::operator++() {
  // TODO: Lab1.2 任务：实现SkipListIterator的++操作符
  // 移动元素
  // 如果当前指针不为空，那么就往下移动一位
  if (current) {
    current = current->forward_[0];
  }
  return *this;
}

bool SkipListIterator::operator==(const BaseIterator &other) const {
  // TODO: Lab1.2 任务：实现SkipListIterator的==操作符
  // 1. 先检查类型是否相同，这里抽象出来了一个BaseIterator类，所以需要检查
  if (this->get_type() != other.get_type()) {
      return false;  // 不同类型肯定不相等
  }

  // 2.这个时候BaseIterator类肯定是IteratorType::SkipListIterator类了，将other实现向下转型
  const SkipListIterator* otherIter = dynamic_cast<const SkipListIterator*>(&other);

  // 3.向下转型后判断是否为空指针，双保险
  if (otherIter == nullptr) {
    return false;
  }
  // 4.看看是不是同一个skiplistnode
  if(current != otherIter->current)
  {
    return false;
  }
  
  return true;
}

bool SkipListIterator::operator!=(const BaseIterator &other) const {
  // TODO: Lab1.2 任务：实现SkipListIterator的!=操作符
  // 直接调用==再取反
  return !(*this == other);
}

SkipListIterator::value_type SkipListIterator::operator*() const {
  // TODO: Lab1.2 任务：实现SkipListIterator的*操作符
  // 这个操作也就是对skipListIterator解引用，拿到里面的元素，而里面的元素就是key,value，所以可以看到这里的返回值是value_type
  if (!current) 
  {
    return {"", ""};
  }
  return {current->key_, current->value_};
}

IteratorType SkipListIterator::get_type() const {
  // TODO: Lab1.2 任务：实现SkipListIterator的get_type
  // ? 主要是为了熟悉基类的定义和继承关系
  return IteratorType::SkipListIterator;
}
```

