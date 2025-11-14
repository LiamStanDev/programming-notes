結構型設計模式: 用來組織不同的類與對象來組成更大的結構完成新的功能。

# 適配器模式 (Adapter)
---
將一個類的接口轉換成客戶希望的另一個接口，將原本不能兼容的類可以兼容。
### 案例
1. 電腦只有 USB 接口，不能插入 SD 卡。
2. 電腦與 SD 卡都已經成形，無法改造
3. 我們需要一個轉換器，也就是讀卡器。
![[Pasted image 20240229131123.png|600]]
### 實作
* IUsb.cs
```cs
public interface IUsb
{
    void Request();
}
```
* Computer.cs
```cs
public class Computer
{
    private IUsb _usb;

    public void SetUsb(IUsb usb)
    {
        _usb = usb;
    }

    public void ConnectUsb()
    {
        if (_usb is not null)
        {
            _usb.Request();
        }
    }
}
```

* SdCard.cs
```cs
public class SdCard
{
    public void ReadAndWrite()
    {
        Console.WriteLine("Getting Data...");
    }
}
```
* SdReader.cs
```cs
public class SdReader : IUsb
{
    private readonly SdCard _sdCard; // 這邊使用合成復用原則，不採取直接繼承。

    public SdReader(SdCard sdCard)
    {
        _sdCard = sdCard;
    }

    public void Request()
    {
        _sdCard.ReadAndWrite();
    }
}
```
* Program.cs
```cs
var computer = new Computer();
SdCard sdCard = new SdCard();
computer.SetUsb(new SdReader(sdCard));

computer.ConnectUsb();
```

### 優缺點
* 優點:
	* 可以讓兩個沒有關聯的類一起運行。
	* 提高類的複用，不用再實作新的 `SdCard`
	* 無須修改原有的結構，遵守 OCP
* 缺點:
	* 過多依賴 Adapter 會讓系統凌亂。
> 適配器模式屬於補償用，專門用來系統後期擴展使用。應優先考慮重構來解決問題。


# 代理模式 (Proxy)
---
### 概念
* **Delegate**: 委派是 C# 中引用方法的類型，相當於 C++ 的函數指針。
* **Proxy**: 代理在生活上的概念就是代理商、代購等，**見他如見我**，不會知道真正的我是誰。
> 代理直接代替被代理者。
![[Pasted image 20240301213313.png|800]]
* **Mediator**: 中介就是中間的橋樑，幫你找到我，**雙方終要見面**。
> 中介只是幫你找
![[Pasted image 20240301213624.png|800]]
### 使用場景
* Windows 快捷鍵
* VPN
* PRC 調用

### 目的
1. 在不改變原有代碼的基礎上，對原有的類進行控制。
2. 由於某種原因不能直接訪問，或直接訪問非常困難。

### UML 類圖
![[Pasted image 20240302122421.png|800]]

### 實作
案例: 我們已經寫好了 `UserRepository` ，我們希望不修改它但擴展它的功能，使其可以添加打印與捕獲異常。
```cs
// 與 UserRepository 一樣實現 IUserRepository
public class UserRepositoryProxy : IUserRepository
{
    private readonly IUserRepository _userRepository = new UserRepository(); // 採用合成復用原則，並且調用者並不需要知道實際訪問是誰(Proxy核心概念)
    private readonly ILogger<UserRepositoryProxy> _logger; // 添加日誌 

    public UserRepositoryProxy(ILogger<UserRepositoryProxy> logger)
    {
        _logger = logger;
    }

    public void Add(User user)
    {
		    // 添加新功能
        string message = $"UserRepositoryProxy-Add In:Username={user.Id}";
        _logger.LogDebug(message);
        // 使用原有功能
        _userRepository.Add(user);
    }

    public void Delete(int id)
    {
        string message = $"UserRepositoryProxy-Delete In:Id={id}";
        _logger.LogDebug(message);
        _userRepository.Delete(id);
    }

    public User? Get(int id)
    {
        string message = $"UserRepositoryProxy-Get In:Id={id}";
        _logger.LogDebug(message);
        return _userRepository.Get(id);
    }

    public void Update(int id, string name)
    {
        string message = $"UserRepositoryProxy-Update In:Username={name}";
        _logger.LogDebug(message);
        _userRepository.Update(id, name);
    }
}
```
> 記住見它如見我。

#### Proxy vs. Adapter
1. 兩者在程式碼上表現幾乎一致
2. Adapter 解決的是**不兼容問題**，原本的兩類不能一起工作。
3. Proxy 解決的是實現**間接訪問與控制**，原本的兩類就能一起工作。

# 外觀模式 (Facade Pattern)
---
### 定義
為子系統中的一組接口提供一個一致的介面，**外觀模式定義了一個高層的接口，使子系統更容易使用**。
![[Pasted image 20240302124120.png]]
> 外觀模式與代理模式類似。
### UML 類圖
![[Pasted image 20240302125543.png]]

### 實作
現在有電燈、冷氣、窗戶與電視(四個子系統)，用戶有三個動作回家、睡覺、出門。
* 尚未修改前 Program.cs
```cs
Console.WriteLine("-----Back home--------");
window.Open();
tv.TurnOn();
light.TurnOn();
ac.TurnOn();

Console.WriteLine("-------Sleep----------");
window.Close();
tv.TurnOff();
light.TurnOff();
ac.TurnOn();


Console.WriteLine("-------Leave----------");
window.Close();
tv.TurnOff();
light.TurnOff();
ac.TurnOff();
```
> 問題: 所有操作都讓用戶具體實現，可能容易出錯而且封裝性不好


* 使用 Facade Pattern
```cs
public class OneClickFacade
{
		private static readonly Window _window = new Window();
		private static readonly Tv _tv = new Tv();
		private static readonly Light _light = new Light();
		private static readonly Ac _ac = new Ac();

		public void BackHome()
		{
				Console.WriteLine("-----Back home--------");
				window.Open();
				tv.TurnOn();
				light.TurnOn();
				ac.TurnOn();
		}
		
		public void Sleep()
		{
				Console.WriteLine("-------Sleep----------");
				window.Close();
				tv.TurnOff();
				light.TurnOff();
				ac.TurnOn();
		}
		
		public void Leave()
		{
				Console.WriteLine("-------Leave----------");
				window.Close();
				tv.TurnOff();
				light.TurnOff();
				ac.TurnOff();
		}
}
```

### Facade Pattern vs. Proxy Pattern
#### 相同
* 都可以在不改變子系統基礎上，對子系統加以控制。
* 原有的系統都可以直接進行訪問，兩者都是為了讓子系統更容易使用

#### 不同
* 外觀模式面向的是多個不同的子系統，代理模式面向是一個子系統。
* **外觀模式強調對多個子系統的整合**，代理模式強調是對某個子系統的代理。


# 裝飾器模式 (Decorator)
---
### 定義
裝飾器模式能動態的給一個對象增加一些額外的職責，相較於使用子類繼承的方式更加靈活。
### 概念推導
案例: 一個飲料店，裡面有很多的飲品，有奶茶與咖啡，飲品可以增加配料。
#### Stage 1: 單純使用繼承
```cs 
// 抽象基類
public abstract class Drink
{
		public string Name {get; set;}
		public string Pirce {get; set; }
		public abstract string Desc { get; }
		public abstract int Cost { get; }
}

// 奶茶
public class MilkTee : Drink
{
		public MilkTee()
		{
				Name = "奶茶";
				Price = 50;
		}
		public override string Desc => this.Name;
		public override int Cost =>　this.Price;
}
// 咖啡
public class MilkTee : Drink
{
		public MilkTee()
		{
				Name = "咖啡";
				Price = 60;
		}
		public override string Desc => this.Name;
		public override int Cost =>　this.Price;
}
// 珍珠奶茶
public class BubbuleTee : Drink
{
		public BubbuleTee()
		{
				Name += "+珍珠";
				Price += 15;
		}
}
```
* 問題: 
	1. 隨者飲品變多，類的數量會急遽上升。
	2. 若想要添加新的功能冰塊，需要所有的類都進行修改，違反開閉原則。

#### Stage 2: 使用抽象工廠概念來收縮類的數量
* 將所有想的到的擴充，都在抽象類裡面添加功能。
```cs
public abstract class Drink
{
		public string Name {get; set;}
		public string Pirce {get; set; }
		public abstract string Desc { get; }
		public abstract int Cost { get; }
		
		public abstract void AddBuding();
		public abstract void AddRedBean();
		public abstract void AddIce();
		public abstract void AddSuger();
}
```

* 問題
	1. 導致所有的子類都需要實現方法，比如說咖啡就不用有家紅豆的方法。

#### Stage 3: 將配件實例化後聚合到基類中
```cs
// 配料基類
public abstract class Dressing 
{
		public string Name { get; set; }
}

// 飲料基類
public abstract class Drink
{
		public string Name {get; set;}
		public string Pirce {get; set; }
		public abstract string Desc { get; }
		public abstract int Cost { get; }
		
		public List<Dressing> Dressings = new List<Dressing>();
		public void AddDressing(Dressing dressing)
		{
				Dressings.Add(dressing);
		}
}

// Program.cs
Drink milkTee = new MilkTee();
milkTee.AddDressing(new RedBeen());
milkTee.AddDressing(new Ice());
```
* 問題:
	* 修改了基類，基類可能無法進行修改。
##### UML 類圖
![[Pasted image 20240305104547.png|600]]
#### Stage 4: 裝飾器模式
我們將配件聚合到基類上，若反過來思考將基類聚合到配件上，並繼承基類，這就是裝飾器模式。

```cs
// 不修改抽象基類
public abstract class Drink
{
		public string Name {get; set;}
		public string Pirce {get; set; }
		public abstract string Desc { get; }
		public abstract int Cost { get; }
}

// 配件基類
public abstract class Dressing : Drink
{
		protected readonly Drink Drink; // 聚合
		public Dressing(Drink drink) // 使用構造器聚合
		{
				Drink = drink;
		}

		public override string Desc => Drink.Desc + "+" + this.Name;
}

// Program.cs
Drink drink = new MilkTee();
// 因使用繼承所以類型還是 Drink
drink = new RedBeen(drink);
drink = new Ice(drink);
```

> 裝飾器模式就是透過聚合+繼承，來擴展原有功能，並遵循了開閉原則，**聚合是為了實現功能，繼承是為了類型約束**。

### 裝飾器模式 vs. 代理模式
* 相同點:
	1. 都是使用聚合或組合+繼承的方式來實現
	2. 都可以實現 AOP。
* 相異點:
	1. 裝飾器模式關注點在動態的**添加功能**，而代理模式關注點在**控制對象訪問**。
	2. 裝飾器模式採用聚合，代理模式組合(因為用戶不需要知道實際是誰)。
	3. 裝飾器模式會多層裝配，而代理模式通常只會有一層(甚至用戶根本沒感覺)。

### 經典案例
在 .NET 類庫中，`System.IO.Stream` 中就是使用裝飾器模式，但並沒有使用到 Decorator 基類。
![[Pasted image 20240305110017.png]]
* `MemoryStream`, `FileStream`, `NetworkStream` 都是繼承 Stream 抽象基類的實例類型
* 配件有 `BufferedStream`, `CrytoStream`, `GZipStream`。
* 使用方式
```cs
Stream fs = new FileStream(@"./test", FileMode.OpenOrCreate);
fs = new BufferedStream(fs);
fs = new GZipStream(fs, CompressionLevel.Optimal);
```

