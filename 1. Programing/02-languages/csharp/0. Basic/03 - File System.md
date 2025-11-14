# 簡介
---
命名空間為 `SystemIO`，主要使用裡面的 static class
* Path
* Directory
* File
以及其實例
* DirectoryInfo
* FileInfo
# Directory 
---
### 介紹
* `Directory`: 為 static class 提供一系列**static method** 操作目錄。
* `DirectoryInfo`: 為 nonstatic class 提供一系列 **instance method** 操作目錄。
簽名如下
```c#
public sealed class DirectoryInfo : FileSystemInfo {}
```
> FileSystemInfo 提供 
> 1. Attribute 屬性: 用來判斷是哪種類型的文件。
> 2. CreationTime 屬性: 創建時間
### 使用
#### 屬性
##### DirectoryInfo
```c#
DirectoryInfo currDir = new DirectoryInfo(".");
Console.WriteLine(currDir.FullName); // /home/liam/Documents/CSharp/demo01
Console.WriteLine(currDir.Name); //  demo01
Console.WriteLine(currDir.Parent); // /home/liam/Document/CSharp
Console.WriteLine(currDir.Attributes); // Directory
Console.WriteLine(currDir.CreationTime); // 12/10/2023 9:00:30 PM
```

#### 操作
##### Directory
```c#
string curPath = Directory.GetCurrentDirectory();
string logDir = Path.Combine(curPath, "log");

if (!Directory.Exist(logDir)) {
	Directory.CreateDirectory(logDir);
	string newLogDir = Path.Combine(curPath, "mylog");
	Directory.Move(logDir, newLogDir);
	Directory.Delete(newLogDir);
}
```

##### DirectoryInfo
```c#
DirectoryInfo dataDir = new DirectoryInfo("./MyData");

if (!dataDir.Exist) {
	dataDir.Create();
	dataDir.MoveTo("./Data"); // 可用來改名
} else {
	dataDir.Delete();
}
```

# File I/O
---
* `File` 類
* `FileInfo` 類
> 關係同 Directory 與 DirectoryInfo
### 查找文件
```c#
// 查找該目錄下所有文件名稱以 .txt 結尾的文件
FileInfo[] files = Directory.GetFiles("./Data", "*.txt", SearchOption.AllDirectories);
```

### 文件操作
```c#
File.Copy(fileName, fileNameCopy); 
File.Move(fileName, fileNameMove); 
File.Delete(fileName);
```
### 寫入文件
* Flush 是什麼？
	若是一個字節寫入磁盤會導致寫入沒有效率，故會先存放在 Cache 中，等到緩存滿了才會一次性寫入，Flush 是叫系統直接寫入。
* 方法命名：
	* Create: 創建或覆蓋式寫入
	* Append: 追加式
	* Text: 字符式
```c#
string logFile = Path.Combine(logDir, "log.txt");
if (!File.Exist(fileName)) {
	if (!Directory.Exist(logDir)) {
		Directory.Create(logDir);
	}
	// 覆蓋式字節寫入
	using (FileStream fs = File.Create(logFile)) {
		string data = "123456";
		byte[] bytes = Encoding.UTF8.GetBytes(data);
		// fs.Write(bytes, 0, bytes.Length);
		// fs.WriteBytes(bytes);
		await fs.WriteAsync(bytes, 0, bytes.Length);
		// fs.Flush();
		await fs.FlushAsync();
	}
	// 添加式字節寫入
	using (FileStream fs = File.Append(logFile)) {
		string data = "123456";
		byte[] bytes = Encoding.UTF8.GetBytes(data);
		// fs.Write(bytes, 0, bytes.Length);
		// fs.WriteBytes(bytes);
		await fs.WriteAsync(bytes, 0, bytes.Length);
		// fs.Flush();
		await fs.FlushAsync();
	}
	// 覆蓋式uft-8寫入
	using (StreamWriter sw = File.CreateText(fileName)) {
		string data = "123456";
		// sw.Write(data);
		await sw.WriteAsync(data); // 添加在後面
		// sw.WriteLine(data);
		await sw.WriteLineAsync(data); // 在新的一行寫入。
		// sw.Flush();
		sw.FlushAsync();
	}
	// 添加式utf-8寫入
	using (StreamWriter sw = File.AppendText(fileName)) {
		string data = "123456";
		// sw.Write(data);
		await sw.WriteAsync(data); // 添加在後面
		// sw.WriteLine(data);
		await sw.WriteLineAsync(data); // 在新的一行寫入。
		// sw.Flush();
		sw.FlushAsync();
	}
}
```

### 文件讀取
#### 一次性讀取
```c#
// 一行一行讀取
// string[] lines = File.ReadAllLines(fileName);
string[] lines = await File.ReadAllLinesAsync(fileName);

// 一次性讀取
// string fullText = File.ReadAllText(fileName); // 字符
// byte[] fullBytes = File.ReadAllBytes(fileName); // 字節
string fullText = await File.ReadAllTextAsync(fileName); // 字符
byte[] fullBytes = await File.ReadAllBytesAsync(fileName); // 字節
```
#### 分批讀取
```c#
// 字節
using (FileStream fs = File.OpenRead(fileName)) {
	int bytesRead = 0;
	byte[] buffer = new byte[10];
	do {
		// bytesRead = fs.Read(buffer, 0, buffer.Length);
		bytesRead = await fs.ReadAsync(buffer, 0, buffer.Length);
	} while (bytesRead > 0);
}
// 字符
using (StreamReader sr = File.OpenText(fileName)) {
	int textRead = 0;
	char[] buffer = new buffer[10];
	do {
		// textRead = fs.Read(buffer, 0, buffer.Length);
		textRead = await fs.ReadAsync(buffer, 0, buffer.Length);
	} while (textRead > 0);
}
```

## Driver
```c#
DriveInfo[] drivers = DriveInfo.GetDrives();
foreach (DriveInfo drive in drivers) {
    if (drive.IsReady) {
        Console.WriteLine(drive.DriveType);
        Console.WriteLine(drive.Name);
        Console.WriteLine(drive.DriveFormat);
        Console.WriteLine(drive.TotalSize);
        Console.WriteLine(drive.TotalFreeSpace);
        Console.WriteLine();
    }
}
```


# Stream
---
stream 是所有的數據流操作(文件、網路、內存)的底層基礎。

### 字節流
以字節為單位處理數據 e.g. 文本、圖像、音樂、影片等
* `Stream`：為以下三個類的抽象基類
* `FileStream`
* `MemoryStream`
* `NetworkStream`
> 以上的類都包含讀寫

### 字符流
以字符流為單位處理文字數據。與字節流不同，將讀寫分為兩個類。
* `StreamReader`
* `StreamWriter`
