## Delegate
---
### 介紹
委託本質是類，繼承 `System.MulticastDelegate` ，用於將方法包裝成一個變量，進行**行為傳遞**，可以將行為交給調用者自行配置，以達成解耦。

### 使用
#### 基本方式
```c#
public delegate void NoReturnDelegate(ref int x, out int y);

public class Solution {
	public NoReturnDelegate NoReturnDelegateHandler;
	public Solution(NoReturnDelegate handler) {
		NoReturnDelegateHander = handler;
	}
	public void DoNothiing() {
		if (NoReturnDelegateHandler != null) {
			// 調用方式一：
			NoReturnDelegateHandler.Invoke();
			// 調用方式二：
			NoReturnDelegateHandler();
		}
	}
}

public class Program {
	static void Main() {
		// 建立方式一：
		NoReturnDelegate method = new NoReturnDelegate(Solutoin.DoNothing);
		// 建立方式二：
		NoReturnDelegate method2 = (ref int x, out int y) => {};
		
		new Solution(method).DoNothing();
	}
}
```

#### 多播委託
```c#
int x = 10;
WithReturnNoParaDelegate method = () => x = 10;

method += () => x = 11;
method += () => x = 11;
method += () => x = 12;
method -= () => x = 12; // 從尾部批配移出符合的一個。

method.Invoke(); // 依序調用
```

## Event
事件與委託不同，**事件是一個實例**，而**委託是一個類型**，事件是用來修飾多播委託的實例，使得多播委託的實例有以下限制：
1. 無法在類外被調用：也就是需要在類裡面的方法進行 Invoke()
2. 不能使用 `=` 進行賦值：使得事件只能添加與移除不能覆蓋 (安全性)

> 委託使你可以註冊行為

* 未始用事件解耦
```c#
public class Cat
{
	public static void Miao()
	{
		Console.WriteLine("Miao~~~");
	    new Mouse().Run();
	    new Baby().Cry();
	    new Mother().Wispher();
	    new Father().Roar();
	    new Neighbor().Awake();
	    new Dog().Wang();
	}
}
```

* 使用事件解耦
```c#
public class Cat { 
	public event Delegate MioDelegateHandlerEvent;
	public static void Miao() { 
		Console.WriteLine("Miao~~~"); 
		if (MiaoDelegateHandler != null) { 
			MiaoDelegateHander.Invoke(); 
		} 
	} 
} 

// Program.cs 
Cat cat new Cat(); 
cat.MioDelegateHandlerEvent += new MiaoDelegateHander(new Mouse().Run());
cat.MioDelegateHandlerEvent += new MiaoDelegateHander(new Baby().Cry());
cat.MioDelegateHandlerEvent += new MiaoDelegateHander(new Mother().Wispher());
.... 
cat.Miao();
```