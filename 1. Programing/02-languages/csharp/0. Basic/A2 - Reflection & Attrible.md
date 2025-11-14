## Reflection
### 反射原理
![[Screenshot from 2023-03-07 12-52-04.png]]
* DLL: 是庫文件無法獨立執行的IL文件
* EXE : 表示可獨立執行的IL文件，具有程式入口。
* metadata : 裡面紀錄IL文件中的結構，所以CLR在載入IL前會先進看metadata才會開始解析IL
> DLL與EXE都是以一個project進行打包的，不是以Solution進行打包的。

Reflection 為 System.Reflection 為.Net平台提供的**幫助類庫**，可以**讀取並使用metadata**
```c#
using System.Reflection;
```
### 反射流程
1. 獲取 dll 建立 Assembly 類型
```c#
// 在編譯路徑output下目錄加載指定dll(最常用) 
Assembly assembly1 = Assembly.Load("Liam.DB.MySql"); 
// 要給定完整路徑 
Assembly assembly2 = Assembly.LoadFile("/home/Document/MyReflection/bin/7.0.102/Debug/Liam.DB.MySql.dll"); 
// 在編譯路徑output下目錄加載指定dll, 需要加上.dll後綴 
Assembly assembly3 = Assembly.LoadFrom("Liam.DB.MySql.dll");
```

2. 取得類型
* 非泛型類型
```c#
Type type = assembly.GetType(typeName);
```
* 泛型類型
```c# 
Type type = assembly.GetType("DB.Server`2"); // `2表示DB.Server類有三個佔位符
Type genericType = type.makeGenericType(new Type[] {typeof(int), typeof(string)});
```

3. 創建對象
* 非泛型類型創建
```c#
// 比較舊的方式
object obj = type.GetConstructor().Invoke();

// 新的方式
// 1. 使用public無參構造函數創建對象
object obj = Activator.CreateInstance(type); 
// 2. 使用private無參構造函數創建對象
object obj = Activator.CreateInstance(type, true); 
// 3. 使用public 有參構造函數
object obj = Activator.CreateInstance(type, new Object[] {20, "p2"}) 
```
> 記得進行類型轉換再使用。

#### 細節
1. Assembly 中有 Type 與 Module等
2. Assembly 是讀取 .dll 與 .exe 文件
3. 取得Type的方式有三種:
	1. `assembly.GetType("typename")` : 從DLL讀取得Type
	2. `typeof(SqlServer)` : 從已經有的類中取得他的Type
	3. `sqlServerObject.GetType()` : 從實例中取得Type

### 使用反射調用方法
為何不使用反射創建對象後再調用方法?
因為我們使用反射時大多都是希望類型可變，故我們會直接使用反射來調用方法。
```c#
// 無參
MethodInfo method = type.GetMethod("Show1");
method.Invoke(type, null); // null 表示沒有參數
// 有參
MethodInfo method = type.GetMethod("Show2");
method.Invoke(type, new object[] {123}); // 要傳type因為他要先建立實例

// 靜態方法
// 1. 傳type -> 先實例，再調用
MethodInfo method = type.GetMethod("Show3");
method.Invoke(type, new object[] {123});
// 2. 不傳type -> 不實例，再調用
method.Invoke(null, new object[] {123});

// 有重載
// GetMethod不知道要找哪個，所以需要傳入參數Type，若沒參傳入new Type[] {}
MethodInfo method = type.GetMethod("Show4", new Type[] {typeof(string), type(int)});
method.Invoke(type, new object[] {123, "Hi"});

// 私有實例無參方法
MethodInfo method = type.GetMethod("Show5", BindingFlags.Instance | BindingFlags.NonPublic)
method.Invoke(type, null);

// 泛型類＋泛型方法
Type type = assembly.GetType("Liam.DB.SqlServer`1");
Type typeGeneric = assembly.MakeGenericType(new Type[] {typeof(int)});
MethodInfon method = typeGeneric.GetMethod("Show"); // 不用佔位符號，泛型不能重載
MethodInfo methodGeneric = method.MakeGenericMethod(new Type[] {typeof(string)});
methodGeneric.Invoke(type, new object[] {"Liam"});
```

### 使用反射調用屬性
```c#
People people = new People();
Type type = type(People); 

PropertyInfo prop = type.GetProperty("Id");
prop.GetInfo(people); // 因為是該實例的屬性
```
#### 使用案例: Object Mapper
以下例子要將從 People 映射到 PeopleDTO
```c#
People p = new People(); // 由一系列操作產生的

Type type1 = typeof(People); // 面向數據庫
Type type2 = typeof(PeopleDTO); // 面向程序
PeopleDTO pDTO = (PeopleDTO) Activator.CreateInstance(type2); // 建立映射對象
foreach (var prop in type2.GetProperties()) {
	object value = type1.GetProperty(prop.Name).GetValue();
	prop.SetValue(pDTO, value);
}
```


## Attribute
特性就是類，但該類必須繼承於 Attribute (直接與間接) ，其可以影響編譯器或者程序執行。
1. 一般會以 Attribute 做結尾，使用 `[]` 時可以不用加上結尾
2. 使用自己建立的Attribute不會對程序有任何的效果，它只會在IL中添加內容，並在Metadata標注出來 
	→ 所以Attribute要與反射使用
> 用於在不給類而外添加屬性與方法時，能夠**補充**該類的特型與功能

#### 常見特性
1. [Obsolete] : 標記程序已經過時，影響編譯器
2. [Serializable] : 表示可以序列化，影響程序運行
3. WebAPI Filter
4. ORM e.g. [table], [key] ...

### 自訂義 Attribute 
#### 定義
```c#
namespace MyAtrribute;

// 修飾該Attribute可以用在Assembly, Class, Method, ...，且可以在同一個位置多次使用相同Attribute
[AttributeUsage(AttributeTarget.Class | AttributeTarget.Method, AllowMutiple = true)] 
public class MyAtrribute: Attribute
{
	private int _id;
	public string Definition {get; set;};
	public MyAttribute() {}
	public MyAttribute(int id) {_id = id; }
	public void Show()
	{
		Console.WriteLine($"id = {_id}");
	}
}
```
* AttributeTarget 是一個枚舉：
	All, Module, Class, Struct, Constructor, Method, Field, Property, Interface, ….
* AttributeUsage的默認為： 
	1. AttributeTarget.All 
	2. AllowMutiple = false 
	3. Inherit = true

#### 標記
```c#
// 聲明方式
[Custom]
[Custom()] // 與第一個相同，使用無參構造函數
[Custom(123)] // 使用有參構造函數
[Custom(123, Description = "123")] // 使用有參構造函數與設定屬性初始化

// 參數與返回值
[return: Custom] // 給返回值使用(很少使用)
public string Answer([Cusom]string name) // 給參數使用
{
	return $"This is {name}";
}
```

#### 使用
```c#
public class Manager
{
	public static void Show(Student student)
	{
		Type type = typeof(Student);
		// 確認該Attribute是否存在
		if(type.IsDefined(typeof(MyAttribute), true)) // 後面的true表示該Attribute的子類也可以
		{
			// 獲取Attribute，這個true與上面一樣
			MyAttribute attribute = type.GetCustomAttribute(typeof(MyAttribute), true) 
																			as MyAttribute 
			attribute.Show();
		}
	}
}
```

>若使用的是方法或者參數的Attribute也都大同小異，只是反射要獲取的對象改調就型
 ParameterInfo, MethodInfo, PropertyInfo都有IsDefined(), GetCustomAttribute()可以獲取Attribute
