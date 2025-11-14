## Console
---
### 輸出
```c#
Console.BackgroundColor = ConsoleColor.Red;
Console.ForegroundColor = ConsoleColor.White;

Console.WriteLine("Hello world!");
```

### 讀取
```c#
string? name = Console.ReadLine();
Console.WriteLine($"my name is {name}");
```

### 格式化輸出
```c#
// currency 輸出保留小數兩位
Console.WriteLine($"Currency : {24.335:c2}"); // $24.34
// Pad with 0s
Console.WriteLine($"Pad with 0s: {23:d4}"); // 0023
// 小數點位數
Console.WriteLine($"3 Decimals: {24.435555:f3}"); // 24.436
// 千位一個 , 保留小數後 4
Console.WriteLine($"Commas: {2300:n4}"); // 2,300.0000
Console
```


## Random
---
```c#
Random rnd = new Random();
int secretNum = rnd.Next(1, 11); // [1 ~ 11) random integer
double secretNum2 = rnd.NextDouble(); // [0, 1) random double
long secretNum2 = rnd.NextInt64(); 
```


## Exception
---
```c#
try {
    throw new DivideByZeroException("Wrong");
} catch (Exception ex) {
    Console.WriteLine("Message: " + ex.Message);
    Console.WriteLine("StackTrace: " + ex.StackTrace);
}
```


## StringBuilder
---
```c#
// 建立，默認容量
StringBuilder sb = new StringBuilder("Randm Text");
// 建立，指定容量
StringBuilder sb2 = new StringBuilder("More Stuff that ...", 256);
// 添加
sb.Append(" : Hiiii!!!");
sb2.AppendLine("The fist thing...");
sb2.AppendFormat($"Hi {233.4:c4}");
// 清空
sb.Clear();
// 轉換
string s = sb2.ToString();
// 插入
sb2.Insert(11, "that's gread");
// 置換
sb2.Replace("that", "this");
// 刪除
sb2.Remove(11, 5);
```

## Function
---
### 不定長參數
```c#
public void Sum(out int val, params int[]nums) {
	val = 0;
	foreach (int num in nums) {
		val += num;
	}
}

public static void Main(string[] args) {
	Sum(out int val, 1,2,3,4);
	Console.WriteLine(val);
}
```


## DateTime
---
* DateTime : 表示一個時間點
* Timespane : 表示一個 interval
```c# 
// 建立
DateTime myDate = new DateTime(2022, 3, 4);
DateTime today = DateTime.Now;

Console.WriteLine(myDate); // 3/4/2022 12:00:00 AM

// 增減
DateTime newday = today.AddDays(22);
newday = newday.AddYear(-20);

// 使用 timespane: 
TimeSpan ts = new TimeSpan(3, 4, 5, 6);
newday = newday.Add(ts); // 要使用 Add 方法
```

# 註釋
---
```c#
/// <summary>
/// Execute a mapping from the source object to a new destination object.
/// The source type is inferred from the source object.
/// </summary>
/// <typeparam name="TDestination">Destination type to create</typeparam>
/// <param name="source">Source object to map from</param>
/// <returns>Mapped destination object</returns>
TDestination? Map<TDestination>(object? source);
```