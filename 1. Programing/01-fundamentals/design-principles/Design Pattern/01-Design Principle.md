
# 設計原則
---
##### 1. Single Responsibility Principle (SRP)
* 一個類只負責一個功能領域中的職責
##### 2. Dependency Inversion Principle (DIP)
* 高層模塊不依賴於低層模塊，兩者都應該依賴於抽象
	* 也就是說高層模快使用抽象，低層模塊實現抽象
* 就是面向接口(抽象類)編程
##### 3. Open Close Principle (OCP)
* 一個軟件應該對擴展開放，對修改關閉。
* 假設添加一個新功能要對原有程式進行修改，就破壞 OCP
##### 4. Interface Segregation Principle (ISP)
* 應該使用多個專門的接口，不使用單一的總接口
##### 5. Liskov Subsititution Principle (LSP)
* 所以基類出現的地方必定能被子類替換，而功能不會受到影響。
	* 不要改寫父類的方法
* 若出現繼承之後不會用到父類所實現的方法時或要改寫大部分方法時，就要考慮不該使用繼承。
> 子類想要改寫父類行為，其本身就不應該為其子類，所有派生類都應可以替換父類。
##### 6. Composite Reuse Principle (CRP)
* 盡量使用對象的組合/聚合，而不是透過繼承來達到複用的目的。
> 聚合(Aggregation) 是 has-A 關係，整體與個體間可以獨立存在，沒有生命週期關係，**透過傳參**數建立。
> 	-> 公司與會計的關係。
> 組合(composition) 是 contains-A 關係，整體與個體不可獨立存在，有生命週期關係，**不透過傳參**建立。
> 	-> 身體與腳的關係。
> 繼承(inheritance) 是 is-A 關係。

```c#
// Aggregation 特徵: 可以更換
public class Graphics
{
	private readonly Shape _shape;
	public Graphics(Shape shape) { _shape = shape; }
	public SetShape(Shape shape) { _shape = shape; }
}

// Composition 特徵: 不能更換
public class Graphics
{
	private readonly Shape _rectangle = new Rectangle(); // 生命週期由類管理
}
```
##### 7. Demeter Law
* 軟件的實體應當盡量少的與其他實體發生相互作用，類與類之間耦合度盡量降低，直接相互調用相互依賴。

> OCP 是目標，DIP 是手段。


# Reference
---
* [c# 設計模式 by 煮詩君](https://www.bilibili.com/video/BV1zZ4y1u7mp/?p=13&spm_id_from=pageDriver&vd_source=81381cbff2baf8dfd1b22050d029f496)