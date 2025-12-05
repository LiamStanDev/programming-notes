
## Multicast Reciever 實現

### 什麼是 UDP Multicast？

Multicast（組播） 是一種網絡通信模式，允許一個發送者同時向多個接收者發送數據，而不需要為每個接收者建立單獨的連接。
三種網絡通信模式對比：
1. **Unicast (單播)     1對1**
   發送者 → 接收者A
   發送者 → 接收者B    (需要發送兩次)
2. **Broadcast (廣播)   1對所有**
   發送者 → 網段內所有設備 (包括不需要的設備)
3. **Multicast (組播)   1對多**
   發送者 → [組播組] → {訂閱該組的接收者們}
   ✅ 只發送一次
   ✅ 只有訂閱者接收
**為什麼 HFT 系統使用 Multicast？**
4. 效率高: 交易所只需發送一次市場數據，所有交易員同時接收
5. 延遲低: 無需建立 TCP 連接的握手開銷
6. 公平性: 所有訂閱者在同一時間收到數據（理論上）
7. 擴展性: 支持數千個接收者而不增加發送者負擔
**Multicast 地址範圍**
- IPv4: 224.0.0.0 ~ 239.255.255.255
- 我們使用: 239.255

### 架構總覽（模塊化分層）
```
┌─────────────────────────────────────────────────┐
│          MarketDataReceiver (協調層)             │
│  - 線程管理                                       │
│  - 生命週期控制 (start/stop)                      │
│  - 統計信息                                       │
└────────────┬──────────────────┬──────────────────┘
             │                  │
             ▼                  ▼
   ┌─────────────────┐  ┌─────────────────┐
   │   UDPSocket     │  │  ITCHParser     │
   │  (網絡層)       │  │  (解析層)       │
   │                 │  │                 │
   │ - socket()      │  │ - validate()    │
   │ - bind()        │  │ - parse()       │
   │ - setsockopt()  │  │ - 類型檢測      │
   │ - recv()        │  │                 │
   │ - multicast加入 │  │                 │
   └─────────────────┘  └─────────────────┘
             │                  │
             └──────────┬───────┘
                        ▼
              ┌──────────────────┐
              │   SPSC Queue     │
              │  (已實現)        │
              └──────────────────┘
```

```cpp
/**
 * MarketDataReceiver 配置參數
 */
struct ReceiverConfig {
  // 網絡配置
  std::string multicast_group = "239.1.1.1";  // Multicast 組地址
  uint16_t port = 12345;                      // UDP 端口
  std::string interface = "0.0.0.0";          // 綁定的網絡接口 (任意接口)
  
  // 緩衝配置
  size_t socket_buffer_size = 256 * 1024;     // SO_RCVBUF (256KB)
  size_t receive_buffer_size = 65536;         // recv() 緩衝區 (64KB)
  
  // 性能配置
  bool enable_timestamps = true;              // 記錄接收時間戳
  bool enable_statistics = true;              // 啟用統計
  
  // 驗證配置是否有效
  bool validate() const {
    return port > 0 && !multicast_group.empty();
  }
};
```

|**設置**|**目的**|**最佳實踐**|
|---|---|---|
|**`SO_RCVBUF` (256KB)**|**容錯與容量**：提供一個足夠大的"水庫"，用於緩衝數據。當應用程式忙碌（例如正在處理交易邏輯）而無法立即呼叫 `recv()` 時，這個緩衝區可以防止核心丟棄新到達的數據包。|設置得**足夠大**，以容納網路延遲與數據速率乘積的數據量。|
|**`recv()` 緩衝區 (64KB)**|**效率與速度**：提供一個高效的大小，用於執行單次 `recv()` 系統呼叫。如果緩衝區太小（如 1KB），您需要進行多次系統呼叫才能讀完核心中的數據。|設置在 **4KB ~ 64KB** 之間通常是最佳的平衡點，太大的話會浪費應用層記憶體。|

### OrderBook 為甚麼要使用 boost::intrusive

#### **場景：處理新的委託**
```
時間點 T0: 市場數據到達（從網卡到內存）
時間點 T1: 數據進入 SPSC Queue（15.6ns - 您已實現）
時間點 T2: 訂單簿處理（目標 < 1μs）
時間點 T3: 信號決策（< 100ns）
時間點 T4: 訂單發出（< 1μs）
```

**問題：在 T2 階段，傳統 C++ 容器會做什麼？**

---

#### 📊 傳統方案 vs. Intrusive 的成本對比

Intrusive 做了甚麼
```cpp
struct Order {
    // 這是 Order 的核心數據 (Core Data)
    OrderID order_id;
    Price price;
    Quantity quantity;
    
    // 這是 bi::list_member_hook<> 在 Order 內部的等效結構 (Hook Data)
    // 用於價格級別 (Price Level) 上的雙向鏈表
    Order* price_level_next; 
    Order* price_level_prev;
    
    // 這是 bi::set_member_hook<> 在 Order 內部的等效結構 (Hook Data)
    // 用於 OrderID 索引的平衡樹
    Order* id_index_parent;
    Order* id_index_left;
    Order* id_index_right;
    Color  id_index_color; // (或其他 Set 管理數據)
};
```

看到以上結構可以知道，`Order` 對象本身作為容器的節點，而不是依賴於 `std::list` 等傳統容器為每個元素在堆上建立一個額外的、獨立的 `ListNode` 結構來指向它。

ntrusive 容器通過這種方式，實現了對傳統 C++ 容器的三個關鍵優化：

1. **消除間接性 (Eliminating Indirection)**
    - **傳統：** `std::list` 遍歷 `ListNode*` → `ListNode` → `Order*` → `Order` 數據。
    - **Intrusive：** 容器直接遍歷 `Order*` 內的 Hook 指針 → 下一個 `Order` 數據。
    - **效益：** 避免了多一層指針跳轉，極大提高了 **CPU 緩存命中率** 和 **數據本地性**。
        
2. **零運行時分配 (Zero Runtime Allocation)**
    - Hook 指針是 `Order` 結構的成員，它們的內存與 `Order` 的核心數據一起，在程序啟動時通過內存池（或其他機制）**預分配**完成。
    - **效益：** 徹底避免了運行時昂貴的 `malloc/new` 調用，將延遲從幾百納秒降低到幾十納秒。
        
3. **單一所有權 (Single Ownership)**
    - 一個 `Order` 對象可以通過其內嵌的不同 Hook，同時作為多個邏輯結構（如價格列表和 ID 索引樹）的節點，而**無需任何額外的內存開銷**。

### Order 生命週期

```
時刻 T0: 程序啟動
  └─ 創建 Order 池（預分配 1,000,000 個）
  └─ 所有 Order 對象在連續內存中
  
時刻 T1: 收到 ADD 訂單事件
  ┌─ 從池中取出一個空的 Order 對象 → order_pool[idx]
  ├─ 初始化：Order.order_id = 001, price = 100.00, qty = 100
  ├─ 插入 bid_levels[100.00].orders (list push_back)  ← list_member_hook 起作用
  ├─ 插入 order_id_index (set insert)                  ← set_member_hook 起作用
  └─ 完成！無堆分配
  
時刻 T2: 收到 REMOVE 訂單事件
  ┌─ 查詢：order_id_index.find(001) → 獲得 Order 引用
  ├─ 從 bid_levels[100.00].orders 移除
  ├─ 從 order_id_index 移除
  ├─ Order 對象歸還池（標記為空）
  └─ 完成！無堆分配
時刻 T3: 程序結束
  └─ Order 池自動銷毀，內存歸還 OS
```