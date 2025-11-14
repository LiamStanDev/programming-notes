### C#泛型原理
**在compile-time時使用佔位符號填充**，編譯完成之後還是依然為佔位符，在run-time時在JIT技術下，再將佔位符換成實際類型。
### Generic Constraint
```c#
public class Constraint
{
    public class Peoeple { }
    public interface IPeople { }
    // 基類泛型約束
    public static void Show<T>(T tParameter) where T : Peoeple
    { }

    // 接口泛型約束
    public static void Get<T>(T tParameter) where T : IPeople
    { }

    // 引用類型約束
    public static void Obtain<T>(T tParameter) where T : class
    { tParameter = null; }

    // 值類型約束
    public static void Fetch<T>(T tParameter) where T : struct
    { tParameter = default; }

    // T 必須要有無參構造函數
    public static void Grab<T>(T tParameter) where T : new()
    { }
}
```
> 為甚麼不傳入基類而使用泛型約束?
> 因為我希望有多個條件，如下:
```c#
public static void Get<T> (T tParameter) where T : People, ISport, ISwin
{
	tParameter.Swim();
}
```

### Covariance & Contravariance
source : https://www.huanlintalk.com/2009/10/c-40covariance-and-contravariance.html
#### 問題
```c#
List<string> strList = new List<string>();
List<object> objList = strList; // Error! 
```
> Cannot implicitly convert type System.Collections.Generic.List`<`string`>` to System.Collections.Generic.List`<`object`>`

為甚麼明明string是object，但是卻無法進行此轉換?
因為要把類型與泛型用整體來看 List`<`string`>` 並沒有繼承 List`<`object`>` 所以轉換失敗，這是原於泛型本身具有不變性(invariance)
#### Covariance
共變性是 .Net 4.0 的特性，用 **out 關鍵字** 來限制泛型參數只能用於返回值。
```c# 
public interface IEnumerable<out T> : IEnumerable {
	IEnumerator<T> GetEnumerator(); // 泛型T只能用於返回值
}

public interface IFoo<out T>
{
    string Convert(T obj);  // 編譯失敗：型別 T 在這裡只能用於方法的傳回值，不可當作參數。
    T GetInstance();        // OK!
}
```
##### 使用案例
使用表資料結構放入多種不同的類型(同一基類)。假設Fruit被Apple與Orange繼承。
1. 使用數組(最簡單)
```c#
Fruit[] fruits = new Apple[10];
fruits[0] = new Apple();
fruits[1] = new Orange();
```
2. 使用IEnumerable`<`T`>` (因為其使用out修飾)
```c#
// 錯誤
// List<Fruit> fruits = new List<Fruit>();
// fruits.Add(new Apple()); // Error!!!!

// 正確
IEnumerable<Fruits> fruits = new List<Apple>();
fruits.Add(new Apple());
fruits.Add(new Orange());
```


#### Contravariance
這邊不用理解太多，只要知道在泛型參數前面使用 in 修飾，表示該泛型參數只能用於參數。