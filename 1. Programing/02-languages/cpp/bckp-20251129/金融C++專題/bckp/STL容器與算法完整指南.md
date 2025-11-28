# STL 容器與算法完整指南

> 本章獨立涵蓋 C++ STL (Standard Template Library) 的所有重要容器、迭代器與算法，提供 Leetcode 刷題與金融系統開發所需的完整知識。

---

## 目錄

1. [序列容器](#1-序列容器)
2. [關聯容器](#2-關聯容器)
3. [無序關聯容器](#3-無序關聯容器)
4. [容器適配器](#4-容器適配器)
5. [輔助工具](#5-輔助工具)
6. [迭代器](#6-迭代器)
7. [算法庫](#7-算法庫)
8. [Leetcode 實戰技巧](#8-leetcode-實戰技巧)
9. [金融系統應用](#9-金融系統應用)
10. [練習題](#10-練習題)

---

## 1. 序列容器

序列容器按照線性順序存儲元素，支持按位置訪問。

### 1.1 std::vector - 動態陣列

**特性**: 連續記憶體、隨機訪問 O(1)、尾部插入均攤 O(1)

```cpp
#include <vector>
using namespace std;

// === 初始化方式 ===
vector<int> v1;                      // 空 vector
vector<int> v2(5);                   // 5 個元素，默認值 0
vector<int> v3(5, 10);               // 5 個元素，值為 10
vector<int> v4 = {1, 2, 3, 4, 5};    // 初始化列表
vector<int> v5(v4);                  // 拷貝構造
vector<int> v6(v4.begin(), v4.end()); // 迭代器範圍

// 2D vector
vector<vector<int>> matrix(3, vector<int>(4, 0));  // 3x4 矩陣，全 0

// === 容量操作 ===
v1.size();           // 元素個數
v1.capacity();       // 當前容量
v1.empty();          // 是否為空
v1.reserve(100);     // 預分配容量（避免多次擴容）
v1.shrink_to_fit();  // 釋放多餘容量
v1.resize(10);       // 調整大小
v1.resize(10, -1);   // 調整大小，新元素值為 -1

// === 元素訪問 ===
v4[0];               // 下標訪問（不檢查邊界）
v4.at(0);            // 帶邊界檢查的訪問
v4.front();          // 第一個元素
v4.back();           // 最後一個元素
v4.data();           // 底層陣列指標

// === 修改操作 ===
v1.push_back(1);     // 尾部添加
v1.emplace_back(1);  // 尾部原地構造（更高效）
v1.pop_back();       // 刪除尾部
v1.insert(v1.begin(), 0);           // 頭部插入
v1.insert(v1.begin() + 1, 3, 5);    // 位置 1 插入 3 個 5
v1.erase(v1.begin());               // 刪除頭部
v1.erase(v1.begin(), v1.begin()+3); // 刪除範圍
v1.clear();          // 清空
v1.swap(v2);         // 交換內容

// === 迭代器 ===
for (auto it = v4.begin(); it != v4.end(); ++it) {
    cout << *it << " ";
}

// 範圍 for 循環
for (int x : v4) cout << x << " ";
for (int& x : v4) x *= 2;  // 修改元素
for (const auto& x : v4) cout << x;  // 最佳實踐

// 反向迭代
for (auto it = v4.rbegin(); it != v4.rend(); ++it) {
    cout << *it << " ";
}
```

**性能特點**:
- 隨機訪問: O(1)
- 尾部插入/刪除: 均攤 O(1)
- 頭部/中間插入/刪除: O(n)
- 查找: O(n)

**使用時機**: 需要隨機訪問、主要在尾部操作、元素數量已知或可預估

---

### 1.2 std::deque - 雙端隊列

**特性**: 分段連續記憶體、頭尾插入 O(1)、隨機訪問 O(1)

```cpp
#include <deque>

deque<int> dq = {1, 2, 3};

// === 雙端操作 ===
dq.push_front(0);    // 頭部添加
dq.push_back(4);     // 尾部添加
dq.emplace_front(0); // 頭部原地構造
dq.emplace_back(4);  // 尾部原地構造
dq.pop_front();      // 刪除頭部
dq.pop_back();       // 刪除尾部

// 訪問方式與 vector 相同
dq[0];
dq.at(0);
dq.front();
dq.back();
```

**使用時機**: 需要高效的頭尾雙端操作、滑動窗口問題

---

### 1.3 std::list - 雙向鏈表

**特性**: 非連續記憶體、任意位置插入刪除 O(1)、不支持隨機訪問

```cpp
#include <list>

list<int> lst = {1, 2, 3, 4, 5};

// === 基本操作 ===
lst.push_front(0);
lst.push_back(6);
lst.pop_front();
lst.pop_back();

// === 插入與刪除 ===
auto it = lst.begin();
advance(it, 2);      // 移動迭代器
lst.insert(it, 10);  // 在位置前插入
lst.erase(it);       // 刪除元素

// === list 特有操作 ===
lst.remove(3);       // 刪除所有值為 3 的元素
lst.remove_if([](int x) { return x % 2 == 0; }); // 條件刪除

lst.unique();        // 刪除連續重複元素
lst.sort();          // 排序（list 專用，比 std::sort 效率高）
lst.reverse();       // 反轉

// 合併兩個已排序 list
list<int> lst2 = {2, 4, 6};
lst.merge(lst2);     // lst2 會變空

// 拼接
list<int> lst3 = {7, 8, 9};
lst.splice(lst.end(), lst3);  // 將 lst3 全部移到 lst 尾部
```

**使用時機**: 頻繁在中間插入刪除、不需要隨機訪問、LRU Cache

---

### 1.4 std::array - 固定大小陣列

**特性**: 編譯期固定大小、棧上分配、零開銷抽象

```cpp
#include <array>

array<int, 5> arr = {1, 2, 3, 4, 5};

arr.size();          // 永遠是 5
arr.fill(0);         // 全部填充為 0
arr[0] = 10;
arr.at(0);
arr.front();
arr.back();
arr.data();          // 底層指標

// 2D array
array<array<int, 3>, 2> arr2d = {{{1,2,3}, {4,5,6}}};
```

**使用時機**: 大小編譯期已知、需要棧分配、替代 C 風格陣列

---

### 1.5 std::string - 字符串

**特性**: 專門為字符串優化的容器，類似 `vector<char>`

```cpp
#include <string>

// === 初始化 ===
string s1;
string s2 = "hello";
string s3(5, 'a');           // "aaaaa"
string s4 = s2;
string s5 = s2.substr(1, 3); // "ell"

// === 基本操作 ===
s1.size();           // 或 s1.length()
s1.empty();
s1.clear();

// === 修改操作 ===
s1 += "world";
s1.append(" !");
s1.push_back('?');
s1.pop_back();
s1.insert(0, "Hi ");
s1.erase(0, 3);
s1.replace(0, 5, "Hello");

// === 查找操作 ===
s2.find("ll");       // 返回位置，找不到返回 string::npos
s2.rfind("l");       // 從後往前找
s2.find_first_of("aeiou");  // 找第一個母音
s2.find_last_of("aeiou");
s2.find_first_not_of("hel");

// === 比較 ===
s1 == s2;
s1 < s2;             // 字典序比較
s1.compare(s2);      // 返回 -1, 0, 1

// === 轉換 ===
int n = stoi("123");
long l = stol("123456789");
double d = stod("3.14");
string numStr = to_string(123);

// === 子串操作 ===
s2.substr(1);        // 從位置 1 到結尾
s2.substr(1, 3);     // 從位置 1 取 3 個字符

// === C 風格字符串 ===
const char* cstr = s2.c_str();

// === 字符操作 ===
#include <cctype>
isalpha(c);  isdigit(c);  isalnum(c);
islower(c);  isupper(c);  isspace(c);
tolower(c);  toupper(c);
```

---

### 1.6 std::forward_list - 單向鏈表 (C++11)

```cpp
#include <forward_list>

forward_list<int> fl = {1, 2, 3};

fl.push_front(0);
fl.pop_front();
fl.insert_after(fl.begin(), 10);
fl.erase_after(fl.begin());
```

**使用時機**: 內存敏感場景、只需要單向遍歷

---

## 2. 關聯容器

基於紅黑樹實現，元素自動排序，操作時間複雜度 O(log n)。

### 2.1 std::set - 有序集合

**特性**: 元素唯一、自動排序、O(log n) 查找/插入/刪除

```cpp
#include <set>

set<int> s = {3, 1, 4, 1, 5};  // {1, 3, 4, 5}，自動去重排序

// === 插入 ===
s.insert(2);
s.emplace(6);
auto [it, success] = s.insert(3);  // C++17 結構化綁定

// === 查找 ===
s.count(3);          // 返回 0 或 1
s.find(3);           // 返回迭代器，找不到返回 end()
s.contains(3);       // C++20，返回 bool

// === 範圍查找 ===
s.lower_bound(3);    // >= 3 的第一個元素
s.upper_bound(3);    // > 3 的第一個元素
auto [lo, hi] = s.equal_range(3);  // 返回 [lower_bound, upper_bound)

// === 刪除 ===
s.erase(3);          // 按值刪除
s.erase(s.begin());  // 按迭代器刪除
s.erase(s.begin(), s.end());

// === 遍歷（已排序）===
for (int x : s) cout << x << " ";  // 1 3 4 5 6
```

---

### 2.2 std::multiset - 可重複有序集合

```cpp
#include <set>

multiset<int> ms = {1, 1, 2, 2, 3};

ms.insert(1);
ms.count(1);         // 返回出現次數
ms.erase(1);         // 刪除所有 1
ms.erase(ms.find(1)); // 只刪除一個 1
```

---

### 2.3 std::map - 有序映射

**特性**: 鍵值對、按鍵排序、鍵唯一

```cpp
#include <map>

map<string, int> m;

// === 插入 ===
m["apple"] = 1;
m.insert({"banana", 2});
m.emplace("cherry", 3);
m.insert_or_assign("apple", 10);  // C++17，插入或更新

// === 訪問 ===
m["apple"];          // 不存在則插入默認值！
m.at("apple");       // 不存在拋異常

// === 查找 ===
m.count("apple");
m.find("apple");
m.contains("apple"); // C++20

// === 遍歷 ===
for (const auto& [key, value] : m) {  // C++17 結構化綁定
    cout << key << ": " << value << endl;
}

// 傳統方式
for (auto it = m.begin(); it != m.end(); ++it) {
    cout << it->first << ": " << it->second << endl;
}

// === 刪除 ===
m.erase("apple");
m.erase(m.begin());

// === 範圍查找 ===
m.lower_bound("b");
m.upper_bound("b");
```

**重要陷阱**:
```cpp
map<string, int> m;
int x = m["nonexistent"];  // 會插入 {"nonexistent", 0}！
// 正確做法
if (m.count("key")) { ... }
if (auto it = m.find("key"); it != m.end()) { ... }
```

---

### 2.4 std::multimap - 可重複鍵有序映射

```cpp
#include <map>

multimap<string, int> mm;
mm.insert({"a", 1});
mm.insert({"a", 2});
mm.insert({"a", 3});

// 獲取所有鍵為 "a" 的值
auto [lo, hi] = mm.equal_range("a");
for (auto it = lo; it != hi; ++it) {
    cout << it->second << " ";  // 1 2 3
}
```

---

## 3. 無序關聯容器

基於哈希表實現，平均 O(1) 操作，最壞 O(n)。

### 3.1 std::unordered_set

```cpp
#include <unordered_set>

unordered_set<int> us = {3, 1, 4, 1, 5};

us.insert(2);
us.erase(3);
us.count(1);
us.find(1);
us.contains(1);  // C++20

// 哈希表特有操作
us.bucket_count();   // 桶數量
us.load_factor();    // 負載因子
us.reserve(100);     // 預留空間
```

---

### 3.2 std::unordered_map

```cpp
#include <unordered_map>

unordered_map<string, int> um;

um["apple"] = 1;
um.insert({"banana", 2});
um.emplace("cherry", 3);

um.count("apple");
um.find("apple");
um["apple"];  // 同樣有插入默認值的陷阱！

// 遍歷（無序）
for (const auto& [k, v] : um) {
    cout << k << ": " << v << endl;
}
```

**自定義類型作為鍵**:
```cpp
struct Point {
    int x, y;
    bool operator==(const Point& other) const {
        return x == other.x && y == other.y;
    }
};

// 自定義哈希函數
struct PointHash {
    size_t operator()(const Point& p) const {
        return hash<int>()(p.x) ^ (hash<int>()(p.y) << 1);
    }
};

unordered_set<Point, PointHash> points;
unordered_map<Point, int, PointHash> pointMap;
```

---

### 3.3 unordered_multiset / unordered_multimap

與 multiset/multimap 類似，但基於哈希表。

---

## 4. 容器適配器

基於其他容器實現的特殊數據結構。

### 4.1 std::stack - 棧

**特性**: LIFO (後進先出)

```cpp
#include <stack>

stack<int> stk;              // 默認基於 deque
stack<int, vector<int>> stk2; // 基於 vector

stk.push(1);
stk.emplace(2);
stk.top();           // 查看頂部元素
stk.pop();           // 刪除頂部
stk.empty();
stk.size();
```

**Leetcode 常見應用**: 括號匹配、單調棧、計算器

```cpp
// 有效括號
bool isValid(string s) {
    stack<char> stk;
    unordered_map<char, char> pairs = {{')', '('}, {']', '['}, {'}', '{'}};
    for (char c : s) {
        if (pairs.count(c)) {
            if (stk.empty() || stk.top() != pairs[c]) return false;
            stk.pop();
        } else {
            stk.push(c);
        }
    }
    return stk.empty();
}
```

---

### 4.2 std::queue - 隊列

**特性**: FIFO (先進先出)

```cpp
#include <queue>

queue<int> q;               // 默認基於 deque

q.push(1);
q.emplace(2);
q.front();           // 隊首
q.back();            // 隊尾
q.pop();             // 刪除隊首
q.empty();
q.size();
```

**Leetcode 常見應用**: BFS、層序遍歷

```cpp
// BFS 模板
void bfs(TreeNode* root) {
    if (!root) return;
    queue<TreeNode*> q;
    q.push(root);
    while (!q.empty()) {
        int size = q.size();
        for (int i = 0; i < size; ++i) {
            TreeNode* node = q.front();
            q.pop();
            // 處理當前節點
            if (node->left) q.push(node->left);
            if (node->right) q.push(node->right);
        }
    }
}
```

---

### 4.3 std::priority_queue - 優先隊列/堆

**特性**: 默認最大堆，堆頂為最大元素

```cpp
#include <queue>

// 最大堆（默認）
priority_queue<int> maxHeap;
maxHeap.push(3);
maxHeap.push(1);
maxHeap.push(4);
maxHeap.top();       // 4
maxHeap.pop();

// 最小堆
priority_queue<int, vector<int>, greater<int>> minHeap;
minHeap.push(3);
minHeap.push(1);
minHeap.push(4);
minHeap.top();       // 1

// 自定義比較
auto cmp = [](const pair<int,int>& a, const pair<int,int>& b) {
    return a.second > b.second;  // 按 second 最小堆
};
priority_queue<pair<int,int>, vector<pair<int,int>>, decltype(cmp)> pq(cmp);

// 使用 struct 自定義比較
struct Compare {
    bool operator()(const pair<int,int>& a, const pair<int,int>& b) {
        return a.first + a.second > b.first + b.second;
    }
};
priority_queue<pair<int,int>, vector<pair<int,int>>, Compare> pq2;
```

**Leetcode 常見應用**: Top K、合併 K 個有序鏈表、Dijkstra

```cpp
// 找第 K 大元素
int findKthLargest(vector<int>& nums, int k) {
    priority_queue<int, vector<int>, greater<int>> minHeap;
    for (int num : nums) {
        minHeap.push(num);
        if (minHeap.size() > k) minHeap.pop();
    }
    return minHeap.top();
}
```

---

## 5. 輔助工具

### 5.1 std::pair

```cpp
#include <utility>

pair<int, string> p1 = {1, "one"};
pair<int, string> p2 = make_pair(2, "two");

p1.first;
p1.second;

// 比較（先比較 first，相等再比較 second）
p1 < p2;

// 結構化綁定 (C++17)
auto [num, str] = p1;
```

---

### 5.2 std::tuple

```cpp
#include <tuple>

tuple<int, double, string> t = {1, 3.14, "hello"};
tuple<int, double, string> t2 = make_tuple(2, 2.71, "world");

// 訪問
get<0>(t);           // 1
get<1>(t);           // 3.14
get<2>(t);           // "hello"

// 結構化綁定 (C++17)
auto [a, b, c] = t;

// 解包
int x; double y; string z;
tie(x, y, z) = t;
tie(x, ignore, z) = t;  // 忽略某些值

// 連接 tuple
auto t3 = tuple_cat(t, t2);
```

---

### 5.3 std::optional (C++17)

```cpp
#include <optional>

optional<int> maybeInt;
optional<int> hasInt = 42;

if (maybeInt) { ... }
if (hasInt.has_value()) { ... }

hasInt.value();      // 有值返回，無值拋異常
hasInt.value_or(0);  // 有值返回，無值返回默認值
*hasInt;             // 直接訪問（不檢查）

maybeInt = 10;
maybeInt.reset();    // 變回無值
maybeInt = nullopt;  // 同上
```

---

### 5.4 std::variant (C++17)

```cpp
#include <variant>

variant<int, double, string> v = 42;

v = 3.14;
v = "hello";

// 訪問
get<string>(v);
get<2>(v);           // 按索引

// 安全訪問
if (holds_alternative<string>(v)) {
    cout << get<string>(v);
}

// 訪問者模式
visit([](auto&& arg) {
    cout << arg << endl;
}, v);
```

---

## 6. 迭代器

### 6.1 迭代器類別

```cpp
// 輸入迭代器: istream_iterator
// 輸出迭代器: ostream_iterator
// 前向迭代器: forward_list::iterator
// 雙向迭代器: list::iterator, set::iterator
// 隨機訪問迭代器: vector::iterator, deque::iterator
```

### 6.2 迭代器操作

```cpp
#include <iterator>

vector<int> v = {1, 2, 3, 4, 5};

// 移動迭代器
auto it = v.begin();
advance(it, 2);      // it 指向第 3 個元素
it = next(it);       // 返回下一個位置
it = prev(it);       // 返回上一個位置
it = next(it, 2);    // 前進 2 步

// 距離
int d = distance(v.begin(), it);

// 插入迭代器
back_insert_iterator<vector<int>> bi(v);
*bi = 6;             // 相當於 push_back(6)

// 更簡潔
back_inserter(v);
front_inserter(deque<int>());
inserter(v, v.begin());
```

### 6.3 反向迭代器

```cpp
vector<int> v = {1, 2, 3, 4, 5};

for (auto it = v.rbegin(); it != v.rend(); ++it) {
    cout << *it << " ";  // 5 4 3 2 1
}

// 反向迭代器轉正向
auto rit = v.rbegin();
advance(rit, 2);     // 指向 3
auto it = rit.base(); // 指向 3 後面的 4
```

---

## 7. 算法庫

### 7.1 排序算法

```cpp
#include <algorithm>

vector<int> v = {3, 1, 4, 1, 5, 9, 2, 6};

// 排序
sort(v.begin(), v.end());              // 升序
sort(v.begin(), v.end(), greater<>()); // 降序

// 自定義比較
sort(v.begin(), v.end(), [](int a, int b) {
    return abs(a) < abs(b);  // 按絕對值排序
});

// 穩定排序
stable_sort(v.begin(), v.end());

// 部分排序（找前 K 小）
partial_sort(v.begin(), v.begin() + 3, v.end());

// 找第 K 小元素
nth_element(v.begin(), v.begin() + 3, v.end());
// v[3] 就是第 4 小的元素，左邊都比它小，右邊都比它大

// 檢查是否已排序
is_sorted(v.begin(), v.end());
```

---

### 7.2 二分查找

```cpp
vector<int> v = {1, 2, 3, 4, 5, 6, 7, 8, 9};

// 必須已排序！

binary_search(v.begin(), v.end(), 5);  // 返回 bool

lower_bound(v.begin(), v.end(), 5);    // >= 5 的第一個
upper_bound(v.begin(), v.end(), 5);    // > 5 的第一個
equal_range(v.begin(), v.end(), 5);    // [lower, upper)

// 使用示例
auto it = lower_bound(v.begin(), v.end(), 5);
if (it != v.end() && *it == 5) {
    cout << "Found at index: " << it - v.begin();
}

// 自定義比較的二分查找
vector<pair<int,int>> vp = {{1,10}, {2,20}, {3,30}};
auto it2 = lower_bound(vp.begin(), vp.end(), 2, 
    [](const pair<int,int>& p, int val) {
        return p.first < val;
    });
```

---

### 7.3 查找算法

```cpp
vector<int> v = {1, 2, 3, 4, 5, 3, 2, 1};

find(v.begin(), v.end(), 3);           // 找第一個 3
find_if(v.begin(), v.end(), [](int x) { return x > 3; });  // 找第一個 >3
find_if_not(v.begin(), v.end(), [](int x) { return x < 3; });

count(v.begin(), v.end(), 3);          // 計數
count_if(v.begin(), v.end(), [](int x) { return x % 2 == 0; });

// 找子序列
vector<int> sub = {3, 4};
search(v.begin(), v.end(), sub.begin(), sub.end());

// 找連續重複
adjacent_find(v.begin(), v.end());     // 找連續相等
```

---

### 7.4 修改算法

```cpp
vector<int> v = {1, 2, 3, 4, 5};
vector<int> v2(5);

// 複製
copy(v.begin(), v.end(), v2.begin());
copy_n(v.begin(), 3, v2.begin());
copy_if(v.begin(), v.end(), v2.begin(), [](int x) { return x % 2; });

// 移動
move(v.begin(), v.end(), v2.begin());

// 變換
transform(v.begin(), v.end(), v2.begin(), [](int x) { return x * 2; });

// 雙輸入變換
transform(v.begin(), v.end(), v2.begin(), v.begin(), 
    [](int a, int b) { return a + b; });

// 填充
fill(v.begin(), v.end(), 0);
fill_n(v.begin(), 3, -1);
generate(v.begin(), v.end(), []() { return rand() % 100; });

// 替換
replace(v.begin(), v.end(), 3, 30);    // 把 3 換成 30
replace_if(v.begin(), v.end(), [](int x) { return x < 0; }, 0);

// 刪除（注意：只是移動，需要配合 erase）
auto newEnd = remove(v.begin(), v.end(), 3);
v.erase(newEnd, v.end());

// Erase-Remove 慣用法
v.erase(remove_if(v.begin(), v.end(), [](int x) { return x < 0; }), v.end());

// 去重（需先排序）
sort(v.begin(), v.end());
v.erase(unique(v.begin(), v.end()), v.end());

// 反轉
reverse(v.begin(), v.end());

// 旋轉
rotate(v.begin(), v.begin() + 2, v.end());  // 左旋 2 位

// 洗牌
#include <random>
random_device rd;
mt19937 g(rd());
shuffle(v.begin(), v.end(), g);
```

---

### 7.5 數值算法

```cpp
#include <numeric>

vector<int> v = {1, 2, 3, 4, 5};

// 累加
int sum = accumulate(v.begin(), v.end(), 0);

// 自定義累加
int product = accumulate(v.begin(), v.end(), 1, multiplies<int>());

// 內積
vector<int> v2 = {1, 2, 3, 4, 5};
int dot = inner_product(v.begin(), v.end(), v2.begin(), 0);

// 前綴和
vector<int> prefix(5);
partial_sum(v.begin(), v.end(), prefix.begin());
// prefix = {1, 3, 6, 10, 15}

// 差分
vector<int> diff(5);
adjacent_difference(v.begin(), v.end(), diff.begin());
// diff = {1, 1, 1, 1, 1}

// iota - 填充遞增序列
iota(v.begin(), v.end(), 1);  // {1, 2, 3, 4, 5}

// C++17 並行算法
#include <execution>
reduce(execution::par, v.begin(), v.end());
transform_reduce(execution::par, v.begin(), v.end(), v2.begin(), 0);
```

---

### 7.6 最值與比較

```cpp
vector<int> v = {3, 1, 4, 1, 5, 9, 2, 6};

// 最值
*min_element(v.begin(), v.end());
*max_element(v.begin(), v.end());
auto [minIt, maxIt] = minmax_element(v.begin(), v.end());

min(3, 5);
max(3, 5);
minmax(3, 5);        // 返回 pair
min({3, 1, 4, 1, 5}); // 初始化列表
clamp(10, 1, 5);     // 限制範圍，返回 5

// 比較
equal(v.begin(), v.end(), v2.begin());
lexicographical_compare(v.begin(), v.end(), v2.begin(), v2.end());
```

---

### 7.7 排列組合

```cpp
vector<int> v = {1, 2, 3};

// 全排列
do {
    for (int x : v) cout << x << " ";
    cout << endl;
} while (next_permutation(v.begin(), v.end()));

// 上一個排列
prev_permutation(v.begin(), v.end());

// 檢查是否為排列
is_permutation(v.begin(), v.end(), v2.begin());
```

---

### 7.8 集合操作（有序範圍）

```cpp
vector<int> a = {1, 2, 3, 4, 5};
vector<int> b = {3, 4, 5, 6, 7};
vector<int> result;

// 並集
set_union(a.begin(), a.end(), b.begin(), b.end(), back_inserter(result));
// {1, 2, 3, 4, 5, 6, 7}

// 交集
result.clear();
set_intersection(a.begin(), a.end(), b.begin(), b.end(), back_inserter(result));
// {3, 4, 5}

// 差集
result.clear();
set_difference(a.begin(), a.end(), b.begin(), b.end(), back_inserter(result));
// {1, 2}

// 對稱差集
result.clear();
set_symmetric_difference(a.begin(), a.end(), b.begin(), b.end(), back_inserter(result));
// {1, 2, 6, 7}

// 包含關係
includes(a.begin(), a.end(), b.begin(), b.end());
```

---

### 7.9 堆操作

```cpp
vector<int> v = {3, 1, 4, 1, 5, 9, 2, 6};

// 建堆
make_heap(v.begin(), v.end());         // 最大堆

// 添加元素
v.push_back(10);
push_heap(v.begin(), v.end());

// 刪除堆頂
pop_heap(v.begin(), v.end());          // 堆頂移到末尾
v.pop_back();

// 堆排序
sort_heap(v.begin(), v.end());

// 檢查是否為堆
is_heap(v.begin(), v.end());
```

---

## 8. Leetcode 實戰技巧

### 8.1 容器選擇指南

| 場景 | 推薦容器 |
|------|----------|
| 動態陣列、隨機訪問 | vector |
| 需要頻繁頭部操作 | deque |
| 頻繁中間插入刪除 | list |
| 去重 + 排序 | set |
| 去重 + O(1) 查找 | unordered_set |
| 鍵值對 + 排序 | map |
| 鍵值對 + O(1) 查找 | unordered_map |
| LIFO | stack |
| FIFO / BFS | queue |
| Top K / 堆 | priority_queue |

---

### 8.2 常用技巧

#### 8.2.1 Two Pointers

```cpp
// 反轉陣列
void reverse(vector<int>& v) {
    int l = 0, r = v.size() - 1;
    while (l < r) swap(v[l++], v[r--]);
}

// 移除元素
int removeElement(vector<int>& nums, int val) {
    int slow = 0;
    for (int fast = 0; fast < nums.size(); fast++) {
        if (nums[fast] != val) {
            nums[slow++] = nums[fast];
        }
    }
    return slow;
}
```

#### 8.2.2 Sliding Window

```cpp
// 最長無重複子串
int lengthOfLongestSubstring(string s) {
    unordered_set<char> window;
    int left = 0, maxLen = 0;
    for (int right = 0; right < s.size(); right++) {
        while (window.count(s[right])) {
            window.erase(s[left++]);
        }
        window.insert(s[right]);
        maxLen = max(maxLen, right - left + 1);
    }
    return maxLen;
}
```

#### 8.2.3 前綴和

```cpp
// 子陣列和等於 K
int subarraySum(vector<int>& nums, int k) {
    unordered_map<int, int> prefixCount;
    prefixCount[0] = 1;
    int sum = 0, count = 0;
    for (int num : nums) {
        sum += num;
        if (prefixCount.count(sum - k)) {
            count += prefixCount[sum - k];
        }
        prefixCount[sum]++;
    }
    return count;
}
```

#### 8.2.4 單調棧

```cpp
// 下一個更大元素
vector<int> nextGreaterElement(vector<int>& nums) {
    int n = nums.size();
    vector<int> result(n, -1);
    stack<int> stk;  // 存索引
    for (int i = 0; i < n; i++) {
        while (!stk.empty() && nums[stk.top()] < nums[i]) {
            result[stk.top()] = nums[i];
            stk.pop();
        }
        stk.push(i);
    }
    return result;
}
```

#### 8.2.5 單調隊列

```cpp
// 滑動窗口最大值
vector<int> maxSlidingWindow(vector<int>& nums, int k) {
    deque<int> dq;  // 存索引，遞減隊列
    vector<int> result;
    for (int i = 0; i < nums.size(); i++) {
        // 移除窗口外的元素
        if (!dq.empty() && dq.front() <= i - k) {
            dq.pop_front();
        }
        // 保持遞減
        while (!dq.empty() && nums[dq.back()] < nums[i]) {
            dq.pop_back();
        }
        dq.push_back(i);
        // 記錄結果
        if (i >= k - 1) {
            result.push_back(nums[dq.front()]);
        }
    }
    return result;
}
```

#### 8.2.6 使用 multiset 維護中位數

```cpp
class MedianFinder {
    multiset<int> small, large;
public:
    void addNum(int num) {
        if (small.empty() || num <= *small.rbegin()) {
            small.insert(num);
        } else {
            large.insert(num);
        }
        // 平衡
        if (small.size() > large.size() + 1) {
            large.insert(*small.rbegin());
            small.erase(prev(small.end()));
        } else if (large.size() > small.size()) {
            small.insert(*large.begin());
            large.erase(large.begin());
        }
    }
    
    double findMedian() {
        if (small.size() > large.size()) {
            return *small.rbegin();
        }
        return (*small.rbegin() + *large.begin()) / 2.0;
    }
};
```

---

### 8.3 常見陷阱

```cpp
// 1. 迭代器失效
vector<int> v = {1, 2, 3, 4, 5};
for (auto it = v.begin(); it != v.end(); ) {
    if (*it % 2 == 0) {
        it = v.erase(it);  // erase 返回下一個有效迭代器
    } else {
        ++it;
    }
}

// 2. map 的 [] 會插入
map<int, int> m;
if (m[5] == 0) { ... }  // 錯！會插入 {5, 0}
if (m.count(5)) { ... } // 對

// 3. size() 返回 unsigned
vector<int> v;
for (int i = 0; i < (int)v.size() - 1; i++) { ... }  // 注意轉型

// 4. string 的 find 返回 npos
string s = "hello";
if (s.find("x") == string::npos) { ... }  // 正確
if (s.find("x") == -1) { ... }  // 危險！npos 是很大的正數

// 5. priority_queue 默認是最大堆
priority_queue<int> pq;  // 最大堆，top() 是最大值
```

---

## 9. 金融系統應用

### 9.1 訂單簿實現

```cpp
#include <map>
#include <unordered_map>

struct Order {
    int id;
    double price;
    int quantity;
    bool isBuy;
};

class OrderBook {
    // 買單：價格降序（map 默認升序，用 greater）
    map<double, int, greater<double>> bids;  // price -> total quantity
    // 賣單：價格升序
    map<double, int> asks;
    // 訂單查找
    unordered_map<int, Order> orders;
    
public:
    void addOrder(const Order& order) {
        orders[order.id] = order;
        if (order.isBuy) {
            bids[order.price] += order.quantity;
        } else {
            asks[order.price] += order.quantity;
        }
    }
    
    void cancelOrder(int orderId) {
        if (!orders.count(orderId)) return;
        Order& order = orders[orderId];
        if (order.isBuy) {
            bids[order.price] -= order.quantity;
            if (bids[order.price] == 0) bids.erase(order.price);
        } else {
            asks[order.price] -= order.quantity;
            if (asks[order.price] == 0) asks.erase(order.price);
        }
        orders.erase(orderId);
    }
    
    pair<double, double> getBestBidAsk() {
        double bestBid = bids.empty() ? 0 : bids.begin()->first;
        double bestAsk = asks.empty() ? 0 : asks.begin()->first;
        return {bestBid, bestAsk};
    }
};
```

---

### 9.2 時間序列數據

```cpp
#include <deque>
#include <numeric>

class MovingAverage {
    deque<double> window;
    int maxSize;
    double sum = 0;
    
public:
    MovingAverage(int size) : maxSize(size) {}
    
    double next(double val) {
        window.push_back(val);
        sum += val;
        if (window.size() > maxSize) {
            sum -= window.front();
            window.pop_front();
        }
        return sum / window.size();
    }
};

// 使用 multiset 維護滑動窗口中位數
class SlidingWindowMedian {
    multiset<double> small, large;
    int k;
    
    void balance() {
        if (small.size() > large.size() + 1) {
            large.insert(*small.rbegin());
            small.erase(prev(small.end()));
        } else if (large.size() > small.size()) {
            small.insert(*large.begin());
            large.erase(large.begin());
        }
    }
    
public:
    SlidingWindowMedian(int k) : k(k) {}
    
    void add(double num) {
        if (small.empty() || num <= *small.rbegin()) {
            small.insert(num);
        } else {
            large.insert(num);
        }
        balance();
    }
    
    void remove(double num) {
        if (num <= *small.rbegin()) {
            small.erase(small.find(num));
        } else {
            large.erase(large.find(num));
        }
        balance();
    }
    
    double getMedian() {
        if (small.size() > large.size()) {
            return *small.rbegin();
        }
        return (*small.rbegin() + *large.begin()) / 2.0;
    }
};
```

---

### 9.3 高效能計數器

```cpp
#include <unordered_map>
#include <chrono>

class RateLimiter {
    unordered_map<string, deque<long long>> requests;
    int maxRequests;
    long long windowMs;
    
public:
    RateLimiter(int max, long long windowMs) 
        : maxRequests(max), windowMs(windowMs) {}
    
    bool allow(const string& clientId) {
        auto now = chrono::duration_cast<chrono::milliseconds>(
            chrono::system_clock::now().time_since_epoch()
        ).count();
        
        auto& dq = requests[clientId];
        
        // 移除過期請求
        while (!dq.empty() && dq.front() < now - windowMs) {
            dq.pop_front();
        }
        
        if (dq.size() >= maxRequests) {
            return false;
        }
        
        dq.push_back(now);
        return true;
    }
};
```

---

## 10. 練習題

### 基礎練習

**練習 1: Vector 操作**
實現一個函數，接收一個 vector，刪除所有偶數，將剩餘元素平方後返回。

```cpp
vector<int> processVector(vector<int> v) {
    // 你的代碼
}

// 測試: {1,2,3,4,5,6} -> {1,9,25}
```


```cpp
vector<int> processVector(vector<int> v) {
    // 刪除偶數
    v.erase(remove_if(v.begin(), v.end(), [](int x) { return x % 2 == 0; }), v.end());
    // 平方
    transform(v.begin(), v.end(), v.begin(), [](int x) { return x * x; });
    return v;
}
```


---

**練習 2: Map 頻率統計**
統計字串中每個字符出現的次數，返回出現最多的字符。

```cpp
char mostFrequent(const string& s) {
    // 你的代碼
}
```


```cpp
char mostFrequent(const string& s) {
    unordered_map<char, int> freq;
    for (char c : s) freq[c]++;
    
    return max_element(freq.begin(), freq.end(),
        [](const auto& a, const auto& b) {
            return a.second < b.second;
        })->first;
}
```


---

**練習 3: Set 操作**
找出兩個陣列的交集，結果去重。

```cpp
vector<int> intersection(vector<int>& nums1, vector<int>& nums2) {
    // 你的代碼
}
```


```cpp
vector<int> intersection(vector<int>& nums1, vector<int>& nums2) {
    set<int> s1(nums1.begin(), nums1.end());
    set<int> s2(nums2.begin(), nums2.end());
    vector<int> result;
    set_intersection(s1.begin(), s1.end(), s2.begin(), s2.end(), 
                     back_inserter(result));
    return result;
}
```

---

**練習 4: Priority Queue**
合併 K 個有序陣列。

```cpp
vector<int> mergeKArrays(vector<vector<int>>& arrays) {
    // 你的代碼
}
```

```cpp
vector<int> mergeKArrays(vector<vector<int>>& arrays) {
    // {值, 陣列索引, 元素索引}
    auto cmp = [](const tuple<int,int,int>& a, const tuple<int,int,int>& b) {
        return get<0>(a) > get<0>(b);
    };
    priority_queue<tuple<int,int,int>, vector<tuple<int,int,int>>, decltype(cmp)> pq(cmp);
    
    // 初始化
    for (int i = 0; i < arrays.size(); i++) {
        if (!arrays[i].empty()) {
            pq.push({arrays[i][0], i, 0});
        }
    }
    
    vector<int> result;
    while (!pq.empty()) {
        auto [val, arrIdx, elemIdx] = pq.top();
        pq.pop();
        result.push_back(val);
        
        if (elemIdx + 1 < arrays[arrIdx].size()) {
            pq.push({arrays[arrIdx][elemIdx + 1], arrIdx, elemIdx + 1});
        }
    }
    
    return result;
}
```


---

### 進階練習

**練習 5: LRU Cache**
實現一個 LRU (Least Recently Used) 緩存。

```cpp
class LRUCache {
public:
    LRUCache(int capacity) {
        // 你的代碼
    }
    
    int get(int key) {
        // 你的代碼
    }
    
    void put(int key, int value) {
        // 你的代碼
    }
};
```


```cpp
class LRUCache {
    int capacity;
    list<pair<int, int>> cache;  // {key, value}
    unordered_map<int, list<pair<int, int>>::iterator> map;
    
public:
    LRUCache(int capacity) : capacity(capacity) {}
    
    int get(int key) {
        if (!map.count(key)) return -1;
        
        // 移到最前
        cache.splice(cache.begin(), cache, map[key]);
        return map[key]->second;
    }
    
    void put(int key, int value) {
        if (map.count(key)) {
            map[key]->second = value;
            cache.splice(cache.begin(), cache, map[key]);
            return;
        }
        
        if (cache.size() == capacity) {
            int oldKey = cache.back().first;
            cache.pop_back();
            map.erase(oldKey);
        }
        
        cache.push_front({key, value});
        map[key] = cache.begin();
    }
};
```

---

**練習 6: 前 K 個高頻元素**
找出陣列中出現頻率最高的 K 個元素。

```cpp
vector<int> topKFrequent(vector<int>& nums, int k) {
    // 你的代碼
}
```


```cpp
vector<int> topKFrequent(vector<int>& nums, int k) {
    unordered_map<int, int> freq;
    for (int num : nums) freq[num]++;
    
    // 最小堆
    auto cmp = [](const pair<int,int>& a, const pair<int,int>& b) {
        return a.second > b.second;
    };
    priority_queue<pair<int,int>, vector<pair<int,int>>, decltype(cmp)> pq(cmp);
    
    for (auto& [num, cnt] : freq) {
        pq.push({num, cnt});
        if (pq.size() > k) pq.pop();
    }
    
    vector<int> result;
    while (!pq.empty()) {
        result.push_back(pq.top().first);
        pq.pop();
    }
    return result;
}
```

---

**練習 7: 滑動窗口中位數**
計算每個大小為 K 的滑動窗口的中位數。

```cpp
vector<double> medianSlidingWindow(vector<int>& nums, int k) {
    // 你的代碼
}
```


```cpp
vector<double> medianSlidingWindow(vector<int>& nums, int k) {
    multiset<int> small, large;
    vector<double> result;
    
    auto balance = [&]() {
        while (small.size() > large.size() + 1) {
            large.insert(*small.rbegin());
            small.erase(prev(small.end()));
        }
        while (large.size() > small.size()) {
            small.insert(*large.begin());
            large.erase(large.begin());
        }
    };
    
    auto getMedian = [&]() -> double {
        if (k % 2) return *small.rbegin();
        return ((double)*small.rbegin() + *large.begin()) / 2.0;
    };
    
    for (int i = 0; i < nums.size(); i++) {
        // 添加
        if (small.empty() || nums[i] <= *small.rbegin()) {
            small.insert(nums[i]);
        } else {
            large.insert(nums[i]);
        }
        balance();
        
        // 移除
        if (i >= k) {
            int toRemove = nums[i - k];
            if (toRemove <= *small.rbegin()) {
                small.erase(small.find(toRemove));
            } else {
                large.erase(large.find(toRemove));
            }
            balance();
        }
        
        // 記錄
        if (i >= k - 1) {
            result.push_back(getMedian());
        }
    }
    
    return result;
}
```


---

**練習 8: 股票價格跨度**
計算股票連續小於等於今日價格的天數。

```cpp
class StockSpanner {
public:
    StockSpanner() {
        // 你的代碼
    }
    
    int next(int price) {
        // 你的代碼
    }
};
```


```cpp
class StockSpanner {
    stack<pair<int, int>> stk;  // {price, span}
    
public:
    StockSpanner() {}
    
    int next(int price) {
        int span = 1;
        while (!stk.empty() && stk.top().first <= price) {
            span += stk.top().second;
            stk.pop();
        }
        stk.push({price, span});
        return span;
    }
};
```


---

**練習 9: 設計推特**
設計一個簡化版 Twitter。

```cpp
class Twitter {
public:
    Twitter() {}
    
    void postTweet(int userId, int tweetId) {}
    
    vector<int> getNewsFeed(int userId) {}  // 最近 10 條
    
    void follow(int followerId, int followeeId) {}
    
    void unfollow(int followerId, int followeeId) {}
};
```


```cpp
class Twitter {
    int time = 0;
    unordered_map<int, vector<pair<int, int>>> tweets;  // userId -> {time, tweetId}
    unordered_map<int, unordered_set<int>> following;   // userId -> followees
    
public:
    Twitter() {}
    
    void postTweet(int userId, int tweetId) {
        tweets[userId].push_back({time++, tweetId});
    }
    
    vector<int> getNewsFeed(int userId) {
        auto cmp = [](const pair<int,int>& a, const pair<int,int>& b) {
            return a.first < b.first;
        };
        priority_queue<pair<int,int>, vector<pair<int,int>>, decltype(cmp)> pq(cmp);
        
        // 自己的推文
        for (auto& [t, id] : tweets[userId]) {
            pq.push({t, id});
        }
        // 關注者的推文
        for (int followee : following[userId]) {
            for (auto& [t, id] : tweets[followee]) {
                pq.push({t, id});
            }
        }
        
        vector<int> result;
        for (int i = 0; i < 10 && !pq.empty(); i++) {
            result.push_back(pq.top().second);
            pq.pop();
        }
        return result;
    }
    
    void follow(int followerId, int followeeId) {
        if (followerId != followeeId) {
            following[followerId].insert(followeeId);
        }
    }
    
    void unfollow(int followerId, int followeeId) {
        following[followerId].erase(followeeId);
    }
};
```

---

**練習 10: 時間戳 Key-Value Store**
設計一個支持時間戳版本的 Key-Value 存儲。

```cpp
class TimeMap {
public:
    TimeMap() {}
    
    void set(string key, string value, int timestamp) {}
    
    string get(string key, int timestamp) {}  // 返回 <= timestamp 的最新值
};
```


```cpp
class TimeMap {
    unordered_map<string, vector<pair<int, string>>> store;
    
public:
    TimeMap() {}
    
    void set(string key, string value, int timestamp) {
        store[key].push_back({timestamp, value});
    }
    
    string get(string key, int timestamp) {
        if (!store.count(key)) return "";
        
        auto& vec = store[key];
        // 二分查找
        int l = 0, r = vec.size();
        while (l < r) {
            int mid = l + (r - l) / 2;
            if (vec[mid].first <= timestamp) {
                l = mid + 1;
            } else {
                r = mid;
            }
        }
        
        if (l == 0) return "";
        return vec[l - 1].second;
    }
};
```

---

## 總結

### 容器時間複雜度對比

| 容器 | 訪問 | 查找 | 插入 | 刪除 |
|------|------|------|------|------|
| vector | O(1) | O(n) | O(n)* | O(n) |
| deque | O(1) | O(n) | O(n)* | O(n) |
| list | O(n) | O(n) | O(1) | O(1) |
| set/map | O(log n) | O(log n) | O(log n) | O(log n) |
| unordered_set/map | O(1)** | O(1)** | O(1)** | O(1)** |

\* 尾部插入均攤 O(1)  
\*\* 平均情況，最壞 O(n)

### 選擇建議

1. **默認使用 vector**，除非有特殊需求
2. **需要快速查找用 unordered_set/map**，需要排序用 set/map
3. **頻繁中間插入刪除用 list**
4. **堆/優先隊列用 priority_queue**
5. **固定大小用 array**

掌握這些 STL 容器和算法，你就能輕鬆應對 Leetcode 刷題和金融系統開發中的各種數據結構需求！
