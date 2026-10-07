## 一、脏读、不可重复读、幻读

```text
┌─────────────────────────────────────────────────────────────────────┐
│                    三种并发问题对比                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  脏读 (Dirty Read)                                                  │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ 事务A                    事务B                               │    │
│  │   │                        │                                 │    │
│  │   ├─ 修改 key1=200        │                                 │    │
│  │   │                        ├─ 读取 key1 → 200 (脏读)        │    │
│  │   ├─ 回滚 (key1恢复100)   │                                 │    │
│  │   │                        └─ 基于200的后续操作 ❌          │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  不可重复读 (Non-Repeatable Read)                                   │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ 事务A                    事务B                               │    │
│  │   │                        │                                 │    │
│  │   ├─ 读取 key1=100       │                                 │    │
│  │   │                        ├─ 修改 key1=200                │    │
│  │   │                        └─ 提交                         │    │
│  │   ├─ 再次读取 key1=200   │                                 │    │
│  │   └─ 两次结果不同 ❌      │                                 │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  幻读 (Phantom Read)                                                │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ 事务A                    事务B                               │    │
│  │   │                        │                                 │    │
│  │   ├─ 范围查询: 20-30岁     │                                 │    │
│  │   │   返回: 22, 25, 28    │                                 │    │
│  │   │                        ├─ 插入新记录: 23岁             │    │
│  │   │                        └─ 提交                         │    │
│  │   ├─ 再次范围查询: 20-30岁 │                                 │    │
│  │   │   返回: 22, 23, 25, 28│                                 │    │
│  │   └─ 记录数变化 ❌         │                                 │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```







## 二、隔离级别概览

```text
┌─────────────────────────────────────────────────────────────────┐
│                    隔离级别层次图                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  SERIALIZABLE (最高)                                            │
│       ↑                                                         │
│       │ 增加：范围锁、幻读保护                                   │
│       │                                                         │
│  REPEATABLE_READ                                                │
│       ↑                                                         │
│       │ 增加：读快照、可重复读                                   │
│       │                                                         │
│  READ_COMMITTED                                                 │
│       ↑                                                         │
│       │ 增加：提交时可见、避免脏读                               │
│       │                                                         │
│  READ_UNCOMMITTED (最低)                                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```







## 三、READ_UNCOMMITTED（读未提交）

会出现脏读，不可重复读，幻读

```cpp
// Put: 直接写入MemTable
void TranContext::put(const std::string &key, const std::string &value) {
    // 保存旧值用于回滚
    auto prev_record = engine_->get(key, 0);
    rollback_map_[key] = prev_record;
    
    // 直接写入数据库（其他事务立即可见！）
    engine_->put(key, value, tranc_id_);
}

// Get: 读取最新值（包括未提交）
std::optional<std::string> TranContext::get(const std::string &key) {
    // 参数0表示读取所有事务的最新值
    return engine_->get(key, 0);
}
```



**实例演示：脏读**

```cpp
// 初始数据
key1 = 100

// 事务A - READ_UNCOMMITTED
tranA.begin();
tranA.put("key1", 200);  // 直接写入 MemTable

// 事务B - READ_UNCOMMITTED
tranB.begin();
int val = tranB.get("key1");  // 读到 200 ❌ 脏读！
// 事务A可能回滚，但B已经读到了

tranA.abort();  // 回滚，key1恢复100
// 事务B基于200的计算全部错误
```



## 四、READ_COMMITTED（读已提交）

### 实现原理

```cpp
// Put: 暂存到事务上下文
void TranContext::put(const std::string &key, const std::string &value) {
    // 不直接写入数据库，先暂存
    temp_map_[key] = value;
    operations.push_back({PUT, key, value});
}

// Get: 只读取已提交的数据
std::optional<std::string> TranContext::get(const std::string &key) {
    // 先查自己的临时修改
    if (temp_map_.find(key) != temp_map_.end()) {
        return temp_map_[key];
    }
    
    // 从数据库读取，参数tranc_id_表示只读取已提交的数据
    // 只返回事务ID <= max_finished_tranc_id_ 的数据
    return engine_->get(key, this->tranc_id_);
}

// Commit: 提交时才真正写入
bool TranContext::commit() {
    // 冲突检测...
    
    // 将temp_map_中的所有修改写入MemTable
    for (auto &[k, v] : temp_map_) {
        engine_->put(k, v, tranc_id_);
    }
    
    // 更新最大已完成事务ID
    tranManager_->update_max_finished_tranc_id(tranc_id_);
}
```



**实例演示：避免脏读**

```cpp
// 初始数据
key1 = 100

// 事务A - READ_COMMITTED
tranA.begin();
tranA.put("key1", 200);  // 暂存到 temp_map_，未写入

// 事务B - READ_COMMITTED
tranB.begin();
int val = tranB.get("key1");  // 读到 100 ✅ 避免脏读

// 事务A提交
tranA.commit();  // 写入 MemTable

// 事务B再次读取
int val2 = tranB.get("key1");  // 读到 200 (不可重复读)
```





**实战演示: 出现了不可重复读**

```cpp
// 初始数据
key1 = 100

// 事务A（ID=2）- READ_COMMITTED
事务A.begin();
事务A.get("key1");  // 读到 100
事务A.put("key1", 200);  // 暂存到temp_map，未写入MemTable

// 事务B（ID=3）- READ_COMMITTED
事务B.begin();
事务B.get("key1");  // 读到 100 ✅ 读不到未提交数据

// 事务A提交
事务A.commit();  // 现在才将200写入MemTable
                // 更新max_finished_tranc_id_ = 2

// 事务B再次读取
事务B.get("key1");  // 读到 200 ❌ 不可重复读
```

**结果**：避免了脏读，但出现了不可重复读（事务B两次读取结果不同）



## 五、REPEATABLE_READ（可重复读）

### 实现原理

```cpp
// 添加读快照
std::unordered_map<std::string, std::optional<std::pair<std::string, uint64_t>>> 
    read_map_;  // 保存第一次读取的值

// Get: 使用读快照保证可重复读
std::optional<std::string> TranContext::get(const std::string &key) {
    // 先查自己的临时修改
    if (temp_map_.find(key) != temp_map_.end()) {
        return temp_map_[key];
    }
    
    // 检查是否已经读取过这个key
    if (read_map_.find(key) != read_map_.end()) {
        // 返回第一次读取的值
        return read_map_[key].has_value() ? 
               read_map_[key]->first : std::nullopt;
    }
    
    // 第一次读取，保存快照
    auto query = engine_->get(key, this->tranc_id_);
    read_map_[key] = query;
    return query.has_value() ? query->first : std::nullopt;
}

// Commit: 提交时检测写冲突
bool TranContext::commit() {
    // 检查每个修改的key是否有冲突
    for (auto &[k, v] : temp_map_) {
        // 检查MemTable
        auto res = memtable.get_(k, 0);
        if (res.is_valid() && res.get_tranc_id() > tranc_id_) {
            return false;  // 发现冲突，提交失败
        }
        
        // 检查SSTable
        auto sst_res = engine_->sst_get_(k, 0);
        if (sst_res.has_value() && sst_res->second > tranc_id_) {
            return false;  // 发现冲突
        }
    }
    
    // 无冲突，应用所有修改
    for (auto &[k, v] : temp_map_) {
        engine_->put(k, v, tranc_id_);
    }
    
    return true;
}
```



**实例演示：可重复读 + 写冲突检测**

```cpp
// 初始数据
key1 = 100
key2 = 200

// ========== 场景1：可重复读 ==========
// 事务A（ID=2）- REPEATABLE_READ
事务A.begin();
事务A.get("key1");  // 读到 100，保存到read_map_

// 事务B（ID=3）- READ_COMMITTED
事务B.begin();
事务B.put("key1", 300);  // 暂存
事务B.commit();  // 提交，key1变成300

// 事务A再次读取
事务A.get("key1");  // 从read_map_返回 100 ✅ 可重复读

// ========== 场景2：写冲突检测 ==========
// 事务C（ID=4）- REPEATABLE_READ
事务C.begin();
事务C.get("key2");  // 读到 200
事务C.put("key2", 400);  // 暂存

// 事务D（ID=5）- REPEATABLE_READ
事务D.begin();
事务D.put("key2", 500);  // 暂存
事务D.commit();  // 提交成功，key2=500，max_finished=5

// 事务C尝试提交
事务C.commit();  
// 冲突检测：key2的最新事务ID=5 > 事务C的ID=4
// 发现冲突！提交失败 ❌
```

**结果**：实现了可重复读，并通过写冲突检测防止了丢失更新。



**幻读：**

```cpp
// 初始数据：用户表
users: [
    {id: 1, name: "Alice", age: 25},
    {id: 2, name: "Bob", age: 28},
    {id: 3, name: "Charlie", age: 32}
]

// ========== 事务A (REPEATABLE_READ) ==========
auto tranA = lsm.begin_tran(REPEATABLE_READ);

// 第一次范围查询：年龄在 25-30 之间
auto list1 = tranA->scan("age:25", "age:30");
// list1 = [Alice(25), Bob(28)]  → 2条记录

// ========== 事务B (任何隔离级别) ==========
auto tranB = lsm.begin_tran(REPEATABLE_READ);
tranB->put("user:David", "{age:26}");  // 插入年龄26的用户
tranB->commit();  // 提交

// ========== 事务A 再次查询 ==========
auto list2 = tranA->scan("age:25", "age:30");
// list2 = [Alice(25), David(26), Bob(28)]  → 3条记录 ❌
// 幻读！结果集变了
```





## 六、SERIALIZABLE（可串行化）

### 实现原理

```cpp
// 在REPEATABLE_READ基础上增加范围锁
class TranContext {
private:
    std::set<std::string> range_locks_;  // 范围锁
    
public:
    // 范围查询时加锁
    std::vector<KeyValue> scan(const std::string& start, const std::string& end) {
        if (isolation_level == SERIALIZABLE) {
            // 对查询范围加锁
            range_locks_.insert(start);
            range_locks_.insert(end);
            
            // 在提交时检查是否有幻读
            // 如果有新数据插入到范围内，则提交失败
        }
        // 执行范围查询...
    }
    
    bool commit() {
        // 1. 先执行REPEATABLE_READ的冲突检测
        // 2. 再检查范围锁范围内的数据是否被修改
        for (auto& range : range_locks_) {
            // 检查范围内是否有新插入的数据
            if (has_new_data_in_range(range)) {
                return false;  // 检测到幻读，提交失败
            }
        }
        return true;
    }
};
```



**实例演示：防止幻读**

```cpp
// 初始数据
users表：age=25, age=28, age=32

// ========== 场景：幻读问题 ==========
// 事务A（ID=2）- REPEATABLE_READ
事务A.begin();
事务A.scan("age > 25", "age < 35");  
// 返回: [28, 32] 共2条记录

// 事务B（ID=3）- READ_COMMITTED
事务B.begin();
事务B.put("age", 30);  // 插入一个新用户 age=30
事务B.commit();  // 提交

// 事务A再次扫描
事务A.scan("age > 25", "age < 35");  
// 返回: [28, 30, 32] 共3条记录 ❌ 幻读！

// ========== SERIALIZABLE解决方案 ==========
// 事务A（ID=2）- SERIALIZABLE
事务A.begin();
事务A.scan("age > 25", "age < 35");
// 返回: [28, 32]
// 同时，对范围 (25, 35) 加范围锁 🔒

// 事务B（ID=3）- SERIALIZABLE
事务B.begin();
事务B.put("age", 30);
// 检测到范围锁冲突！等待或立即失败

// 事务A提交
事务A.commit();  // 释放范围锁

// 事务B才能继续
事务B.commit();  // 现在插入，不会产生幻读
```



## 七、四种隔离级别对比总结

### 5.1 并发问题解决情况

| 隔离级别         | 脏读   | 不可重复读 | 幻读   | 丢失更新 |
| :--------------- | :----- | :--------- | :----- | :------- |
| READ_UNCOMMITTED | 可能 ❌ | 可能 ❌     | 可能 ❌ | 可能 ❌   |
| READ_COMMITTED   | 解决 ✅ | 可能 ❌     | 可能 ❌ | 可能 ❌   |
| REPEATABLE_READ  | 解决 ✅ | 解决 ✅     | 可能 ❌ | 解决 ✅   |
| SERIALIZABLE     | 解决 ✅ | 解决 ✅     | 解决 ✅ | 解决 ✅   |



### 5.2 实现机制对比

```cpp
// READ_UNCOMMITTED
put: 直接写MemTable
get: 读最新值（所有事务可见）
冲突检测: 无

// READ_COMMITTED  
put: 暂存temp_map
get: 读已提交数据（ID <= max_finished）
冲突检测: 提交时检测写冲突

// REPEATABLE_READ
put: 暂存temp_map
get: 读快照（第一次读的值保存到read_map）
冲突检测: 提交时检测写冲突 + 读快照

// SERIALIZABLE
put: 暂存temp_map
get: 读快照 + 范围锁
冲突检测: 写冲突 + 范围冲突 + 幻读检测
```





### 5.3 性能对比

```cpp
// 并发度对比
READ_UNCOMMITTED:  最高并发
    ↓ 性能下降
READ_COMMITTED:    高并发（读不加锁）
    ↓ 性能下降
REPEATABLE_READ:   中等并发（写冲突检测）
    ↓ 性能下降
SERIALIZABLE:      最低并发（范围锁）

// 实际场景选择
- 分析系统、报表：READ_UNCOMMITTED（追求速度）
- Web应用：READ_COMMITTED（平衡性能和一致性）
- 金融系统：SERIALIZABLE（数据一致性最重要）
- LSM-Tree默认：REPEATABLE_READ（大多数场景）
```





## 八、实际代码中的体现

```cpp
// 在事务管理器中的使用
int main() {
    LSM lsm("example_data");
    
    // READ_UNCOMMITTED - 最快，但可能脏读
    auto tran1 = lsm.begin_tran(READ_UNCOMMITTED);
    tran1->put("key1", "value1");  // 立即生效
    tran1->commit();  // 简单提交
    
    // READ_COMMITTED - 避免脏读
    auto tran2 = lsm.begin_tran(READ_COMMITTED);
    tran2->put("key2", "value2");  // 暂存
    tran2->commit();  // 提交时才写入
    
    // REPEATABLE_READ - 可重复读
    auto tran3 = lsm.begin_tran(REPEATABLE_READ);
    tran3->get("key3");  // 第一次读取，保存快照
    // ... 其他事务修改
    tran3->get("key3");  // 返回第一次的值
    tran3->commit();  // 冲突检测
    
    // SERIALIZABLE - 最严格
    auto tran4 = lsm.begin_tran(SERIALIZABLE);
    tran4->scan("a", "z");  // 范围查询，加范围锁
    // ... 防止幻读
    tran4->commit();  // 释放锁
}
```

**总结**：四种隔离级别在"一致性"和"性能"之间做了不同的权衡，从低到高逐步增加保护机制，但也逐步降低并发性能。选择哪个级别取决于具体业务场景对数据一致性的要求。





```
LSM 存储引擎项目

项目描述
基于 LSM-Tree 架构的高性能键值存储引擎，完整实现 MemTable 到 SSTable 的分层存储模型，支持事务处理、崩溃恢复及 Redis 协议兼容，适用于高吞吐写入场景。

技术栈：C++17 + 多线程 + 文件系统 + Redis RESP 协议

我的职责
负责存储引擎核心架构设计与性能优化，解决并发控制、数据持久化等关键技术问题。

· 分层存储架构：设计 MemTable（SkipList）+ 多层 SSTable 存储模型，实现自动 Compaction 机制，打通写入/查询/压缩全链路；

· WAL 与崩溃恢复：实现预写日志机制，系统启动时自动 replay 恢复已提交数据，配合 checkpoint 策略控制日志增长，确保数据持久性；

· MVCC 事务处理：基于多版本并发控制实现快照读取，在各级迭代器中嵌入事务 ID 过滤逻辑，支持一致性读和不可见版本过滤；

· 布隆过滤器优化：集成布隆过滤器减少无效磁盘读取，查询性能提升 40%；

· Redis 协议兼容：完整实现 RESP 协议，支持 SET/GET/INCR/SADD/HSET/ZADD 等常用命令，与 redis-cli 无缝交互；

· 性能测试：使用 redis-benchmark（-c 100 -n 100000）压测，SET 达到 142k QPS / 0.50ms，GET 达到 134k QPS / 0.52ms，其他操作均在 120k+ QPS，延迟低于 0.62ms；

· 跨平台构建：使用 xmake 实现一键编译与启动，支持 Linux/macOS/Windows。
```





```
ssh baeon@ssh_ip -p 127.0.0.1
```



```bash

```









