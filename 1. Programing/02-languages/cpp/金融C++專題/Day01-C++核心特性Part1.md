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
