

# 1 数的范围

## 事务ID的含义

在MVCC（多版本并发控制）中：

- **tranc_id（事务ID）**：表示创建这个版本的事务的ID
- **事务ID是递增的**：事务ID越大，表示事务开始得越晚

`curr_tranc_id <= tranc_id` 的含义

这个条件的意思是：**找到一个版本，它的事务ID小于等于目标事务ID**





```cpp
class Solution {
	public int minArray(int[] numbers) {
        int min = numbers[0];
        for(int i =l; i < numbers.length; i++)
        {
            if(min > numbers[i]) min = numbers[i];
            return min;
        }
}
```



```cpp
int l = 0, r = n - 1;
while (l < r) {
    int mid = l + r >> 1;
    if (a[mid] < x)  l = mid + 1;
    else    r = mid;
}
```

```
1 2 2 3 4
// 先用二分查找找到key的范围
  int left = 0, right = offsets.size() - 1;
  int first_key_pos = -1;
  
  // 找到第一个等于key的位置
  while (left <= right) {
    int mid = left + (right - left) / 2;
    std::string mid_key = get_key_at(offsets[mid]);
    
    int cmp = key.compare(mid_key);
    if (cmp == 0) {
      first_key_pos = mid;
      right = mid - 1;  // 继续向左找第一个
    } else if (cmp < 0) {
      right = mid - 1;
    } else {
      left = mid + 1;
    }
  }
  
  if (first_key_pos == -1) {
    return std::nullopt;
  }
  
  // 找到最后一个等于key的位置
  int last_key_pos = first_key_pos;
  while (last_key_pos + 1 < static_cast<int>(offsets.size())) {
    std::string next_key = get_key_at(offsets[last_key_pos + 1]);
    if (next_key == key) {
      last_key_pos++;
    } else {
      break;
    }
  }
  
  // 数据按事务ID降序排列（最新的在前）
  // 索引: 0(tranc_id=3), 1(tranc_id=2), 2(tranc_id=1)
  // 需要找到第一个事务ID <= tranc_id的记录
  int result_pos = -1;
  for (int i = first_key_pos; i <= last_key_pos; i++) {
    uint64_t curr_tranc_id = get_tranc_id_at(offsets[i]);
    if (curr_tranc_id <= tranc_id) {
      result_pos = i;
      break;  // 找到第一个就返回
    }
  }
  
  if (result_pos != -1) {
    return result_pos;
  }
  
  // 如果没有找到 <= tranc_id 的记录（说明tranc_id比所有记录都小）
  // 返回最新的记录（第一个）
  return first_key_pos;
```





```cpp
if (offsets.empty()) {
    return std::nullopt;
  }
  
  int l = -1, r = offsets.size(), mid;
  // 例如，这里是 1，3，3，5
  // 我们得找到第一个3，那么第一个3是满足 左区间 < 3，右区间 >= 3
  while(l + 1 != r)
  {
    mid = (l + r) >> 1;
    std::string key_mid = get_key_at(offsets[mid]);
    if(key_mid < key)
    {
      l = mid;
    }
    else{
      r = mid;
    }
  }
  
  int first_pos = r;
  if(first_pos == offsets.size() || get_key_at(offsets[first_pos]) != key)
  {
    return std::nullopt;
  }

  l = -1, r = offsets.size();
  while(l + 1 != r)
  {
    mid = (l + r) >> 1;
    std::string key_mid = get_key_at(offsets[mid]);
    if(key_mid <= key)
    {
      l = mid;
    }
    else{
      r = mid;
    }
  }
  int end_pos = l;
  if (end_pos < first_pos) {
    end_pos = first_pos;  // 确保至少有一个元素
  }
  for(int i = first_pos; i <= end_pos; i ++)
  {
    uint64_t curr_tranc_id = get_tranc_id_at(offsets[i]);
    if(curr_tranc_id <= tranc_id)
    {
      return i;
    }
  }

  return first_pos;
```





```cpp
std::optional<std::pair<SstIterator, SstIterator>> sst_iters_monotony_predicate(
    std::shared_ptr<SST> sst, uint64_t tranc_id,
    std::function<int(const std::string &)> predicate) {
  
  std::optional<SstIterator> final_begin = std::nullopt;
  std::optional<SstIterator> final_end = std::nullopt;
  bool found_any = false;
  int last_match_block = -1;
  std::shared_ptr<BlockIterator> last_i_end = nullptr;
  
  for (int block_idx = 0; block_idx < sst->meta_entries.size(); block_idx++) {
    BlockMeta &meta_i = sst->meta_entries[block_idx];
    
    int first_res = predicate(meta_i.first_key);
    int last_res = predicate(meta_i.last_key);
    
    // 如果整个Block在目标范围左边，继续向右找
    if (last_res < 0) {
      continue;
    }
    
    // 如果整个Block在目标范围右边，停止查找
    if (first_res > 0 && last_res > 0) {
      break;
    }
    
    // Block可能包含目标key
    auto block = sst->read_block(block_idx);
    auto result_i = block->get_monotony_predicate_iters(tranc_id, predicate);
    
    if (result_i.has_value()) {
      auto [i_begin, i_end] = result_i.value();
      
      // 设置开始迭代器
      if (!final_begin.has_value()) {
        SstIterator tmp_begin(sst, tranc_id);
        tmp_begin.set_block_idx(block_idx);
        tmp_begin.set_block_it(i_begin);
        final_begin = tmp_begin;
      }
      
      // 保存最后一个有效Block的 i_end
      last_i_end = i_end;
      last_match_block = block_idx;
      found_any = true;
    }
  }
  
  // 构建最终的结束迭代器
  if (found_any && last_match_block != -1) {
    SstIterator tmp_end(sst, tranc_id);
    
    if (last_i_end->is_end()) {
      // 如果指向Block末尾，则指向下一个Block的开头
      if (last_match_block + 1 < sst->num_blocks()) {
        auto next_block = sst->read_block(last_match_block + 1);
        auto block_begin = std::make_shared<BlockIterator>(next_block, 0, tranc_id);
        tmp_end.set_block_idx(last_match_block + 1);
        tmp_end.set_block_it(block_begin);
      } else {
        // 已经是最后一个Block，指向SST末尾
        tmp_end.set_block_idx(sst->num_blocks());
        tmp_end.set_block_it(nullptr);
      }
    } else {
      tmp_end.set_block_idx(last_match_block);
      tmp_end.set_block_it(last_i_end);
    }
    
    final_end = tmp_end;
  }
  
  if (!final_begin.has_value() || !final_end.has_value() || !found_any) {
    return std::nullopt;
  }
  
  return std::make_pair(final_begin.value(), final_end.value());
}
```





```cpp
// TODO Lab 3.1 使用二分查找获取key对应的索引
  // 由于skiplistnode是越前面，这个key就是最新的key，但是如果加入版本控制了，tran_id的大小就是版本控制
  // 这里是传来了一个tranc_id，我们要找出比这个tranc_id小的版本还要最新
  if (offsets.empty()) {
    return std::nullopt;
  }
  
  int l = -1, r = offsets.size(), mid;
  // 例如，这里是 1，3，3，5
  // 我们得找到第一个3，那么第一个3是满足 左区间 < 3，右区间 >= 3
  while(l + 1 != r)
  {
    mid = (l + r) >> 1;
    std::string key_mid = get_key_at(offsets[mid]);
    if(key_mid < key)
    {
      l = mid;
    }
    else{
      r = mid;
    }
  }
  
  int first_pos = r;
  if(first_pos == offsets.size() || get_key_at(offsets[first_pos]) != key)
  {
    return std::nullopt;
  }

  l = -1, r = offsets.size();
  while(l + 1 != r)
  {
    mid = (l + r) >> 1;
    std::string key_mid = get_key_at(offsets[mid]);
    if(key_mid <= key)
    {
      l = mid;
    }
    else{
      r = mid;
    }
  }
  int end_pos = l;
  if (end_pos < first_pos) {
    end_pos = first_pos;  // 确保至少有一个元素
  }
  for(int i = first_pos; i <= end_pos; i ++)
  {
    uint64_t curr_tranc_id = get_tranc_id_at(offsets[i]);
    if(curr_tranc_id <= tranc_id)
    {
      return i;
    }
  }
  return first_pos;
```

