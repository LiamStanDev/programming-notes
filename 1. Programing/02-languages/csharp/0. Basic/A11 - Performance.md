# Lazy Initialize (延遲初始化)
在某些場景我們希望物件被第一次調用時，才對他進行初始化。
```cs
public class Customer
{
		// Oder 會消耗大量資源
		public Lazy<Order> Order = new Lazy<Order>();
}
```
表示在建立 Customer 時並不會直接初始化 Oder 字段。而是真的調用 Order 時才進行初始化。
.NetFramework 4 提供了四種方式:
1. `Lazy<T>`: 線程安全且能進行懶加載。
2. `ThreadLocal<T>`: 多線程都可以有自己的實例的懶加載。
3. `LazyInitializer<T>`: skip...
### Lazy
```cs
// 會調用無參構造器
Lazy<Orders> _orders = new Lazy<Orders>();
// 指定調用構造器
Lazy<Order> _orders = new Lazy<Orders>(() => new Orders(100));

// 使用
Order oVal = _orders.Value; // 調用 Value 才會建立
```


