# Day 1: C++ 核心特性 (Part 1)

> 專注於現代 C++ 高性能特性:智能指標、移動語義、Lambda 與模板進階

---

## 1.1 智能指標 (Smart Pointers)

### 核心概念

智能指標是 RAII (Resource Acquisition Is Initialization) 原則的典型應用,通過物件生命週期自動管理記憶體。

### 1.1.1 `unique_ptr` - 獨佔所有權

**特性:**
- 獨佔資源所有權
- 零運行時開銷 (相對於裸指標)
- 不可複製,只能移動
- 自動釋放資源

**基本使用:**

```cpp
#include <memory>
#include <iostream>

class Resource {
public:
    Resource(int id) : id_(id) {
        std::cout << "Resource " << id_ << " created\n";
    }
    ~Resource() {
        std::cout << "Resource " << id_ << " destroyed\n";
    }
    void use() { std::cout << "Using resource " << id_ << "\n"; }
private:
    int id_;
};

int main() {
    // 使用 make_unique (C++14)
    auto ptr1 = std::make_unique<Resource>(1);
    ptr1->use();
    
    // 移動語義
    auto ptr2 = std::move(ptr1);  // ptr1 現在為 nullptr
    if (!ptr1) {
        std::cout << "ptr1 is nullptr\n";
    }
    ptr2->use();
    
    // 自動釋放
}  // ptr2 超出作用域,Resource 自動銷毀
```

**自定義刪除器:**

```cpp
// 用於管理非記憶體資源
struct FileDeleter {
    void operator()(FILE* fp) const {
        if (fp) {
            std::cout << "Closing file\n";
            std::fclose(fp);
        }
    }
};

// 使用自定義刪除器
std::unique_ptr<FILE, FileDeleter> fp(std::fopen("data.txt", "r"));

// Lambda 作為刪除器
auto deleter = [](Resource* r) {
    std::cout << "Custom delete\n";
    delete r;
};
std::unique_ptr<Resource, decltype(deleter)> ptr(new Resource(5), deleter);
```

### 1.1.2 `shared_ptr` - 共享所有權

**特性:**
- 引用計數 (Reference Counting)
- 多個指標可共享同一資源
- 最後一個 `shared_ptr` 銷毀時釋放資源
- 執行緒安全的引用計數 (原子操作)

**基本使用:**

```cpp
void example_shared_ptr() {
    auto sp1 = std::make_shared<Resource>(10);
    std::cout << "use_count: " << sp1.use_count() << "\n";  // 1
    
    {
        auto sp2 = sp1;  // 複製,引用計數增加
        std::cout << "use_count: " << sp1.use_count() << "\n";  // 2
        sp2->use();
    }  // sp2 銷毀,引用計數減少
    
    std::cout << "use_count: " << sp1.use_count() << "\n";  // 1
}  // sp1 銷毀,引用計數歸零,Resource 被釋放
```

**性能陷阱:**

```cpp
// 問題:兩次記憶體分配
std::shared_ptr<Resource> sp1(new Resource(1));

// 正確:一次分配 (控制塊 + 物件)
auto sp2 = std::make_shared<Resource>(2);
```

`make_shared` 的優勢:
1. 只分配一次記憶體 (控制塊和物件連續)
2. 減少 cache miss
3. 異常安全

### 1.1.3 `weak_ptr` - 弱引用

**用途:**
- 打破循環引用
- 觀察 `shared_ptr` 但不增加引用計數
- 需要時升級為 `shared_ptr`

**範例:打破循環引用**

```cpp
class Node {
public:
    std::shared_ptr<Node> next;     // 強引用
    std::weak_ptr<Node> prev;       // 弱引用,避免循環
    
    ~Node() { std::cout << "Node destroyed\n"; }
};

void circular_reference_example() {
    auto node1 = std::make_shared<Node>();
    auto node2 = std::make_shared<Node>();
    
    node1->next = node2;
    node2->prev = node1;  // 弱引用,不會形成循環
}  // node1 和 node2 都能正確釋放
```

**使用 `weak_ptr`:**

```cpp
void use_weak_ptr() {
    std::weak_ptr<Resource> wp;
    
    {
        auto sp = std::make_shared<Resource>(20);
        wp = sp;  // weak_ptr 指向 shared_ptr
        
        // 使用 lock() 升級為 shared_ptr
        if (auto locked = wp.lock()) {
            locked->use();
            std::cout << "Resource still alive\n";
        }
    }  // sp 銷毀,Resource 釋放
    
    // 嘗試訪問已釋放的資源
    if (auto locked = wp.lock()) {
        locked->use();
    } else {
        std::cout << "Resource already destroyed\n";
    }
}
```

### 1.1.4 智能指標性能考量

**高頻交易場景的選擇:**

```cpp
// ✓ 推薦:使用 unique_ptr,零開銷
std::unique_ptr<Order> order = std::make_unique<Order>();

// ✗ 避免:shared_ptr 原子操作有開銷
std::shared_ptr<Order> order = std::make_shared<Order>();  // 原子遞增/遞減

// ✓✓ 最佳:物件池 + 裸指標 (完全避免動態分配)
Order* order = order_pool.allocate();
// ... 使用後返回池
order_pool.deallocate(order);
```

---

## 1.2 移動語義與右值引用 (Move Semantics)

### 核心概念

移動語義允許資源的"轉移"而非"複製",在高性能場景中至關重要。

### 1.2.1 左值 (Lvalue) vs 右值 (Rvalue)

**定義:**
- **左值 (Lvalue)**: **有名字的物件**,可以取地址
- **右值 (Rvalue)**: 臨時物件,不能取地址

```cpp
int x = 10;       // x 是左值
int y = x + 5;    // x+5 是右值 (臨時結果)

int& lr = x;      // 左值引用
// int& lr2 = 10; // 錯誤:不能綁定右值

int&& rr = 10;    // 右值引用
int&& rr2 = x + 5;// 右值引用綁定臨時物件
```

### 1.2.2 移動建構子與移動賦值運算子

**傳統複製 vs 移動:**

```cpp
class Buffer {
private:
    char* data_;
    size_t size_;

public:
    // 建構子
    Buffer(size_t size) : size_(size), data_(new char[size]) {
        std::cout << "Constructor: allocate " << size << " bytes\n";
    }
    
    // 解構子
    ~Buffer() {
        std::cout << "Destructor: free memory\n";
        delete[] data_;
    }
    
    // 複製建構子 (深拷貝)
    Buffer(Buffer &&other) noexcept
      : data_(std::exchange(other.data_, nullptr)),
        size_(std::exchange(other.size_, 0)) {
        std::cout << "Copy constructor: allocate and copy " << size_ << " bytes\n";
        std::memcpy(data_, other.data_, size_);
    }
    
    // 移動建構子 (轉移所有權)
    Buffer(Buffer&& other) noexcept 
        : size_(other.size_), data_(other.data_) {
        std::cout << "Move constructor: transfer ownership\n";
        // 重置來源物件
        other.data_ = nullptr;
        other.size_ = 0;
    }
    
    // 複製賦值運算子
    // 方式一: 非異常安全, 因為我們已經修改 this 內容，在對空間進行分配
    // 若分配失敗 this 也已經失效
	// Buffer &operator=(const Buffer &other) {
	//   if (this != &other) {
	//     delete[] data_;
	//     size_ = other.size_;
	//     data_ = new char[size_];
	//     std::memcpy(data_, other.data_, size_);
	//   }
	//
	//   return *this;
	// }
	// 方式二: 透過拷貝構造函數在參數傳遞時拷貝, 確認構造成功才交換
	Buffer &operator=(Buffer other) {
		Buffer tmp(other);
		swap(tmp);
		
		return *this;
	}


    
    // 移動賦值運算子
    Buffer& operator=(Buffer&& other) noexcept {
        if (this != &other) {
            std::cout << "Move assignment\n";
            delete[] data_;
            
            data_ = std::exchange(other.data_, nullptr);
            size_ = std::exchange(other.size_, 0);
        }
        return *this;
    }
    
    void swap(Buffer &other) noexcept {
        std::swap(data_, other.data_);
        std::swap(size_, other.size_);
    }
    
    size_t size() const { return size_; }
};

void move_semantics_demo() {
    Buffer b1(1024);                    // Constructor
    Buffer b2 = b1;                     // Copy constructor (深拷貝)
    Buffer b3 = std::move(b1);          // Move constructor (轉移)
    
    Buffer b4(512);
    b4 = b2;                            // Copy assignment
    b4 = std::move(b2);                 // Move assignment
}
```

**性能對比:**
- **複製**: 分配新記憶體 + 複製數據 → O(n)
- **移動**: 轉移指標 → O(1)

### 1.2.3 `std::move` 與 `std::forward`

**`std::move` - 強制轉換為右值引用:**

```cpp
template<typename T>
void process(T&& value) {
    // value 在函數內是左值 (有名字)
    // 需要 std::move 將其轉為右值
    Buffer b = std::move(value);  // 觸發移動建構子
}

void example_move() {
    Buffer b1(1024);
    process(std::move(b1));  // 顯式移動
    // b1 現在處於"已移動"狀態,不應再使用
}
```

**`std::forward` - 完美轉發:**

```cpp
// 萬能引用 (Universal Reference) T&&
template<typename T>
void wrapper(T&& arg) {
    // 保持 arg 的左值/右值屬性
    process(std::forward<T>(arg));
}

void example_forward() {
    Buffer b(1024);
    wrapper(b);              // 左值,轉發為左值
    wrapper(Buffer(512));    // 右值,轉發為右值
}
```

### 1.2.4 RVO/NRVO 優化

**返回值優化 (Return Value Optimization):**

```cpp
Buffer create_buffer(size_t size) {
    Buffer b(size);
    // ... 操作
    return b;  // 不會調用移動/複製建構子 (NRVO)
}

void rvo_demo() {
    Buffer b = create_buffer(1024);  // 直接在 b 的位置構造
}
```

**注意**: 不要對返回值使用 `std::move`,會阻礙 RVO!

```cpp
// ✗ 錯誤:阻礙 RVO
Buffer create_buffer_wrong(size_t size) {
    Buffer b(size);
    return std::move(b);  // 不要這樣做!
}

// ✓ 正確:讓編譯器進行 RVO
Buffer create_buffer_correct(size_t size) {
    Buffer b(size);
    return b;
}
```

### 1.2.5 五法則 vs 零法則

**五法則 (Rule of Five):**

如果需要自定義以下任一函數,通常需要全部定義:
1. 解構子
2. 複製建構子
3. 複製賦值運算子
4. 移動建構子
5. 移動賦值運算子

**零法則 (Rule of Zero):**

優先使用標準容器和智能指標,避免手動管理資源。

```cpp
// ✓ 零法則:不需要自定義特殊成員函數
class Message {
private:
    std::string content_;
    std::vector<char> data_;
    std::unique_ptr<Payload> payload_;
public:
    // 編譯器自動生成正確的複製/移動函數
};
```

---

## 1.3 Lambda 表達式

### 核心概念

Lambda 是匿名函數物件,在高頻交易中常用於回調、事件處理。

### 1.3.1 基本語法

```cpp
// [捕獲列表](參數列表) -> 返回類型 { 函數體 }
auto lambda = [](int x, int y) -> int {
    return x + y;
};

int result = lambda(3, 4);  // 7

// 返回類型自動推導
auto lambda2 = [](int x) { return x * 2; };
```

### 1.3.2 捕獲機制

**值捕獲 `[=]`:**

```cpp
int multiplier = 10;
auto lambda = [multiplier](int x) {
    return x * multiplier;  // 複製 multiplier
};
// 修改外部變數不影響 lambda
multiplier = 20;
std::cout << lambda(5);  // 50 (使用捕獲時的值 10)
```

**引用捕獲 `[&]`:**

```cpp
int counter = 0;
auto lambda = [&counter]() {
    counter++;  // 修改外部變數
};
lambda();
lambda();
std::cout << counter;  // 2
```

**混合捕獲:**

```cpp
int a = 10, b = 20;
auto lambda = [a, &b](int x) {
    // a 值捕獲 (唯讀)
    // b 引用捕獲 (可修改)
    b += x;
    return a * b;
};
```

**`this` 捕獲:**

```cpp
class EventHandler {
private:
    int id_;
public:
    EventHandler(int id) : id_(id) {}
    
    void register_callback() {
        // [this] 捕獲當前物件指標
        auto callback = [this](const Event& e) {
            std::cout << "Handler " << id_ << " received event\n";
            process(e);
        };
        event_system.add_listener(callback);
    }
    
    // C++17: [*this] 捕獲物件副本
    void register_callback_safe() {
        auto callback = [*this](const Event& e) {
            // 即使原物件銷毀,lambda 仍持有副本
            std::cout << "Handler " << id_ << " received event\n";
        };
    }
    
    void process(const Event& e) { /* ... */ }
};
```

### 1.3.3 初始化捕獲 (C++14)

```cpp
auto lambda = [ptr = std::make_unique<Resource>(10)]() {
    ptr->use();
};

// 移動捕獲
std::unique_ptr<Buffer> buffer = std::make_unique<Buffer>(1024);
auto lambda2 = [buf = std::move(buffer)]() {
    // buffer 已移動,lambda 擁有所有權
};
```

### 1.3.4 泛型 Lambda (C++14)

```cpp
auto generic_lambda = [](auto x, auto y) {
    return x + y;
};

std::cout << generic_lambda(1, 2);          // 3 (int)
std::cout << generic_lambda(1.5, 2.5);      // 4.0 (double)
std::cout << generic_lambda(std::string("Hello"), " World");  // "Hello World"
```

### 1.3.5 性能考量: `std::function` vs 裸 Lambda

**`std::function` 開銷:**

```cpp
#include <functional>

// ✗ std::function 有虛擬呼叫開銷
std::function<int(int)> func = [](int x) { return x * 2; };

// ✓ 直接使用 lambda (零開銷)
auto lambda = [](int x) { return x * 2; };

// ✓ 模板參數 (最高性能)
template<typename Func>
void process(Func&& f) {
    int result = f(10);  // 可內聯
}
```
> 1. `std::function` 的目的是能夠儲存**任何**符合特定簽名的可呼叫物件，無論是自由函式、函式指標、`std::bind` 產生的物件，還是帶捕獲或不帶捕獲的 **Lambda**
>  2. `std::function` 必須使用一個內部指標來指向實際儲存的物件（例如 Lambda 產生的匿名類別），並透過一個跳板函式（通常實現類似 **虛擬函式呼叫 (Virtual Function Call)** 的機制）來呼叫真正的邏輯。

**基準測試:**

```cpp
// std::function: ~5-10ns 額外開銷 (虛擬調用 + 間接跳轉)
// 裸 lambda: 完全內聯,零開銷
```

**HFT 建議**: 避免使用 `std::function`,優先使用模板或裸 lambda。

---

## 1.4 模板進階

### 核心概念

模板是 C++ 泛型程式設計的基礎,在高性能場景中用於零開銷抽象。

### 1.4.1 函數模板與類別模板

**函數模板:**

```cpp
template<typename T>
T max(T a, T b) {
    return (a > b) ? a : b;
}

int main() {
    std::cout << max(10, 20);           // int 版本
    std::cout << max(3.14, 2.71);       // double 版本
}
```

**類別模板:**

```cpp
template<typename T, size_t N>
class FixedArray {
private:
    T data_[N];
public:
    constexpr size_t size() const { return N; }
    T& operator[](size_t idx) { return data_[idx]; }
    const T& operator[](size_t idx) const { return data_[idx]; }
};

FixedArray<int, 100> arr;
arr[0] = 42;
```

### 1.4.2 模板特化

**全特化 (Full Specialization):**

```cpp
template<typename T>
class Serializer {
public:
    static void serialize(const T& value) {
        // 通用實作
    }
};

// 針對 int 的全特化
template<>
class Serializer<int> {
public:
    static void serialize(const int& value) {
        // 優化的整數序列化
    }
};
```

**偏特化 (Partial Specialization):**

```cpp
// 通用版本
template<typename T, typename U>
class Pair {
public:
    T first;
    U second;
};

// 針對相同類型的偏特化
template<typename T>
class Pair<T, T> {
public:
    T first;
    T second;
    bool is_same_type() const { return true; }
};

// 針對指標的偏特化
template<typename T>
class Pair<T*, T*> {
public:
    T* first;
    T* second;
    void swap() { std::swap(first, second); }
};
```

### 1.4.3 可變參數模板 (Variadic Templates)

**基本用法:**

```cpp
// 遞迴展開
template<typename T>
void print(const T& value) {
    std::cout << value << "\n";
}

template<typename T, typename... Args>
void print(const T& first, const Args&... rest) {
    std::cout << first << " ";
    print(rest...);  // 遞迴展開參數包
}

int main() {
    print(1, 2.5, "hello", 'c');  // 1 2.5 hello c
}
```

**Fold Expressions (C++17):**

```cpp
template<typename... Args>
auto sum(Args... args) {
    return (args + ...);  // 一元右折疊
}

std::cout << sum(1, 2, 3, 4, 5);  // 15

template<typename... Args>
void print_all(Args... args) {
    (std::cout << ... << args) << "\n";  // 一元左折疊
}

print_all(1, " + ", 2, " = ", 3);  // 1 + 2 = 3
```

### 1.4.4 SFINAE 與 `std::enable_if`

**SFINAE (Substitution Failure Is Not An Error):**

```cpp
#include <type_traits>

// 只對整數類型啟用
template<typename T>
typename std::enable_if<std::is_integral<T>::value, T>::type
process(T value) {
    return value * 2;
}

// 只對浮點類型啟用
template<typename T>
typename std::enable_if<std::is_floating_point<T>::value, T>::type
process(T value) {
    return value * 3.14;
}

int main() {
    std::cout << process(10);      // 20 (整數版本)
    std::cout << process(2.0);     // 6.28 (浮點版本)
    // process("hello");           // 編譯錯誤:無匹配函數
}
```

**C++17 簡化版 `if constexpr`:**

```cpp
template<typename T>
T process(T value) {
    if constexpr (std::is_integral_v<T>) {
        return value * 2;
    } else if constexpr (std::is_floating_point_v<T>) {
        return value * 3.14;
    } else {
        static_assert(always_false<T>, "Unsupported type");
    }
}
```

### 1.4.5 類型萃取 (Type Traits)

```cpp
#include <type_traits>

template<typename T>
void check_type() {
    std::cout << std::boolalpha;
    std::cout << "is_integral: " << std::is_integral_v<T> << "\n";
    std::cout << "is_pointer: " << std::is_pointer_v<T> << "\n";
    std::cout << "is_const: " << std::is_const_v<T> << "\n";
}

// 移除 const 和引用
template<typename T>
void process(T value) {
    using RawType = std::remove_cv_t<std::remove_reference_t<T>>;
    // RawType 是去除修飾的原始類型
}
```

---

## 課後練習

### 練習 1: 支援移動語義的 Vector 容器

**任務**: 實作簡化版 `MyVector<T>`,支援移動語義。

**要求**:
- 實作建構子、解構子
- 實作複製建構子與複製賦值運算子 (深拷貝)
- 實作移動建構子與移動賦值運算子 (轉移所有權)
- 實作 `push_back` (左值與右值版本)
- 測試性能差異


```cpp
#include <iostream>
#include <utility>
#include <algorithm>

template<typename T>
class MyVector {
private:
    T* data_;
    size_t size_;
    size_t capacity_;
    
    void reallocate(size_t new_capacity) {
        T* new_data = new T[new_capacity];
        for (size_t i = 0; i < size_; ++i) {
            new_data[i] = std::move(data_[i]);
        }
        delete[] data_;
        data_ = new_data;
        capacity_ = new_capacity;
    }

public:
    // 建構子
    MyVector() : data_(nullptr), size_(0), capacity_(0) {}
    
    explicit MyVector(size_t capacity) 
        : data_(new T[capacity]), size_(0), capacity_(capacity) {}
    
    // 解構子
    ~MyVector() {
        delete[] data_;
    }
    
    // 複製建構子
    MyVector(const MyVector& other) 
        : data_(new T[other.capacity_]), size_(other.size_), capacity_(other.capacity_) {
        std::cout << "Copy constructor\n";
        std::copy(other.data_, other.data_ + size_, data_);
    }
    
    // 複製賦值運算子
    MyVector& operator=(const MyVector& other) {
        if (this != &other) {
            std::cout << "Copy assignment\n";
            delete[] data_;
            capacity_ = other.capacity_;
            size_ = other.size_;
            data_ = new T[capacity_];
            std::copy(other.data_, other.data_ + size_, data_);
        }
        return *this;
    }
    
    // 移動建構子
    MyVector(MyVector&& other) noexcept
        : data_(other.data_), size_(other.size_), capacity_(other.capacity_) {
        std::cout << "Move constructor\n";
        other.data_ = nullptr;
        other.size_ = 0;
        other.capacity_ = 0;
    }
    
    // 移動賦值運算子
    MyVector& operator=(MyVector&& other) noexcept {
        if (this != &other) {
            std::cout << "Move assignment\n";
            delete[] data_;
            
            data_ = other.data_;
            size_ = other.size_;
            capacity_ = other.capacity_;
            
            other.data_ = nullptr;
            other.size_ = 0;
            other.capacity_ = 0;
        }
        return *this;
    }
    
    // push_back 左值版本
    void push_back(const T& value) {
        if (size_ == capacity_) {
            reallocate(capacity_ == 0 ? 1 : capacity_ * 2);
        }
        data_[size_++] = value;
    }
    
    // push_back 右值版本
    void push_back(T&& value) {
        if (size_ == capacity_) {
            reallocate(capacity_ == 0 ? 1 : capacity_ * 2);
        }
        data_[size_++] = std::move(value);
    }
    
    size_t size() const { return size_; }
    T& operator[](size_t idx) { return data_[idx]; }
    const T& operator[](size_t idx) const { return data_[idx]; }
};

// 測試
int main() {
    MyVector<std::string> v1;
    v1.push_back("Hello");
    
    std::string s = "World";
    v1.push_back(s);              // 左值,複製
    v1.push_back(std::string("!")); // 右值,移動
    
    MyVector<std::string> v2 = std::move(v1);  // 移動建構子
    MyVector<std::string> v3;
    v3 = std::move(v2);           // 移動賦值運算子
    
    return 0;
}
```

---

### 練習 2: Type-safe 的訊息佇列

**任務**: 使用模板實作類型安全的訊息佇列。

**要求**:
- 支援任意類型的訊息
- 實作 `push` (加入訊息) 和 `pop` (取出訊息)
- 執行緒不安全版本即可
- 使用 `std::optional` 處理空佇列情況


```cpp
#include <queue>
#include <optional>
#include <iostream>

template<typename T>
class MessageQueue {
private:
    std::queue<T> queue_;

public:
    void push(const T& message) {
        queue_.push(message);
    }
    
    void push(T&& message) {
        queue_.push(std::move(message));
    }
    
    std::optional<T> pop() {
        if (queue_.empty()) {
            return std::nullopt;
        }
        T value = std::move(queue_.front());
        queue_.pop();
        return value;
    }
    
    bool empty() const {
        return queue_.empty();
    }
    
    size_t size() const {
        return queue_.size();
    }
};

// 測試
struct Order {
    int id;
    double price;
    std::string symbol;
};

int main() {
    MessageQueue<Order> orders;
    
    orders.push(Order{1, 100.5, "AAPL"});
    orders.push(Order{2, 250.3, "GOOGL"});
    
    while (auto order = orders.pop()) {
        std::cout << "Order ID: " << order->id 
                  << ", Price: " << order->price 
                  << ", Symbol: " << order->symbol << "\n";
    }
    
    auto empty_order = orders.pop();
    if (!empty_order) {
        std::cout << "Queue is empty\n";
    }
    
    return 0;
}
```


---

### 練習 3: 使用 SFINAE 的條件編譯函數

**任務**: 實作 `serialize` 函數,根據類型選擇不同的序列化方式。

**要求**:
- 整數類型:輸出 "Serializing integer: [value]"
- 浮點類型:輸出 "Serializing float: [value]"
- 字串類型:輸出 "Serializing string: [value]"
- 其他類型:編譯錯誤

```cpp
#include <iostream>
#include <type_traits>
#include <string>

// 整數版本
template<typename T>
typename std::enable_if<std::is_integral<T>::value>::type
serialize(const T& value) {
    std::cout << "Serializing integer: " << value << "\n";
}

// 浮點版本
template<typename T>
typename std::enable_if<std::is_floating_point<T>::value>::type
serialize(const T& value) {
    std::cout << "Serializing float: " << value << "\n";
}

// 字串版本
void serialize(const std::string& value) {
    std::cout << "Serializing string: " << value << "\n";
}

// C++17 版本 (使用 if constexpr)
template<typename T>
void serialize_cpp17(const T& value) {
    if constexpr (std::is_integral_v<T>) {
        std::cout << "Serializing integer: " << value << "\n";
    } else if constexpr (std::is_floating_point_v<T>) {
        std::cout << "Serializing float: " << value << "\n";
    } else if constexpr (std::is_same_v<T, std::string>) {
        std::cout << "Serializing string: " << value << "\n";
    } else {
        static_assert(!std::is_same_v<T, T>, "Unsupported type");
    }
}

int main() {
    serialize(42);                // integer
    serialize(3.14);              // float
    serialize(std::string("test")); // string
    
    // serialize_cpp17(42);
    // serialize_cpp17(3.14);
    // serialize_cpp17(std::string("test"));
    
    return 0;
}
```

---

### 練習 4: 智能指標資源管理器

**任務**: 實作一個資源管理器,使用 `shared_ptr` 和 `weak_ptr` 管理共享資源。

**要求**:
- 資源管理器可以創建和管理多個資源
- 使用 `weak_ptr` 實現資源緩存,避免循環引用
- 實現 `getOrCreate` 模式
- 資源在沒有使用者時自動釋放

```cpp
#include <memory>
#include <unordered_map>
#include <string>
#include <iostream>

class Resource {
public:
    Resource(const std::string& name) : name_(name) {
        std::cout << "Resource '" << name_ << "' created\n";
    }
    ~Resource() {
        std::cout << "Resource '" << name_ << "' destroyed\n";
    }
    void use() { std::cout << "Using resource '" << name_ << "'\n"; }
private:
    std::string name_;
};

class ResourceManager {
private:
    // 使用 weak_ptr 作為緩存,不會阻止資源釋放
    std::unordered_map<std::string, std::weak_ptr<Resource>> cache_;
    
public:
    std::shared_ptr<Resource> getOrCreate(const std::string& name) {
        // 檢查緩存
        auto it = cache_.find(name);
        if (it != cache_.end()) {
            // 嘗試升級 weak_ptr
            if (auto sp = it->second.lock()) {
                std::cout << "Cache hit for '" << name << "'\n";
                return sp;
            }
        }
        
        // 創建新資源
        std::cout << "Cache miss for '" << name << "'\n";
        auto resource = std::make_shared<Resource>(name);
        cache_[name] = resource;  // 存入緩存
        return resource;
    }
    
    void cleanup() {
        // 移除已過期的緩存條目
        for (auto it = cache_.begin(); it != cache_.end(); ) {
            if (it->second.expired()) {
                it = cache_.erase(it);
            } else {
                ++it;
            }
        }
    }
    
    size_t cacheSize() const { return cache_.size(); }
};

int main() {
    ResourceManager manager;
    
    {
        auto r1 = manager.getOrCreate("database");
        auto r2 = manager.getOrCreate("database");  // 應該返回相同資源
        r1->use();
        
        std::cout << "r1 use_count: " << r1.use_count() << "\n";  // 2
    }
    // r1, r2 超出範圍,資源釋放
    
    auto r3 = manager.getOrCreate("database");  // 應該創建新資源
    
    manager.cleanup();
    std::cout << "Cache size after cleanup: " << manager.cacheSize() << "\n";
    
    return 0;
}
```

---

### 練習 5: 完美轉發工廠函數

**任務**: 實作一個通用的工廠函數,使用完美轉發創建物件。

**要求**:
- 支援任意數量和類型的建構子參數
- 使用完美轉發保持參數的值類別
- 返回 `unique_ptr` 管理的物件

```cpp
#include <memory>
#include <utility>
#include <string>
#include <iostream>

// 測試類別
class Order {
public:
    Order(int id, std::string symbol, double price)
        : id_(id), symbol_(std::move(symbol)), price_(price) {
        std::cout << "Order created: " << id_ << ", " << symbol_ << ", " << price_ << "\n";
    }
    
    // 禁止複製
    Order(const Order&) = delete;
    Order& operator=(const Order&) = delete;
    
    // 允許移動
    Order(Order&&) = default;
    Order& operator=(Order&&) = default;
    
    void print() const {
        std::cout << "Order[" << id_ << "]: " << symbol_ << " @ " << price_ << "\n";
    }

private:
    int id_;
    std::string symbol_;
    double price_;
};

// 通用工廠函數
template<typename T, typename... Args>
std::unique_ptr<T> make_object(Args&&... args) {
    return std::make_unique<T>(std::forward<Args>(args)...);
}

// 帶日誌的工廠函數
template<typename T, typename... Args>
std::unique_ptr<T> make_logged(const std::string& tag, Args&&... args) {
    std::cout << "[" << tag << "] Creating object...\n";
    auto obj = std::make_unique<T>(std::forward<Args>(args)...);
    std::cout << "[" << tag << "] Object created successfully\n";
    return obj;
}

int main() {
    // 測試完美轉發
    auto order1 = make_object<Order>(1, "AAPL", 150.5);
    order1->print();
    
    std::string symbol = "GOOGL";
    auto order2 = make_object<Order>(2, symbol, 2800.0);  // 左值
    auto order3 = make_object<Order>(3, std::move(symbol), 2850.0);  // 右值
    
    // 帶日誌版本
    auto order4 = make_logged<Order>("Factory", 4, "MSFT", 300.0);
    order4->print();
    
    return 0;
}
```

---

### 練習 6: Lambda 事件處理系統

**任務**: 實作一個簡單的事件處理系統,使用 Lambda 作為事件處理器。

**要求**:
- 支援多種事件類型
- 使用模板避免 `std::function` 開銷
- 實現事件訂閱和發布機制

```cpp
#include <vector>
#include <functional>
#include <iostream>
#include <string>

// 事件類型
struct MarketDataEvent {
    std::string symbol;
    double price;
    int volume;
};

struct OrderEvent {
    int orderId;
    std::string action;  // "NEW", "FILLED", "CANCELLED"
};

// 高性能版本:使用模板,避免 std::function
template<typename EventType>
class EventDispatcher {
private:
    // 這裡為了簡化使用 std::function,實際 HFT 應使用函數指標陣列
    std::vector<std::function<void(const EventType&)>> handlers_;
    
public:
    template<typename Handler>
    void subscribe(Handler&& handler) {
        handlers_.emplace_back(std::forward<Handler>(handler));
    }
    
    void publish(const EventType& event) {
        for (auto& handler : handlers_) {
            handler(event);
        }
    }
    
    size_t handlerCount() const { return handlers_.size(); }
};

// 零開銷版本:編譯期多態
template<typename... Handlers>
class StaticDispatcher {
private:
    std::tuple<Handlers...> handlers_;
    
    template<typename Event, std::size_t... Is>
    void dispatchImpl(const Event& event, std::index_sequence<Is...>) {
        (std::get<Is>(handlers_)(event), ...);  // C++17 fold expression
    }
    
public:
    StaticDispatcher(Handlers... handlers) : handlers_(handlers...) {}
    
    template<typename Event>
    void dispatch(const Event& event) {
        dispatchImpl(event, std::index_sequence_for<Handlers...>{});
    }
};

// 輔助函數
template<typename... Handlers>
auto makeDispatcher(Handlers... handlers) {
    return StaticDispatcher<Handlers...>(handlers...);
}

int main() {
    // 動態版本
    EventDispatcher<MarketDataEvent> marketDispatcher;
    
    marketDispatcher.subscribe([](const MarketDataEvent& e) {
        std::cout << "Handler 1: " << e.symbol << " @ " << e.price << "\n";
    });
    
    marketDispatcher.subscribe([](const MarketDataEvent& e) {
        if (e.price > 100) {
            std::cout << "Handler 2: High price alert for " << e.symbol << "!\n";
        }
    });
    
    marketDispatcher.publish({"AAPL", 150.5, 1000});
    
    // 靜態版本 (零開銷)
    auto handler1 = [](const OrderEvent& e) {
        std::cout << "Order " << e.orderId << ": " << e.action << "\n";
    };
    
    auto handler2 = [](const OrderEvent& e) {
        if (e.action == "FILLED") {
            std::cout << "Order " << e.orderId << " filled, updating position\n";
        }
    };
    
    auto staticDispatcher = makeDispatcher(handler1, handler2);
    staticDispatcher.dispatch(OrderEvent{12345, "FILLED"});
    
    return 0;
}
```

---

### 練習 7: 可變參數日誌系統

**任務**: 實作一個類型安全的日誌系統,使用可變參數模板。

**要求**:
- 支援任意數量和類型的參數
- 支援多種日誌級別
- 編譯期格式檢查
- 高性能 (避免不必要的字串分配)

```cpp
#include <iostream>
#include <chrono>
#include <iomanip>
#include <sstream>

enum class LogLevel { DEBUG, INFO, WARNING, ERROR };

class Logger {
private:
    LogLevel minLevel_ = LogLevel::DEBUG;
    
    void printTimestamp() {
        auto now = std::chrono::system_clock::now();
        auto time = std::chrono::system_clock::to_time_t(now);
        std::cout << std::put_time(std::localtime(&time), "%Y-%m-%d %H:%M:%S");
    }
    
    const char* levelToString(LogLevel level) {
        switch (level) {
            case LogLevel::DEBUG:   return "DEBUG";
            case LogLevel::INFO:    return "INFO ";
            case LogLevel::WARNING: return "WARN ";
            case LogLevel::ERROR:   return "ERROR";
        }
        return "UNKNOWN";
    }
    
    // 基礎情況:無參數
    void logArgs() {
        std::cout << "\n";
    }
    
    // 遞迴展開參數
    template<typename T, typename... Args>
    void logArgs(const T& first, const Args&... rest) {
        std::cout << first;
        if constexpr (sizeof...(rest) > 0) {
            std::cout << " ";
        }
        logArgs(rest...);
    }
    
public:
    void setMinLevel(LogLevel level) { minLevel_ = level; }
    
    template<typename... Args>
    void log(LogLevel level, const Args&... args) {
        if (level < minLevel_) return;
        
        std::cout << "[";
        printTimestamp();
        std::cout << "] [" << levelToString(level) << "] ";
        logArgs(args...);
    }
    
    template<typename... Args>
    void debug(const Args&... args) { log(LogLevel::DEBUG, args...); }
    
    template<typename... Args>
    void info(const Args&... args) { log(LogLevel::INFO, args...); }
    
    template<typename... Args>
    void warning(const Args&... args) { log(LogLevel::WARNING, args...); }
    
    template<typename... Args>
    void error(const Args&... args) { log(LogLevel::ERROR, args...); }
};

// 全域 Logger 實例
Logger& getLogger() {
    static Logger instance;
    return instance;
}

// 方便使用的宏
#define LOG_DEBUG(...) getLogger().debug(__VA_ARGS__)
#define LOG_INFO(...) getLogger().info(__VA_ARGS__)
#define LOG_WARNING(...) getLogger().warning(__VA_ARGS__)
#define LOG_ERROR(...) getLogger().error(__VA_ARGS__)

int main() {
    Logger& logger = getLogger();
    
    logger.info("Application started");
    logger.debug("Processing order", 12345, "for symbol", "AAPL");
    logger.warning("High latency detected:", 150, "ms");
    logger.error("Connection lost to exchange", "NYSE");
    
    // 使用宏
    LOG_INFO("Using macro interface");
    
    // 調整日誌級別
    logger.setMinLevel(LogLevel::WARNING);
    LOG_DEBUG("This won't be printed");
    LOG_WARNING("This will be printed");
    
    return 0;
}
```

---

### 練習 8: 編譯期計算的查找表

**任務**: 使用模板元編程在編譯期生成查找表。

**要求**:
- 編譯期計算費波那契數列
- 編譯期計算階乘
- 使用 `constexpr` 優化

```cpp
#include <iostream>
#include <array>

// 編譯期階乘
constexpr unsigned long long factorial(int n) {
    return (n <= 1) ? 1 : n * factorial(n - 1);
}

// 編譯期費波那契
constexpr unsigned long long fibonacci(int n) {
    if (n <= 1) return n;
    return fibonacci(n - 1) + fibonacci(n - 2);
}

// 生成編譯期查找表
template<std::size_t N>
constexpr std::array<unsigned long long, N> generateFactorialTable() {
    std::array<unsigned long long, N> table{};
    for (std::size_t i = 0; i < N; ++i) {
        table[i] = factorial(i);
    }
    return table;
}

template<std::size_t N>
constexpr std::array<unsigned long long, N> generateFibonacciTable() {
    std::array<unsigned long long, N> table{};
    for (std::size_t i = 0; i < N; ++i) {
        table[i] = fibonacci(i);
    }
    return table;
}

// 查找表 (編譯期生成)
constexpr auto factorialTable = generateFactorialTable<21>();  // 20! 是 unsigned long long 的極限
constexpr auto fibonacciTable = generateFibonacciTable<50>();

// 模板元編程版本 (C++11 相容)
template<int N>
struct Factorial {
    static constexpr unsigned long long value = N * Factorial<N - 1>::value;
};

template<>
struct Factorial<0> {
    static constexpr unsigned long long value = 1;
};

template<int N>
struct Fibonacci {
    static constexpr unsigned long long value = Fibonacci<N - 1>::value + Fibonacci<N - 2>::value;
};

template<>
struct Fibonacci<0> {
    static constexpr unsigned long long value = 0;
};

template<>
struct Fibonacci<1> {
    static constexpr unsigned long long value = 1;
};

int main() {
    // 使用查找表 (O(1) 查詢,編譯期計算)
    std::cout << "5! = " << factorialTable[5] << "\n";
    std::cout << "10! = " << factorialTable[10] << "\n";
    std::cout << "20! = " << factorialTable[20] << "\n";
    
    std::cout << "Fib(10) = " << fibonacciTable[10] << "\n";
    std::cout << "Fib(40) = " << fibonacciTable[40] << "\n";
    
    // 模板元編程版本
    std::cout << "Factorial<10>::value = " << Factorial<10>::value << "\n";
    std::cout << "Fibonacci<20>::value = " << Fibonacci<20>::value << "\n";
    
    // 驗證編譯期計算
    static_assert(factorialTable[5] == 120, "Compile-time check failed");
    static_assert(Fibonacci<10>::value == 55, "Compile-time check failed");
    
    return 0;
}
```

---

### 練習 9: 自定義智能指標

**任務**: 實作一個簡化版的 `unique_ptr`。

**要求**:
- 禁止複製
- 支援移動語義
- 支援自定義刪除器
- 實現 `reset`, `release`, `get` 方法

```cpp
#include <iostream>
#include <utility>

template<typename T, typename Deleter = std::default_delete<T>>
class MyUniquePtr {
private:
    T* ptr_ = nullptr;
    Deleter deleter_;
    
public:
    // 建構子
    explicit MyUniquePtr(T* ptr = nullptr) : ptr_(ptr) {}
    
    MyUniquePtr(T* ptr, Deleter deleter) : ptr_(ptr), deleter_(deleter) {}
    
    // 解構子
    ~MyUniquePtr() {
        if (ptr_) {
            deleter_(ptr_);
        }
    }
    
    // 禁止複製
    MyUniquePtr(const MyUniquePtr&) = delete;
    MyUniquePtr& operator=(const MyUniquePtr&) = delete;
    
    // 移動建構子
    MyUniquePtr(MyUniquePtr&& other) noexcept
        : ptr_(std::exchange(other.ptr_, nullptr)),
          deleter_(std::move(other.deleter_)) {}
    
    // 移動賦值
    MyUniquePtr& operator=(MyUniquePtr&& other) noexcept {
        if (this != &other) {
            reset();
            ptr_ = std::exchange(other.ptr_, nullptr);
            deleter_ = std::move(other.deleter_);
        }
        return *this;
    }
    
    // 解引用
    T& operator*() const { return *ptr_; }
    T* operator->() const { return ptr_; }
    
    // 獲取裸指標
    T* get() const { return ptr_; }
    
    // 釋放所有權
    T* release() {
        return std::exchange(ptr_, nullptr);
    }
    
    // 重置
    void reset(T* ptr = nullptr) {
        T* old = std::exchange(ptr_, ptr);
        if (old) {
            deleter_(old);
        }
    }
    
    // 布林轉換
    explicit operator bool() const { return ptr_ != nullptr; }
    
    // 交換
    void swap(MyUniquePtr& other) noexcept {
        std::swap(ptr_, other.ptr_);
        std::swap(deleter_, other.deleter_);
    }
};

// make_unique 實作
template<typename T, typename... Args>
MyUniquePtr<T> my_make_unique(Args&&... args) {
    return MyUniquePtr<T>(new T(std::forward<Args>(args)...));
}

// 測試類別
class TestClass {
public:
    TestClass(int value) : value_(value) {
        std::cout << "TestClass(" << value_ << ") created\n";
    }
    ~TestClass() {
        std::cout << "TestClass(" << value_ << ") destroyed\n";
    }
    int getValue() const { return value_; }
private:
    int value_;
};

int main() {
    // 基本使用
    auto ptr1 = my_make_unique<TestClass>(42);
    std::cout << "Value: " << ptr1->getValue() << "\n";
    
    // 移動
    auto ptr2 = std::move(ptr1);
    if (!ptr1) {
        std::cout << "ptr1 is now null\n";
    }
    
    // reset
    ptr2.reset(new TestClass(100));
    
    // release
    TestClass* raw = ptr2.release();
    delete raw;
    
    // 自定義刪除器
    auto customDeleter = [](TestClass* p) {
        std::cout << "Custom deleter called\n";
        delete p;
    };
    MyUniquePtr<TestClass, decltype(customDeleter)> ptr3(
        new TestClass(200), customDeleter);
    
    return 0;
}
```


---

## 總結

### 核心要點

1. **智能指標**
   - `unique_ptr`: 零開銷,HFT 首選
   - `shared_ptr`: 有原子操作開銷,謹慎使用
   - 最佳實踐:物件池 + 裸指標

2. **移動語義**
   - 複製 vs 移動:O(n) vs O(1)
   - 務必實作移動建構子/賦值運算子
   - 避免阻礙 RVO

3. **Lambda**
   - 避免 `std::function`,使用模板或裸 lambda
   - 捕獲注意生命週期
   - HFT 場景:優先值捕獲或無捕獲

4. **模板**
   - 零運行時開銷的泛型
   - SFINAE/`if constexpr` 實現條件編譯
   - 編譯期計算與優化

### 下一步

Day 2 將深入 C++17/20 特性、記憶體管理與 STL 容器選擇。

---

## 參考資料 (References)

1. [C++ Reference - Smart Pointers](https://en.cppreference.com/w/cpp/memory)
2. [C++ Reference - Move Semantics](https://en.cppreference.com/w/cpp/language/move_constructor)
3. Scott Meyers, "Effective Modern C++" (2014)
4. Nicolai M. Josuttis, "C++17 - The Complete Guide" (2019)
5. Bjarne Stroustrup, "The C++ Programming Language, 4th Edition" (2013)
