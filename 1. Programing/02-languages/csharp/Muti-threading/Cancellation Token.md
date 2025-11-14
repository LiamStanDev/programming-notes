# Intro
---
### What is Cancellation Token ?
Cancellation Token 是用來取消異步操作的令牌，用於不藉由拋出異常的方式來結束一個長期運行的任務。

#### Basic 
Cancellation Token 由一個 object 建立，該 object 有一個或多個異步操做，該 object 將 token 發給所有的異步操作，並且也能將其傳遞到其嵌套的所有異步操作，所有持有 token 需要對其負責任，並對建立 Token 的 object 發起的取消請求做出反應。

### 2 .Net classes
* `CancellationTokenSource`: 用於建立與後續請求取消的物件。
* `CancellationToken`: 被 listener 監聽的結構體。

# 使用
---
### Steps
1. 建立一個 `CancellationTokenSource`
2. 透過 `CancellationTokenSouce.Token` 來傳遞 Token 給所有 listeners (Tasks, Threads)
3. 為每一個 Task or Thread 提供一種機制來回應取消請求
4. 在要取消的情況時調用 `CancellationTokenSource.Cancel()` 方法

```c#
public async Task CancellableMethod() 
{
	var tokenSource = new CancellationTokenSource();
	for (int i = 0; i < 10; i++) 
	{
		Task.Run(() => DoSomeWork(tokenSource.Token), tokenSource.Token);
	}
	// After some delay
	tokenSource.Cancel();
}
```

### Mechanisms
#### 1. Polling
```c#
public static void DoSomeWork(CancellationToken ct)
{
    while(!ct.IsCancellationRequested)
    {
        DoWork();
    }
    // perform cleanup
}
```

#### 2. CallBack
```c#
public static void DownloadSomeHugeFile(CancellationToken ct)
{
    WebClient wc = new WebClient();
    ct.Register(() => // 註冊，並會在 Cancel() 時候被調用
    {
        wc.CancelAsync();
    });
    // optionally can also store this registration in a variable
    wc.DownloadStringAsync("https://some-download-path");
}
```

#### 3. Thread Cancel Exception
```c#
public static void SomeLongRunningOperation(CancellationToken ct)
{
    while(true)
    {
        DoWork(); // perform one unit of work
        ct.ThrowIfCancellationRequested(); // this is extremely fast
    }
}
```

