### 語法與內置函數
* 值交換: `a, b = b, a`
* 列表生成式：`[x * x for x in range(10)]`
* enumerate: `for i, v in enumerate(arr)`
* zip 配對：`for a, b in zip(arr1, arr2)`
* 自定排序：`sorted(arr, key=lambda x: x[1])`

### 集合操作

##### List
```python
nums = [1, 2, 3]
nums.append(4)          # 加到尾端
nums.pop()              # 移除尾端
nums.pop(1)             # 移除指定位置 (返回刪除元素)
del nums[1]             # 移除指定位置
nums.insert(1, 100)     # 插入
nums.remove(2)          # 移除值為 2 的元素W
nums.sort()             # 原地排序
nums[::-1]              # 反轉列表
```

##### Set
```python
s = {1, 2, 3}
s = set([1, 2, 2, 3])    # 去重
s.add(4)
s.remove(1)
s1 & s2                  # 交集
s1 | s2                  # 聯集
s1 - s2                  # 差集 e.g. s1 - {2}
if "a" in s              # 查找
```


##### Dict
```python
d = {'a': 1, 'b': 2}
d['c'] = 3
d.get('d', 0)           # 如果沒找到返回預設值
d.pop('c')              # 移除指定位置 (返回刪除元素)
del d['c']              # 移除指定位置
for k, v in d.items():  # 迭代 key, value
d.keys()                # 取得所有 key
d.values()              # 取得所有 value

if "a" in d             # 與 set 相同
```


##### Queue / Dequeue
```python
from collections import deque
q = deque()
q.append(1)       # 入隊
q.popleft()       # 出隊
q.appendleft(2)   # 左側入隊
q.pop()           # 右側出隊

```

##### Stack
```python
stack = []
stack.append(x)
stack.pop()
```

##### Heap (priority queue)
```python
import heapq
h = []
heapq.heappush(h, 2)
heapq.heappop(h)
# 最大堆 (轉負值)
heapq.heappush(h, -val)
-heapq.heappop(h)
```