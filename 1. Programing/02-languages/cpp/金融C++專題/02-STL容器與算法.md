# STL 容器與算法

> 本章詳細介紹 C++ STL (Standard Template Library) 容器和算法,專注於 LeetCode 刷題所需的核心知識,包含詳細的增刪改查操作、時間複雜度分析和實戰示例。

---

## 目錄

> **學習優先級**: ⭐⭐⭐ 必看 | ⭐⭐ 建議 | ⭐ 有空再看

1. [序列容器](#1-序列容器) ⭐⭐⭐
2. [關聯容器](#2-關聯容器) ⭐⭐⭐
3. [無序容器](#3-無序容器) ⭐⭐⭐
4. [容器適配器](#4-容器適配器) ⭐⭐⭐
5. [STL 算法](#5-stl-算法) ⭐⭐⭐
6. [迭代器](#6-迭代器) ⭐⭐
7. [LeetCode 實戰](#7-leetcode-實戰) ⭐⭐⭐

---

## 1. 序列容器

### 1.1 vector - 動態數組 ⭐⭐⭐

**特點**: 連續存儲、隨機訪問、尾部高效插入

#### 構造方法

```cpp
#include <vector>
using namespace std;

// 1. 默認構造
vector<int> v1;                        // 空 vector

// 2. 指定大小
vector<int> v2(10);                    // 10 個元素,默認值 0
vector<int> v3(10, 5);                 // 10 個元素,值為 5

// 3. 初始化列表 (C++11)
vector<int> v4{1, 2, 3, 4, 5};         // {1, 2, 3, 4, 5}
vector<int> v5 = {1, 2, 3};            // 同上

// 4. 範圍構造
vector<int> v6(v4.begin(), v4.end());  // 拷貝 v4
vector<int> v7(v4.begin(), v4.begin()+3); // 拷貝前 3 個元素

// 5. 拷貝構造
vector<int> v8(v4);                    // 深拷貝
vector<int> v9 = v4;                   // 同上
```

#### 增加元素

```cpp
vector<int> v;

// 1. 尾部添加 - O(1) 均攤
v.push_back(10);                       // 添加 10
v.push_back(20);                       // 添加 20

// 2. 原地構造 - O(1) 均攤 (C++11)
v.emplace_back(30);                    // 直接在尾部構造元素

// 3. 指定位置插入 - O(n)
v.insert(v.begin(), 0);                // 在開頭插入 0
v.insert(v.begin() + 2, 15);           // 在索引 2 處插入 15
v.insert(v.end(), 40);                 // 在末尾插入 40

// 4. 插入多個元素 - O(n)
v.insert(v.begin(), 3, 100);           // 在開頭插入 3 個 100
v.insert(v.end(), {50, 60, 70});       // 在末尾插入多個元素

// 5. 範圍插入 - O(n)
vector<int> v2{1, 2, 3};
v.insert(v.end(), v2.begin(), v2.end()); // 插入另一個 vector
```

#### 刪除元素

```cpp
vector<int> v{1, 2, 3, 4, 5};

// 1. 刪除尾部元素 - O(1)
v.pop_back();                          // 刪除最後一個元素

// 2. 刪除指定位置 - O(n)
v.erase(v.begin());                    // 刪除第一個元素
v.erase(v.begin() + 2);                // 刪除索引 2 的元素

// 3. 刪除範圍 - O(n)
v.erase(v.begin(), v.begin() + 3);     // 刪除前 3 個元素
v.erase(v.begin() + 1, v.end());       // 刪除索引 1 到末尾

// 4. 清空 - O(n)
v.clear();                             // 刪除所有元素,size=0

// 5. 條件刪除 (配合算法)
v = {1, 2, 3, 2, 4, 2, 5};
v.erase(remove(v.begin(), v.end(), 2), v.end()); // 刪除所有 2
// 結果: {1, 3, 4, 5}
```

#### 訪問元素

```cpp
vector<int> v{10, 20, 30, 40, 50};

// 1. 下標訪問 - O(1) (不檢查邊界)
int x = v[0];                          // 10
v[2] = 100;                            // 修改為 100

// 2. at() 訪問 - O(1) (檢查邊界,拋出異常)
int y = v.at(1);                       // 20
// v.at(10);                           // 拋出 out_of_range 異常

// 3. 首尾訪問 - O(1)
int first = v.front();                 // 10 (第一個元素)
int last = v.back();                   // 50 (最後一個元素)

// 4. 指針訪問 - O(1)
int* ptr = v.data();                   // 獲取底層數組指針
int z = ptr[0];                        // 10
```

#### 查詢操作

```cpp
vector<int> v{1, 2, 3, 4, 5};

// 1. 大小和容量
size_t sz = v.size();                  // 5 (元素個數)
size_t cap = v.capacity();             // >= 5 (容量)
bool empty = v.empty();                // false (是否為空)
size_t max_sz = v.max_size();          // 最大可能大小

// 2. 查找元素 (需要 <algorithm>)
auto it = find(v.begin(), v.end(), 3); // 查找值為 3 的元素
if (it != v.end()) {
    cout << "找到: " << *it << "\n";
}

// 3. 計數
int cnt = count(v.begin(), v.end(), 2); // 統計值為 2 的元素個數
```

#### 修改操作

```cpp
vector<int> v{1, 2, 3};

// 1. 調整大小 - O(n)
v.resize(5);                           // 擴展到 5 個元素,新元素為 0
v.resize(10, 100);                     // 擴展到 10 個元素,新元素為 100
v.resize(2);                           // 縮小到 2 個元素

// 2. 預留容量 - O(n)
v.reserve(100);                        // 預留 100 個元素的空間
// 避免多次重新分配,提升性能

// 3. 釋放多餘容量 - O(n)
v.shrink_to_fit();                     // 釋放未使用的容量

// 4. 賦值
v.assign(5, 10);                       // 賦值為 5 個 10
v.assign({1, 2, 3, 4});                // 賦值為 {1, 2, 3, 4}

// 5. 交換 - O(1)
vector<int> v2{7, 8, 9};
v.swap(v2);                            // 交換 v 和 v2 的內容
```

#### 遍歷方式

```cpp
vector<int> v{1, 2, 3, 4, 5};

// 1. 下標遍歷
for (size_t i = 0; i < v.size(); ++i) {
    cout << v[i] << " ";
}

// 2. 迭代器遍歷
for (auto it = v.begin(); it != v.end(); ++it) {
    cout << *it << " ";
}

// 3. 範圍 for (C++11)
for (int x : v) {
    cout << x << " ";
}

// 4. 範圍 for (引用,可修改)
for (int& x : v) {
    x *= 2;                            // 所有元素乘以 2
}

// 5. 反向遍歷
for (auto it = v.rbegin(); it != v.rend(); ++it) {
    cout << *it << " ";
}
```

#### 時間複雜度總結

| 操作          | 時間複雜度 | 說明                 |
| ------------- | ---------- | -------------------- |
| `push_back()` | O(1) 均攤  | 尾部添加             |
| `pop_back()`  | O(1)       | 尾部刪除             |
| `insert()`    | O(n)       | 中間插入需要移動元素 |
| `erase()`     | O(n)       | 刪除需要移動元素     |
| `[]` / `at()` | O(1)       | 隨機訪問             |
| `find()`      | O(n)       | 線性查找             |
| `size()`      | O(1)       | 獲取大小             |

#### LeetCode 示例

```cpp
// [26] Remove Duplicates from Sorted Array
// 使用雙指針 + vector
int removeDuplicates(vector<int>& nums) {
    if (nums.empty()) return 0;
    int slow = 0;
    for (int fast = 1; fast < nums.size(); ++fast) {
        if (nums[fast] != nums[slow]) {
            nums[++slow] = nums[fast];
        }
    }
    return slow + 1;
}

// [283] Move Zeroes
// 雙指針移動零
void moveZeroes(vector<int>& nums) {
    int slow = 0;
    for (int fast = 0; fast < nums.size(); ++fast) {
        if (nums[fast] != 0) {
            swap(nums[slow++], nums[fast]);
        }
    }
}
```

---

### 1.2 deque - 雙端隊列 ⭐⭐

**特點**: 雙端高效插入刪除、隨機訪問

#### 基本操作

```cpp
#include <deque>
using namespace std;

deque<int> dq;

// 雙端添加 - O(1)
dq.push_back(10);                      // 尾部添加
dq.push_front(5);                      // 頭部添加

// 雙端刪除 - O(1)
dq.pop_back();                         // 尾部刪除
dq.pop_front();                        // 頭部刪除

// 訪問 - O(1)
int x = dq[0];                         // 下標訪問
int y = dq.front();                    // 首元素
int z = dq.back();                     // 尾元素

// 其他操作與 vector 類似
dq.insert(dq.begin(), 100);            // 插入
dq.erase(dq.begin());                  // 刪除
dq.size();                             // 大小
```

#### 適用場景

- 需要雙端操作的場景
- 滑動窗口問題
- BFS 隊列

#### LeetCode 示例

```cpp
// [239] Sliding Window Maximum
// 使用單調遞減 deque
vector<int> maxSlidingWindow(vector<int>& nums, int k) {
    deque<int> dq;  // 存儲索引
    vector<int> res;

    for (int i = 0; i < nums.size(); ++i) {
        // 移除窗口外的元素
        if (!dq.empty() && dq.front() < i - k + 1) {
            dq.pop_front();
        }

        // 維護單調遞減
        while (!dq.empty() && nums[dq.back()] < nums[i]) {
            dq.pop_back();
        }

        dq.push_back(i);

        if (i >= k - 1) {
            res.push_back(nums[dq.front()]);
        }
    }

    return res;
}
```

---

### 1.3 list - 雙向鏈表 ⭐⭐

**特點**: 任意位置高效插入刪除、不支持隨機訪問

#### 基本操作

```cpp
#include <list>
using namespace std;

list<int> lst{1, 2, 3, 4, 5};

// 添加元素 - O(1)
lst.push_back(6);                      // 尾部添加
lst.push_front(0);                     // 頭部添加

// 刪除元素 - O(1)
lst.pop_back();                        // 尾部刪除
lst.pop_front();                       // 頭部刪除

// 插入 - O(1) (已知迭代器位置)
auto it = lst.begin();
++it;
lst.insert(it, 10);                    // 在第二個位置插入 10

// 刪除 - O(1) (已知迭代器位置)
lst.erase(it);                         // 刪除迭代器指向的元素

// 特殊操作
lst.remove(3);                         // 刪除所有值為 3 的元素 O(n)
lst.unique();                          // 刪除連續重複元素 O(n)
lst.reverse();                         // 反轉 O(n)
lst.sort();                            // 排序 O(n log n)
```

#### 適用場景

- 頻繁插入刪除
- 不需要隨機訪問
- LRU Cache 實現

---

## 2. 關聯容器

### 2.1 set / multiset - 有序集合 ⭐⭐⭐

**特點**: 自動排序、元素唯一(set)、基於紅黑樹

#### set 基本操作

```cpp
#include <set>
using namespace std;

set<int> s;

// 插入 - O(log n)
s.insert(10);
s.insert(5);
s.insert(15);
s.insert(10);                          // 重複元素不會插入

// 刪除 - O(log n)
s.erase(5);                            // 刪除值為 5 的元素
s.erase(s.begin());                    // 刪除迭代器指向的元素

// 查找 - O(log n)
auto it = s.find(10);                  // 查找值為 10 的元素
if (it != s.end()) {
    cout << "找到: " << *it << "\n";
}

// 計數 - O(log n)
int cnt = s.count(10);                 // 0 或 1 (set 中元素唯一)

// C++20: contains
bool exists = s.contains(15);          // true

// 大小
size_t sz = s.size();
bool empty = s.empty();

// 遍歷 (有序)
for (int x : s) {
    cout << x << " ";                  // 按升序輸出
}
```

#### set 進階操作

```cpp
set<int> s{1, 3, 5, 7, 9};

// lower_bound - O(log n)
// 返回第一個 >= target 的元素
auto it1 = s.lower_bound(5);           // 指向 5
auto it2 = s.lower_bound(4);           // 指向 5 (第一個 >= 4)

// upper_bound - O(log n)
// 返回第一個 > target 的元素
auto it3 = s.upper_bound(5);           // 指向 7

// equal_range - O(log n)
// 返回 [lower_bound, upper_bound)
auto range = s.equal_range(5);
// range.first 指向 5, range.second 指向 7

// 自定義比較器
set<int, greater<int>> s2{1, 3, 5};    // 降序
for (int x : s2) {
    cout << x << " ";                  // 5 3 1
}
```

#### multiset 基本操作

```cpp
#include <set>
using namespace std;

multiset<int> ms;

// 允許重複元素
ms.insert(10);
ms.insert(10);
ms.insert(10);

// count 可以 > 1
int cnt = ms.count(10);                // 3

// erase 會刪除所有相同元素
ms.erase(10);                          // 刪除所有 10

// 只刪除一個
ms.insert(10);
ms.insert(10);
auto it = ms.find(10);
ms.erase(it);                          // 只刪除一個 10
```

#### LeetCode 示例

```cpp
// [220] Contains Duplicate III
// 使用 set 維護滑動窗口
bool containsNearbyAlmostDuplicate(vector<int>& nums, int k, int t) {
    set<long> window;

    for (int i = 0; i < nums.size(); ++i) {
        // 查找 [nums[i]-t, nums[i]+t] 範圍內的元素
        auto it = window.lower_bound((long)nums[i] - t);
        if (it != window.end() && *it <= (long)nums[i] + t) {
            return true;
        }

        window.insert(nums[i]);

        // 維護窗口大小
        if (i >= k) {
            window.erase(nums[i - k]);
        }
    }

    return false;
}
```

---

### 2.2 map / multimap - 有序映射 ⭐⭐⭐

**特點**: 鍵值對、自動按鍵排序、基於紅黑樹

#### map 基本操作

```cpp
#include <map>
using namespace std;

map<string, int> m;

// 插入 - O(log n)
m["apple"] = 10;                       // 下標訪問(會創建)
m["banana"] = 20;
m.insert({"cherry", 30});              // insert
m.insert(make_pair("date", 40));       // make_pair
m.emplace("elderberry", 50);           // emplace (C++11)

// 訪問 - O(log n)
int x = m["apple"];                    // 10
int y = m.at("banana");                // 20 (不存在會拋異常)
// int z = m["grape"];                 // 會創建 grape:0

// 查找 - O(log n)
auto it = m.find("cherry");
if (it != m.end()) {
    cout << it->first << ": " << it->second << "\n";
}

// 計數 - O(log n)
bool exists = m.count("apple");        // 1 或 0

// C++20: contains
bool has = m.contains("banana");       // true

// 刪除 - O(log n)
m.erase("apple");                      // 刪除鍵為 apple 的元素
m.erase(m.begin());                    // 刪除迭代器指向的元素

// 遍歷 (按鍵排序)
for (auto& [key, value] : m) {         // C++17 結構化綁定
    cout << key << ": " << value << "\n";
}

// 傳統遍歷
for (auto& p : m) {
    cout << p.first << ": " << p.second << "\n";
}
```

#### map 進階操作

```cpp
map<int, string> m{{1, "one"}, {3, "three"}, {5, "five"}};

// lower_bound / upper_bound
auto it1 = m.lower_bound(3);           // 指向 {3, "three"}
auto it2 = m.upper_bound(3);           // 指向 {5, "five"}

// 自定義比較器
map<int, string, greater<int>> m2;     // 按鍵降序
m2[1] = "one";
m2[3] = "three";
for (auto& [k, v] : m2) {
    cout << k << ": " << v << "\n";    // 3, 1
}
```

#### LeetCode 示例

```cpp
// [1] Two Sum
// 使用 map 記錄已訪問元素
vector<int> twoSum(vector<int>& nums, int target) {
    map<int, int> m;  // 值 -> 索引

    for (int i = 0; i < nums.size(); ++i) {
        int complement = target - nums[i];
        if (m.count(complement)) {
            return {m[complement], i};
        }
        m[nums[i]] = i;
    }

    return {};
}

// [49] Group Anagrams
// 使用 map 分組
vector<vector<string>> groupAnagrams(vector<string>& strs) {
    map<string, vector<string>> m;

    for (string& s : strs) {
        string key = s;
        sort(key.begin(), key.end());
        m[key].push_back(s);
    }

    vector<vector<string>> res;
    for (auto& [k, v] : m) {
        res.push_back(v);
    }

    return res;
}
```

---

## 3. 無序容器

### 3.1 unordered_set / unordered_multiset - 哈希集合 ⭐⭐⭐

**特點**: 基於哈希表、平均 O(1) 操作、無序

#### unordered_set 基本操作

```cpp
#include <unordered_set>
using namespace std;

unordered_set<int> us;

// 插入 - O(1) 平均
us.insert(10);
us.insert(20);
us.insert(30);

// 刪除 - O(1) 平均
us.erase(20);

// 查找 - O(1) 平均
auto it = us.find(10);
if (it != us.end()) {
    cout << "找到: " << *it << "\n";
}

// 計數 - O(1) 平均
bool exists = us.count(30);            // 1 或 0

// 遍歷 (無序)
for (int x : us) {
    cout << x << " ";
}
```

#### LeetCode 示例

```cpp
// [217] Contains Duplicate
// 使用 unordered_set 檢測重複
bool containsDuplicate(vector<int>& nums) {
    unordered_set<int> seen;
    for (int num : nums) {
        if (seen.count(num)) {
            return true;
        }
        seen.insert(num);
    }
    return false;
}
```

---

### 3.2 unordered_map / unordered_multimap - 哈希映射 ⭐⭐⭐

**特點**: 基於哈希表、平均 O(1) 操作、無序

#### unordered_map 基本操作

```cpp
#include <unordered_map>
using namespace std;

unordered_map<string, int> um;

// 插入 - O(1) 平均
um["apple"] = 10;
um.insert({"banana", 20});
um.emplace("cherry", 30);

// 訪問 - O(1) 平均
int x = um["apple"];

// 查找 - O(1) 平均
auto it = um.find("banana");

// 刪除 - O(1) 平均
um.erase("cherry");

// 遍歷 (無序)
for (auto& [key, value] : um) {
    cout << key << ": " << value << "\n";
}
```

#### set vs unordered_set / map vs unordered_map

| 特性     | set/map            | unordered_set/map |
| -------- | ------------------ | ----------------- |
| 底層實現 | 紅黑樹             | 哈希表            |
| 有序性   | 有序               | 無序              |
| 插入     | O(log n)           | O(1) 平均         |
| 查找     | O(log n)           | O(1) 平均         |
| 刪除     | O(log n)           | O(1) 平均         |
| 空間     | 較小               | 較大              |
| 適用場景 | 需要有序、範圍查詢 | 只需快速查找      |

#### LeetCode 示例

```cpp
// [1] Two Sum (優化版)
// 使用 unordered_map 更快
vector<int> twoSum(vector<int>& nums, int target) {
    unordered_map<int, int> m;

    for (int i = 0; i < nums.size(); ++i) {
        int complement = target - nums[i];
        if (m.count(complement)) {
            return {m[complement], i};
        }
        m[nums[i]] = i;
    }

    return {};
}
```

---

## 4. 容器適配器

### 4.1 stack - 棧 ⭐⭐⭐

**特點**: LIFO (後進先出)、基於 deque 實現

#### 基本操作

```cpp
#include <stack>
using namespace std;

stack<int> stk;

// 入棧 - O(1)
stk.push(10);
stk.push(20);
stk.push(30);

// 訪問棧頂 - O(1)
int top = stk.top();                   // 30

// 出棧 - O(1)
stk.pop();                             // 刪除 30

// 查詢
size_t sz = stk.size();                // 2
bool empty = stk.empty();              // false
```

#### LeetCode 示例

```cpp
// [20] Valid Parentheses
// 使用 stack 匹配括號
bool isValid(string s) {
    stack<char> stk;
    unordered_map<char, char> pairs = {
        {')', '('}, {']', '['}, {'}', '{'}
    };

    for (char c : s) {
        if (pairs.count(c)) {
            if (stk.empty() || stk.top() != pairs[c]) {
                return false;
            }
            stk.pop();
        } else {
            stk.push(c);
        }
    }

    return stk.empty();
}

// [155] Min Stack
// 實現支持 O(1) 獲取最小值的棧
class MinStack {
    stack<int> stk;
    stack<int> minStk;

public:
    void push(int val) {
        stk.push(val);
        if (minStk.empty() || val <= minStk.top()) {
            minStk.push(val);
        }
    }

    void pop() {
        if (stk.top() == minStk.top()) {
            minStk.pop();
        }
        stk.pop();
    }

    int top() {
        return stk.top();
    }

    int getMin() {
        return minStk.top();
    }
};
```

---

### 4.2 queue - 隊列 ⭐⭐⭐

**特點**: FIFO (先進先出)、基於 deque 實現

#### 基本操作

```cpp
#include <queue>
using namespace std;

queue<int> q;

// 入隊 - O(1)
q.push(10);
q.push(20);
q.push(30);

// 訪問 - O(1)
int front = q.front();                 // 10 (隊首)
int back = q.back();                   // 30 (隊尾)

// 出隊 - O(1)
q.pop();                               // 刪除 10

// 查詢
size_t sz = q.size();
bool empty = q.empty();
```

#### LeetCode 示例

```cpp
// [102] Binary Tree Level Order Traversal
// 使用 queue 進行 BFS
vector<vector<int>> levelOrder(TreeNode* root) {
    if (!root) return {};

    vector<vector<int>> res;
    queue<TreeNode*> q;
    q.push(root);

    while (!q.empty()) {
        int size = q.size();
        vector<int> level;

        for (int i = 0; i < size; ++i) {
            TreeNode* node = q.front();
            q.pop();
            level.push_back(node->val);

            if (node->left) q.push(node->left);
            if (node->right) q.push(node->right);
        }

        res.push_back(level);
    }

    return res;
}
```

---

### 4.3 priority_queue - 優先隊列 ⭐⭐⭐

**特點**: 最大堆(默認)、基於 vector + heap 實現

#### 基本操作

```cpp
#include <queue>
using namespace std;

// 默認最大堆
priority_queue<int> pq;

// 入隊 - O(log n)
pq.push(10);
pq.push(30);
pq.push(20);

// 訪問堆頂 - O(1)
int top = pq.top();                    // 30 (最大值)

// 出隊 - O(log n)
pq.pop();                              // 刪除 30

// 查詢
size_t sz = pq.size();
bool empty = pq.empty();

// 最小堆
priority_queue<int, vector<int>, greater<int>> minPq;
minPq.push(10);
minPq.push(30);
minPq.push(20);
int minTop = minPq.top();              // 10 (最小值)

// 自定義比較器
auto cmp = [](int a, int b) { return a > b; }; // 最小堆
priority_queue<int, vector<int>, decltype(cmp)> pq2(cmp);
```

#### LeetCode 示例

```cpp
// [215] Kth Largest Element
// 使用最小堆維護 k 個最大元素
int findKthLargest(vector<int>& nums, int k) {
    priority_queue<int, vector<int>, greater<int>> minHeap;

    for (int num : nums) {
        minHeap.push(num);
        if (minHeap.size() > k) {
            minHeap.pop();
        }
    }

    return minHeap.top();
}

// [23] Merge k Sorted Lists
// 使用優先隊列合併 k 個有序鏈表
struct Compare {
    bool operator()(ListNode* a, ListNode* b) {
        return a->val > b->val;  // 最小堆
    }
};

ListNode* mergeKLists(vector<ListNode*>& lists) {
    priority_queue<ListNode*, vector<ListNode*>, Compare> pq;

    for (ListNode* list : lists) {
        if (list) pq.push(list);
    }

    ListNode dummy(0);
    ListNode* tail = &dummy;

    while (!pq.empty()) {
        ListNode* node = pq.top();
        pq.pop();

        tail->next = node;
        tail = tail->next;

        if (node->next) {
            pq.push(node->next);
        }
    }

    return dummy.next;
}
```

---

## 5. STL 算法

### 5.1 排序算法 ⭐⭐⭐

```cpp
#include <algorithm>
#include <vector>
using namespace std;

vector<int> v{5, 2, 8, 1, 9, 3};

// sort - 快排 O(n log n)
sort(v.begin(), v.end());              // 升序: {1, 2, 3, 5, 8, 9}
sort(v.begin(), v.end(), greater<int>()); // 降序: {9, 8, 5, 3, 2, 1}

// 自定義比較器
sort(v.begin(), v.end(), [](int a, int b) {
    return a > b;                      // 降序
});

// stable_sort - 穩定排序 O(n log n)
stable_sort(v.begin(), v.end());

// partial_sort - 部分排序 O(n log k)
vector<int> v2{5, 2, 8, 1, 9, 3};
partial_sort(v2.begin(), v2.begin()+3, v2.end());
// 前 3 個元素有序: {1, 2, 3, ...}

// nth_element - 第 n 個元素 O(n)
vector<int> v3{5, 2, 8, 1, 9, 3};
nth_element(v3.begin(), v3.begin()+2, v3.end());
// v3[2] 是第 3 小的元素,左邊都 <=,右邊都 >=
```

---

### 5.2 查找算法 ⭐⭐⭐

```cpp
#include <algorithm>
#include <vector>
using namespace std;

vector<int> v{1, 2, 3, 4, 5, 6, 7, 8, 9};

// find - 線性查找 O(n)
auto it = find(v.begin(), v.end(), 5);
if (it != v.end()) {
    cout << "找到: " << *it << "\n";
}

// binary_search - 二分查找 O(log n) (需有序)
bool found = binary_search(v.begin(), v.end(), 5); // true

// lower_bound - 第一個 >= target O(log n)
auto it1 = lower_bound(v.begin(), v.end(), 5);     // 指向 5

// upper_bound - 第一個 > target O(log n)
auto it2 = upper_bound(v.begin(), v.end(), 5);     // 指向 6

// equal_range - [lower_bound, upper_bound) O(log n)
auto range = equal_range(v.begin(), v.end(), 5);

// count - 計數 O(n)
int cnt = count(v.begin(), v.end(), 5);

// count_if - 條件計數 O(n)
int cnt2 = count_if(v.begin(), v.end(), [](int x) {
    return x % 2 == 0;                 // 偶數個數
});
```

---

### 5.3 修改算法 ⭐⭐

```cpp
#include <algorithm>
#include <vector>
using namespace std;

vector<int> v{1, 2, 3, 4, 5};

// reverse - 反轉 O(n)
reverse(v.begin(), v.end());           // {5, 4, 3, 2, 1}

// rotate - 旋轉 O(n)
vector<int> v2{1, 2, 3, 4, 5};
rotate(v2.begin(), v2.begin()+2, v2.end());
// {3, 4, 5, 1, 2}

// unique - 去除連續重複 O(n) (需先排序)
vector<int> v3{1, 1, 2, 2, 3, 3, 3};
auto last = unique(v3.begin(), v3.end());
v3.erase(last, v3.end());              // {1, 2, 3}

// remove - 刪除指定值 O(n)
vector<int> v4{1, 2, 3, 2, 4, 2, 5};
auto last2 = remove(v4.begin(), v4.end(), 2);
v4.erase(last2, v4.end());             // {1, 3, 4, 5}

// remove_if - 條件刪除 O(n)
vector<int> v5{1, 2, 3, 4, 5, 6};
auto last3 = remove_if(v5.begin(), v5.end(), [](int x) {
    return x % 2 == 0;                 // 刪除偶數
});
v5.erase(last3, v5.end());             // {1, 3, 5}
```

---

### 5.4 數值算法 ⭐⭐

```cpp
#include <numeric>
#include <vector>
using namespace std;

vector<int> v{1, 2, 3, 4, 5};

// accumulate - 求和 O(n)
int sum = accumulate(v.begin(), v.end(), 0);       // 15
int product = accumulate(v.begin(), v.end(), 1, multiplies<int>()); // 120

// partial_sum - 前綴和 O(n)
vector<int> prefix(v.size());
partial_sum(v.begin(), v.end(), prefix.begin());
// prefix: {1, 3, 6, 10, 15}

// iota - 填充遞增序列 O(n)
vector<int> v2(5);
iota(v2.begin(), v2.end(), 1);         // {1, 2, 3, 4, 5}
```

---

### 5.5 排列算法 ⭐⭐

```cpp
#include <algorithm>
#include <vector>
using namespace std;

vector<int> v{1, 2, 3};

// next_permutation - 下一個排列 O(n)
do {
    for (int x : v) cout << x << " ";
    cout << "\n";
} while (next_permutation(v.begin(), v.end()));
// 輸出所有排列: 123, 132, 213, 231, 312, 321

// prev_permutation - 上一個排列 O(n)
vector<int> v2{3, 2, 1};
do {
    for (int x : v2) cout << x << " ";
    cout << "\n";
} while (prev_permutation(v2.begin(), v2.end()));
```

---

## 6. 迭代器 ⭐⭐

### 6.1 迭代器類型

```cpp
// 1. 輸入迭代器 (Input Iterator)
// 只讀,單向

// 2. 輸出迭代器 (Output Iterator)
// 只寫,單向

// 3. 前向迭代器 (Forward Iterator)
// 讀寫,單向,可多次遍歷

// 4. 雙向迭代器 (Bidirectional Iterator)
// 讀寫,雙向 (list, set, map)

// 5. 隨機訪問迭代器 (Random Access Iterator)
// 讀寫,隨機訪問 (vector, deque)
```

### 6.2 迭代器操作

```cpp
#include <vector>
#include <iterator>
using namespace std;

vector<int> v{1, 2, 3, 4, 5};

// 基本操作
auto it = v.begin();                   // 指向第一個元素
auto end = v.end();                    // 指向最後一個元素之後

// 移動
++it;                                  // 下一個
--it;                                  // 上一個 (雙向/隨機訪問)
it += 2;                               // 前進 2 (隨機訪問)
it -= 1;                               // 後退 1 (隨機訪問)

// 訪問
int x = *it;                           // 解引用
int y = it[2];                         // 下標訪問 (隨機訪問)

// 距離
int dist = distance(v.begin(), v.end()); // 5

// 前進
advance(it, 3);                        // it 前進 3 個位置

// 反向迭代器
auto rit = v.rbegin();                 // 指向最後一個元素
auto rend = v.rend();                  // 指向第一個元素之前
```

---

## 7. LeetCode 實戰 ⭐⭐⭐

### 7.1 常見題型與 STL 應用

#### 哈希表題型

```cpp
// [1] Two Sum - unordered_map
// [49] Group Anagrams - map/unordered_map
// [217] Contains Duplicate - unordered_set
// [242] Valid Anagram - unordered_map
```

#### 棧題型

```cpp
// [20] Valid Parentheses - stack
// [155] Min Stack - stack
// [739] Daily Temperatures - stack (單調棧)
```

#### 隊列題型

```cpp
// [102] Binary Tree Level Order - queue (BFS)
// [239] Sliding Window Maximum - deque (單調隊列)
```

#### 優先隊列題型

```cpp
// [215] Kth Largest Element - priority_queue
// [23] Merge k Sorted Lists - priority_queue
// [347] Top K Frequent Elements - priority_queue
```

#### 排序題型

```cpp
// [56] Merge Intervals - sort
// [75] Sort Colors - 計數排序
// [215] Kth Largest - nth_element
```

---

### 7.2 性能優化技巧

#### 1. 預留容量

```cpp
// ❌ 不好
vector<int> v;
for (int i = 0; i < 10000; ++i) {
    v.push_back(i);  // 可能多次重新分配
}

// ✅ 好
vector<int> v;
v.reserve(10000);    // 預留空間
for (int i = 0; i < 10000; ++i) {
    v.push_back(i);  // 不會重新分配
}
```

#### 2. emplace vs push

```cpp
// ❌ 不好
vector<pair<int, int>> v;
v.push_back(make_pair(1, 2));  // 創建臨時對象

// ✅ 好
vector<pair<int, int>> v;
v.emplace_back(1, 2);          // 原地構造
```

#### 3. 選擇合適的容器

```cpp
// 需要有序 + 範圍查詢 -> set/map
// 只需快速查找 -> unordered_set/unordered_map
// 頻繁插入刪除 -> list
// 隨機訪問 -> vector
// 雙端操作 -> deque
```

---

## 總結

### STL 容器選擇指南

| 需求              | 推薦容器                         |
| ----------------- | -------------------------------- |
| 動態數組,隨機訪問 | `vector`                         |
| 雙端操作          | `deque`                          |
| 頻繁插入刪除      | `list`                           |
| 快速查找,有序     | `set`, `map`                     |
| 快速查找,無序     | `unordered_set`, `unordered_map` |
| LIFO              | `stack`                          |
| FIFO              | `queue`                          |
| 優先級            | `priority_queue`                 |

### 時間複雜度對比

| 操作     | vector | deque | list | set/map  | unordered |
| -------- | ------ | ----- | ---- | -------- | --------- |
| 隨機訪問 | O(1)   | O(1)  | O(n) | O(log n) | -         |
| 頭部插入 | O(n)   | O(1)  | O(1) | O(log n) | O(1)      |
| 尾部插入 | O(1)   | O(1)  | O(1) | O(log n) | O(1)      |
| 中間插入 | O(n)   | O(n)  | O(1) | O(log n) | O(1)      |
| 查找     | O(n)   | O(n)  | O(n) | O(log n) | O(1)      |

### LeetCode 刷題建議

1. **熟練掌握**: vector, unordered_map, unordered_set, stack, queue
2. **理解原理**: priority_queue, set, map
3. **靈活運用**: 根據題目特點選擇合適容器
4. **注意細節**: 邊界條件、空容器處理

---

## 參考資料 (References)

1. [C++ Reference - Containers](https://en.cppreference.com/w/cpp/container)
2. [C++ Reference - Algorithms](https://en.cppreference.com/w/cpp/algorithm)
3. [LeetCode](https://leetcode.com/)
4. [Effective STL](https://www.oreilly.com/library/view/effective-stl/9780321545183/)
5. [STL Tutorial](https://www.cplusplus.com/reference/stl/)
