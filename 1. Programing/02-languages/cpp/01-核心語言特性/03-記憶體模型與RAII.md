# 記憶體模型與 RAII

> **學習優先級**: ⭐⭐⭐ 必看
>
> RAII 是現代 C++ 的核心原則,正確的資源管理直接影響系統穩定性與性能。

---

## 目錄

1. [RAII 核心理念](#1-raii-核心理念)
2. [物件生命週期與記憶體佈局](#2-物件生命週期與記憶體佈局)
3. [值類別與移動語義基礎](#3-值類別與移動語義基礎)
4. [智能指針深度解析](#4-智能指針深度解析)
5. [高頻交易中的記憶體管理策略](#5-高頻交易中的記憶體管理策略)
6. [實戰檢查清單](#6-實戰檢查清單)

---

## 1. RAII 核心理念

### 1.1 什麼是 RAII?

**RAII (Resource Acquisition Is Initialization)**: 資源獲取即初始化,資源釋放即解構。

**核心思想**:

- 在建構函式中獲取資源
- 在解構函式中釋放資源
- 利用 C++ 的自動解構機制保證資源釋放

```cpp
class FileHandler {
    int fd_;
public:
    explicit FileHandler(const char* path)
        : fd_(::open(path, O_RDONLY)) {
        if (fd_ < 0) throw std::runtime_error("open failed");
    }

    ~FileHandler() {
        if (fd_ >= 0) ::close(fd_);
    }

    // 禁止拷貝,允許移動
    FileHandler(const FileHandler&) = delete;
    FileHandler& operator=(const FileHandler&) = delete;

    FileHandler(FileHandler&& other) noexcept
        : fd_(std::exchange(other.fd_, -1)) {}

    FileHandler& operator=(FileHandler&& other) noexcept {
        if (this != &other) {
            if (fd_ >= 0) ::close(fd_);
            fd_ = std::exchange(other.fd_, -1);
        }
        return *this;
    }

    int get() const { return fd_; }
};

void process_file() {
    FileHandler fh("/path/to/file");
    // 即使發生異常,解構函數也會被調用
    // 無需手動 close(fd)
} // 自動關閉文件
```

### 1.2 RAII 的優勢

**與手動管理對比**:

```cpp
// ❌ 手動管理 (容易洩漏)
void bad_example() {
    int* p = new int(42);
    process(p);
    delete p;  // 如果 process 拋出異常,不會執行!
}

// ✅ RAII (自動管理)
void good_example() {
    auto p = std::make_unique<int>(42);
    process(p.get());
    // 離開作用域自動釋放,即使異常
}
```

**與其他語言對比**:

```csharp
// C# 使用 IDisposable + using
using (var file = File.OpenRead("path")) {
    // 處理文件
} // 自動調用 Dispose()
```

```python
# Python 使用 context manager
with open("path") as file:
    # 處理文件
    pass  # 自動關閉
```

### 1.3 高頻交易應用案例

```cpp
// 交易會話管理
class TradingSession {
    std::unique_ptr<MarketDataConnection> md_conn_;
    std::unique_ptr<OrderConnection> order_conn_;

public:
    TradingSession(const Config& cfg)
        : md_conn_(std::make_unique<MarketDataConnection>(cfg.md_endpoint))
        , order_conn_(std::make_unique<OrderConnection>(cfg.order_endpoint)) {
        // 連接建立失敗會自動清理已分配資源
    }

    // 解構時自動斷開所有連接
    ~TradingSession() = default;
};

// 市場數據訂閱管理
class MarketDataSubscription {
    struct SubscriptionDeleter {
        void operator()(Subscription* sub) const {
            if (sub) {
                sub->unsubscribe();  // 先取消訂閱
                delete sub;
            }
        }
    };

    std::unique_ptr<Subscription, SubscriptionDeleter> sub_;

public:
    explicit MarketDataSubscription(const Symbol& symbol)
        : sub_(SubscriptionManager::subscribe(symbol)) {}
};
```

---

## 2. 物件生命週期與記憶體佈局

### 2.1 物件大小與對齊

```cpp
#include <iostream>

struct Empty {};  // 仍佔用 1 byte (佔位符)

struct Aligned {
    char c;      // 1 byte
    // 3 bytes padding
    int i;       // 4 bytes
    char c2;     // 1 byte
    // 3 bytes padding
}; // sizeof = 12 bytes

struct Optimized {
    int i;       // 4 bytes
    char c;      // 1 byte
    char c2;     // 1 byte
    // 2 bytes padding
}; // sizeof = 8 bytes

int main() {
    std::cout << sizeof(Empty) << '\n';      // 1
    std::cout << sizeof(Aligned) << '\n';    // 16
    std::cout << sizeof(Optimized) << '\n';  // 8
}
```

**高頻交易要點**:

- 小心結構體填充 (padding),影響 cache line 利用率
- 通常一個 cache line 為 64 bytes
- 熱路徑數據結構應盡量緊湊
- 將常用欄位放在一起,提高 cache 命中率

**Cache Line 對齊範例**:

```cpp
// ❌ 不好: False Sharing
struct BadCounter {
    std::atomic<int> counter1;  // 可能在同一 cache line
    std::atomic<int> counter2;  // 多執行緒競爭同一 cache line
};

// ✅ 好: 避免 False Sharing
struct alignas(64) GoodCounter {
    alignas(64) std::atomic<int> counter1;
    alignas(64) std::atomic<int> counter2;
};
```

### 2.2 物件生命週期

```cpp
// 自動儲存期 (Automatic Storage Duration)
void function() {
    int x = 42;  // 在堆疊上分配
    // x 在函數結束時自動銷毀
}

// 靜態儲存期 (Static Storage Duration)
int global = 42;  // 程式啟動時初始化,結束時銷毀

void function() {
    static int counter = 0;  // 第一次調用時初始化
    ++counter;
}

// 動態儲存期 (Dynamic Storage Duration)
void function() {
    int* p = new int(42);  // 在堆上分配
    delete p;  // 必須手動釋放
}

// 執行緒儲存期 (Thread Storage Duration) - C++11
thread_local int tls_var = 0;  // 每個執行緒一份
```

---

## 3. 值類別與移動語義基礎

### 3.1 值類別 (Value Categories)

C++11 引入五種值類別:

```cpp
int x = 10;
int& lref = x;           // lvalue reference
int&& rref = 10;         // rvalue reference

// lvalue: 有名字,可取地址
std::string s1 = "hello";

// prvalue (pure rvalue): 臨時對象
std::string{"world"};
42;

// xvalue (expiring value): 即將銷毀的對象
std::move(s1);
std::string{"temp"}.substr(0, 2);
```

**記憶技巧**:

1. **有名字的為左值**,沒有名字的為右值
2. `A&& arg` 為一個右值引用**類型**,表示可以綁定右值,但因為 arg 有名字故本身為**左值**

### 3.2 移動語義基礎

**為何需要移動?**

```cpp
// 低效的拷貝
std::vector<int> create_large_vector() {
    std::vector<int> v(1'000'000);
    // ... 填充數據
    return v;  // C++11 前: 拷貝整個 vector
}

// C++11 後: 自動移動 (RVO/NRVO)
std::vector<int> v = create_large_vector();  // O(1) 移動,非 O(n) 拷貝
```

**移動語義最佳實踐**:

```cpp
// 1. 按值返回 (依賴 RVO/NRVO)
std::vector<Trade> get_trades() {
    std::vector<Trade> trades;
    // ... 填充
    return trades;  // 不要 std::move(trades)!
}

// 2. 函數參數: sink 參數使用按值傳遞
class OrderManager {
    std::vector<Order> pending_orders_;
public:
    // sink 參數: 支持移動與拷貝
    void add_order(Order order) {
        pending_orders_.push_back(std::move(order));
    }
};

// 使用方式
Order o = create_order();
manager.add_order(std::move(o));  // 移動
manager.add_order(create_order()); // 移動 (臨時對象)
```

---

## 4. 智能指針深度解析

### 4.1 所有權模型對比

| 指針類型              | 所有權             | 開銷   | 適用場景       |
| ----------------- | --------------- | ---- | ---------- |
| `std::unique_ptr` | 唯一所有權           | 零開銷  | 默認選擇,獨佔資源  |
| `std::shared_ptr` | 共享所有權           | 原子計數 | 多個所有者,計數管理 |
| `std::weak_ptr`   | 非所有權            | 無    | 打破循環引用     |
| 原始指針 (`T*`)       | 借用 (non-owning) | 零開銷  | 觀察者,不負責釋放  |

### 4.2 unique_ptr: 零開銷抽象

```cpp
#include <memory>

// 創建方式
auto p1 = std::make_unique<Order>(/*args*/);  // C++14 (推薦)
std::unique_ptr<Order> p2(new Order(/*args*/)); // C++11

// 訪問
*p1 = new_value;
p1->method();

// 移動 (轉移所有權)
auto p3 = std::move(p1);  // p1 變為 nullptr
// *p1;  // Error! p1 已無效

// 釋放並獲取裸指針
Order* raw = p3.release();  // p3 變為 nullptr,不會 delete
delete raw;  // 需手動刪除

// 重置
p3.reset(new Order(100));  // 刪除舊值,持有新值
p3.reset();  // 刪除並變為 nullptr
```

**自定義刪除器**:

```cpp
// 文件描述符管理
auto fd_deleter = [](int* pfd) {
    if (pfd && *pfd >= 0) ::close(*pfd);
    delete pfd;
};

std::unique_ptr<int, decltype(fd_deleter)>
    fd_ptr(new int(::open("file", O_RDONLY)), fd_deleter);

// 數組版本
auto arr = std::make_unique<int[]>(10);  // C++14
arr[0] = 42;
// 自動調用 delete[]
```

### 4.3 shared_ptr: 引用計數的代價

```cpp
// 創建
auto p1 = std::make_shared<int>(42);  // 推薦 (單次分配)
std::shared_ptr<int> p2(new int(42)); // 兩次分配

// 引用計數
std::cout << p1.use_count() << "\n";  // 1

// 共享
auto p3 = p1;  // 引用計數 +1
std::cout << p1.use_count() << "\n";  // 2

{
    auto p4 = p1;  // 引用計數 +1
    std::cout << p1.use_count() << "\n";  // 3
}  // p4 銷毀,計數 -1

std::cout << p1.use_count() << "\n";  // 2

// 重置
p3.reset();  // 計數 -1
std::cout << p1.use_count() << "\n";  // 1
```

**性能考量**:

```cpp
// 引用計數開銷:
// 1. 額外記憶體: 控制塊 (control block) 16-32 bytes
// 2. 原子操作: 計數增減是 thread-safe 的
// 3. cache 友好性差: 控制塊與對象分離

// ❌ 高頻交易中避免過度使用 shared_ptr
void process_orders(std::shared_ptr<OrderBook> book) {
    // 每次調用都增減引用計數 (atomic++)
}

// ✅ 優先使用引用或裸指針
void process_orders(const OrderBook& book) {
    // 零開銷,僅借用
}
```

### 4.4 weak_ptr: 解決循環引用

```cpp
class Portfolio;

class Position {
    std::weak_ptr<Portfolio> portfolio_;  // 不增加引用計數
public:
    void check_risk() {
        if (auto port = portfolio_.lock()) {  // 安全升級為 shared_ptr
            // 使用 port
        } else {
            // portfolio 已被銷毀
        }
    }
};

class Portfolio {
    std::vector<std::shared_ptr<Position>> positions_;
};

// 使用 weak_ptr
std::weak_ptr<Node> weak = n1;

// 檢查是否存活
if (auto shared = weak.lock()) {  // 嘗試提升為 shared_ptr
    // n1 仍存活
    shared->next;
} else {
    // n1 已銷毀
}

// 或檢查 expired
if (weak.expired()) {
    // 已銷毀
}
```

### 4.5 智能指針最佳實踐

```cpp
// ✅ 工廠函數返回 unique_ptr
std::unique_ptr<Base> create_object(int type) {
    if (type == 1) return std::make_unique<Derived1>();
    return std::make_unique<Derived2>();
}

// ✅ 函數接受裸指針或引用 (不轉移所有權)
void process(const Widget& w);  // 推薦
void process(Widget* w);        // 可空時使用

// ❌ 不要接受智能指針 (除非要轉移/共享所有權)
void bad(std::unique_ptr<Widget> w);  // 強制轉移
void bad(std::shared_ptr<Widget> w);  // 強制共享

// ✅ 只在需要時轉移所有權
void take_ownership(std::unique_ptr<Widget> w);
```

---

## 5. 高頻交易中的記憶體管理策略

### 5.1 物件池 (Object Pool)

**概念**: 預先建立好多個物件(相同的),讓調用方取得然後使用完自動回收但不釋放(只還原狀態)

**目的**: 避免頻繁分配/釋放，減少內存碎片化

**場景**: 連線池、訂單物件池

```cpp
template <typename T> class ObjectPool {
public:
  ObjectPool(size_t initial_size) {
    pool_.reserve(initial_size);
    available_.reserve(initial_size);

    for (size_t i = 0; i < initial_size; ++i) {
      auto obj = std::make_unique<T>();
      available_.push_back(obj.get());
      pool_.push_back(std::move(obj));
    }
  }

  T *aquire() {
    if (available_.empty()) {
      auto obj = std::make_unique<T>();
      T *ptr = obj.get();
      pool_.push_back(std::move(obj));
      return ptr;
    }

    T *obj = available_.back();
    available_.pop_back();
    return obj;
  }

  void release(T *obj) {
    obj->reset();
    available_.push_back(obj);
  }

private:
  std::vector<std::unique_ptr<T>> pool_; // 持有 ownership
  std::vector<T *> available_;           // Tracking
};

// Tracking Object
// 物件池最大的問題就是使用完之後忘記歸還，所以在該物件離開作用域，自動的返還給物件池
template <typename T> class PooledObject {
public:
  PooledObject(ObjectPool<T> &pool) : obj_(pool.aquire()), pool_(&pool) {}

  ~PooledObject() {
    if (obj_)
      pool_->release(obj_);
  }

  PooledObject(const PooledObject &) = delete;
  PooledObject &operator=(const PooledObject &) = delete;

  PooledObject(PooledObject &&other)
      : obj_(std::exchange(other.obj_, nullptr)),
        pool_(std::exchange(other.pool_, nullptr)) {}
  PooledObject &operator=(PooledObject &&other) {
    if (this != &other) {
      if (obj_) {
        pool_->release(obj_);
      }

      obj_ = std::exchange(other.obj_, nullptr);
      pool_ = std::exchange(other.pool_, nullptr);
    }

    return *this;
  }

  T *operator->() { return obj_; }
  T &operator*() { return *obj_; }

private:
  T *obj_;
  ObjectPool<T> *pool_;
};
```

### 5.2 環形緩衝 (Ring Buffer)

```cpp
template<typename T, size_t N>
class RingBuffer {
    alignas(64) std::array<T, N> buffer_;  // cache line 對齊
    alignas(64) std::atomic<size_t> write_pos_{0};
    alignas(64) std::atomic<size_t> read_pos_{0};

public:
    bool try_push(T&& item) {
        const size_t current_write = write_pos_.load(std::memory_order_relaxed);
        const size_t next_write = (current_write + 1) % N;

        if (next_write == read_pos_.load(std::memory_order_acquire)) {
            return false;  // 滿了
        }

        buffer_[current_write] = std::move(item);
        write_pos_.store(next_write, std::memory_order_release);
        return true;
    }

    bool try_pop(T& item) {
        const size_t current_read = read_pos_.load(std::memory_order_relaxed);

        if (current_read == write_pos_.load(std::memory_order_acquire)) {
            return false;  // 空的
        }

        item = std::move(buffer_[current_read]);
        read_pos_.store((current_read + 1) % N, std::memory_order_release);
        return true;
    }
};
```

### 5.3 預分配與就地構造

```cpp
class OrderBook {
    // 預分配,避免運行時分配
    std::array<Order, 10000> orders_;
    size_t size_ = 0;

public:
    template<typename... Args>
    Order& emplace_order(Args&&... args) {
        // 就地構造,避免臨時對象
        new (&orders_[size_]) Order(std::forward<Args>(args)...);
        return orders_[size_++];
    }
};
```

### 5.4 自定義記憶體分配器

```cpp
// 簡單的線性分配器 (適合短生命週期物件)
class LinearAllocator {
    char* buffer_;
    size_t size_;
    size_t offset_ = 0;

public:
    explicit LinearAllocator(size_t size)
        : buffer_(new char[size]), size_(size) {}

    ~LinearAllocator() { delete[] buffer_; }

    void* allocate(size_t n, size_t alignment = alignof(std::max_align_t)) {
        size_t padding = (alignment - (offset_ % alignment)) % alignment;
        size_t new_offset = offset_ + padding + n;

        if (new_offset > size_) {
            throw std::bad_alloc();
        }

        void* ptr = buffer_ + offset_ + padding;
        offset_ = new_offset;
        return ptr;
    }

    void reset() { offset_ = 0; }  // 重置,不釋放記憶體
};
```

---

## 6. 實戰檢查清單

### 6.1 類設計檢查

- [ ] 是否需要自定義解構函數? → 考慮 Rule of Five
- [ ] 是否能使用 Rule of Zero? (優先選擇)
- [ ] 移動操作是否標記為 `noexcept`?
- [ ] 是否使用 `= delete` 禁止不必要的操作?
- [ ] 成員變量順序是否優化 (減少 padding)?

### 6.2 智能指針選擇

- [ ] 默認使用 `std::unique_ptr`
- [ ] 確實需要共享所有權才用 `std::shared_ptr`
- [ ] 使用 `std::make_unique`/`std::make_shared` (異常安全)
- [ ] 避免從 `this` 創建 `shared_ptr` (使用 `enable_shared_from_this`)

### 6.3 高頻交易優化

- [ ] 熱路徑避免動態分配
- [ ] 使用物件池管理頻繁創建/銷毀的對象
- [ ] 數據結構 cache line 對齊
- [ ] 避免不必要的 `shared_ptr` 拷貝
- [ ] 預分配容器容量 (`reserve`)

---

## 參考資料

1. Meyers, Scott. _Effective Modern C++_. O'Reilly, 2014.
2. Stroustrup, Bjarne. _A Tour of C++ (2nd Edition)_. Addison-Wesley, 2018.
3. [C++ Core Guidelines](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines)
4. [Understanding Move Semantics](https://www.cprogramming.com/c++11/rvalue-references-and-move-semantics-in-c++11.html)
5. Alexandrescu, Andrei. _Modern C++ Design_. Addison-Wesley, 2001.
6. [CppCon Talks on Memory Management](https://www.youtube.com/user/CppCon)
