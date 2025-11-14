# 概念
---
## 線程 vs. 協程
* 線程(Threads)：為操作系統提供，一個進程可以有一個或以上的線程，一個線程為程序執行的最小單位，會被操作系統的線程調度器 (Scheduler) 選擇，交給 CPU 執行，操作系統是知道線程的存在的。.Net 提供以下 API
	1. Thread: 代表一個操作系統的線程，提供了直接控制線程的能力
		-> 最靈活。
	2. ThreadPool: 重用一組線程，避免了線程創建和銷毀的開銷。
		-> 適合大量短小的任務。
	3. Task: 基於 Task Parallel Library (TPL) 的，提供了一種基於任務的異步模式。
		-> 底層也是類似於 ThreadPool，本身就代表一個異步操作，可以使用 async / await 是他的最重要的優點，且處理有狀態的異步(返回值)很方便。
	4. Parallel: 基於 TPL，專門用於數據並行和並行迴圈（如 Parallel.For 和 Parallel.ForEach）。
	
* 協程(Coroutine): 不由操作系統提供，為用戶級線程，操作系統本身不知道協程的存在，提供非組塞的執行方式，讓一段程序可以暫停與恢復進行(狀態機)，讓 CPU 可以不用等待，但與多 CPU 執行並不相同。
	* C# 中使用 async / await 來達成協程的效果，使用 await 暫停該方法的執行，async 則告訴編譯器這邊需要實現狀態機，等待 await 方法執行完成後恢復運行。
# Thread
---
### 啟動線程
```c#
Thread thread1 = new Thread(() => {
    Console.WriteLine("Thread1: " + Thread.GetCurrentProcessorId());
    Thread.Sleep(1000);
});


Thread thread2 = new Thread(() => {
    Console.WriteLine("Thread2: " + Thread.GetCurrentProcessorId());
    Thread.Sleep(1000);
});
thread1.Start();
thread2.Start();

Console.WriteLine("Main: " + Thread.GetCurrentProcessorId());
thread1.Join();
thread2.Join();
```

### 改為背景運行
```c#
thread1.IsBackground = true;
```


### 其他重點
* Thread 中的很多方法 e.g. Abort, Suspend, Resume 等都被標記 Deprecated，因為這些操作都是立即性的，還是要取決於操作系統本身。

# 線程同步
---
### 線程間通信 - 共享內存
在 C# 中，lock、Mutex 和 Monitor 都是用於同步多線程訪問共享資源的機制，但它們在用途、行為和適用範圍方面有所不同。
#### Mutex 互斥鎖
* 相較於其他兩個 Mutex 是可以跨進程使用的同步原語。來自操作系統。
```c#
public class CounterMutex {
    private int count = 0;
    private readonly Mutex mutex = new Mutex(); // 互斥鎖

    public void Increament() {
        for (int i = 0; i < 1000000; i++) {
            mutex.WaitOne(); // 獲取鎖
            try {
                count++;
            } finally {
                mutex.ReleaseMutex(); // 釋放鎖
            }
        }
    }
}
```
#### Monitor 對象鎖
* 為工具類用於控制對象鎖的鎖定。相較於操作系統的 Mutex 更輕量
```c#
public class CounterMonitor {
    private int count = 0;
    private readonly object objectLock = new Object();

    public void Increament() {
        for (int i = 0; i < 1000000; i++) {
            Monitor.Enter(objectLock); // 獲取互斥鎖
            count++;
            Monitor.Exit(objectLock);
        }
    }
}
```
> 使用 Stopwatch 測試開啟四個線程，兩者運行時間相差 35 倍
#### lock 對象鎖
* 為 c# 的**語法糖**，使用 Monitor 來實現的。可以省去 Enter 與 Exit
> 被 lock 包裹起來的可看作是一個**原子操作**。

```c#
public class CounterLock {
    private int count = 0;
    private readonly object objectLock = new Object();

    public void Increament() {
        for (int i = 0; i < 1000000; i++) {
            lock (objectLock) {
                count++;
            }
        }
    }
}
```

### 線程間通信 - 消息傳遞 Todo

### 順序控制 - 條件變量
* 條件變量可以使得當變量滿足條件之後，通知其他線程。最常見的場景為生產消費模型。
#### 生產消費模型
![[20230708_23h44m28s_grim.png]]
> 使用互斥鎖可以保證數據一個一個進入緩存隊列，條件變量用於通知生產者。
```c#
public class ProducerConsumerQueue {
    private Queue<int> queue = new();
    private object objectLock = new Object();

    public void Produce(int num) {
        Random rnd = new Random();
        lock (objectLock) {
            for (int i = 0; i < num; i++) {
                int item = rnd.Next(0, 100);
                queue.Enqueue(item);

                Console.WriteLine("Producer Thread id: " + Thread.CurrentThread.ManagedThreadId.ToString());
                Console.WriteLine("Produce item: " + item);
                Thread.Sleep(500);
            }
            Monitor.PulseAll(objectLock); // Pulse: 心跳, 不會釋放當前鎖
        }
    }

    public void Consume() {
        while (true) {
            lock (objectLock) {
                while (queue.Count == 0) { // 避免假喚醒
	                // Wait 會釋放鎖，故不會有死鎖問題
                    Monitor.Wait(objectLock); 
                }

                int item = queue.Dequeue();
                Console.WriteLine("Consumer Thread id: " + Thread.CurrentThread.ManagedThreadId.ToString());
                Console.WriteLine("Consume item: " + item);
                Thread.Sleep(1000);
            }
        }
    }
}
```

* Program.cs
```c#
ProducerConsumerQueue model = new ProducerConsumerQueue();
List<Task> tasks = new List<Task>();

tasks.Add(Task.Run(() => model.Produce(3)));
tasks.Add(Task.Run(() => model.Produce(2)));
tasks.Add(Task.Run(() => model.Produce(1)));
tasks.Add(Task.Run(() => model.Consume()));
tasks.Add(Task.Run(() => model.Consume()));
tasks.Add(Task.Run(() => model.Consume()));
tasks.Add(Task.Run(() => model.Consume()));
tasks.Add(Task.Run(() => model.Consume()));
tasks.Add(Task.Run(() => model.Produce(10)));

Task.WaitAll(tasks.ToArray());
```

* 為什麼 Monitor.Palse 要寫在 lock 裡面?
	只有當當前線程持有鎖時，才能夠修改共享資源和發送信號。這防止了同時對共享資源進行讀寫操作的競爭條件。

### 最大併發數 - Semaphore 信號量 Todo 
* 限制可以同時訪問某一共享資源的線程數量，控制著有限數量的線程可以同時進入。
* Semaphore 為守衛，要先取得許可證才能進入
	* 第一個參數 initialCount: 當信號量創建時立即可用的許可證數量
	* 第二個參數 maximumCount: 最大許可證數量
```c#
public class Semo {
    private static Semaphore semaphore = new Semaphore(3, 5);

    public void DoWork() {
        semaphore.WaitOne(); // 取得許可證
        Console.WriteLine($"Thread id {Thread.CurrentThread.ManagedThreadId.ToString()},  Working...");
        Thread.Sleep(10000);
        Console.WriteLine("Down!");
        semaphore.Release();
    }
}
```
* Program.cs
```c#
Semo semo = new Semo();
List<Task> tasks = new List<Task>();

for (int i = 0; i < 10000; i++) {
    tasks.Add(Task.Run(() => semo.DoWork()));
}

Task.WaitAll(tasks.ToArray());
```

# ThreadPool
---
### 啟動線程
* 為 Backgroun Thread，使用 QueueUserWorkItem 就會直接從 ThreadPool 使用一個線程運行。
```c#
// t 參數不能省略
ThreadPool.QueueUserWorkItem((t) => {
    Console.WriteLine("Thread: " + Thread.GetCurrentProcessorId() + " Start");
    Thread.Sleep(1000);
    Console.WriteLine("Thread: " + Thread.GetCurrentProcessorId() + " End");
});

ThreadPool.QueueUserWorkItem((t) => {
    Console.WriteLine("Thread: " + Thread.GetCurrentProcessorId() + " Start");
    Thread.Sleep(1000);
    Console.WriteLine("Thread: " + Thread.GetCurrentProcessorId() + " End");
});
```
### 配置
```c#
ThreadPool.SetMaxThreads(1000, 1000);
ThreadPool.SetMinThreads(1000, 1000);
```

# Task
---
### 啟動方式
```c#
// method 1: 最常使用
Task.Run(() =>
{
    Console.WriteLine($"Thread: {Thread.CurrentThread.ManagedThreadId} start!");
    Thread.Sleep(5000);
    Console.WriteLine($"Thread: {Thread.CurrentThread.ManagedThreadId} end!");
});

// method 2: 創建對象(可以給名子)
Task myTask = new Task(() =>
{
    Console.WriteLine($"Thread: {Thread.CurrentThread.ManagedThreadId} start!");
    Thread.Sleep(5000);
    Console.WriteLine($"Thread: {Thread.CurrentThread.ManagedThreadId} end!");
}, "myTask");
myTask.Start();
Console.WriteLine(myTask.AysncState)


// method 3: 使用工廠(有很多好用的方法)
TaskFactory myFactory = new TaskFactory();
myFactory.StartNew(() =>
{
    Console.WriteLine($"Thread: {Thread.CurrentThread.ManagedThreadId} start!");
    Thread.Sleep(5000);
    Console.WriteLine($"Thread: {Thread.CurrentThread.ManagedThreadId} end!");
});
```

### APIs
#### WaitAll
* 組塞當前線程，等待所有 Task 完成，可以設定超時等待
```c#
List<Task> tasks = new();

tasks.Add(Task.Run(() => {
    Thread.Sleep(1000);
    Console.WriteLine("Hi");
}));
tasks.Add(Task.Run(() => {
    Thread.Sleep(1000);
    Console.WriteLine("Hi2");
}));
tasks.Add(Task.Run(() => {
    Thread.Sleep(1000);
    Console.WriteLine("Hi3");
}));

Task.WaitAll(tasks.ToArray());
```

#### WaitAny
* 組塞當前線程，等待其中一個 Task 完成，可以設定超時等待，沒有返回值。
* 應用：多個下載點，哪個先下載好就用那個
```c#
tasks.Add(Task.Run(() => {
    Thread.Sleep(100);
    Console.WriteLine("Hi");
}));
tasks.Add(Task.Run(() => {
    Thread.Sleep(100000);
    Console.WriteLine("Hi2");
}));
tasks.Add(Task.Run(() => {
    Thread.Sleep(2000);
    Console.WriteLine("Hi3");
}));

Task.WaitAny(tasks.ToArray());
```

#### WhenAll / WhenAny + ContinueWith
* 不會阻塞當前線程，會返回 Task 與 ContinueWith 一起用，可以配置後續任務
```c#
List<Task> tasks = new();

tasks.Add(Task.Run(() => {
    Thread.Sleep(100);
    Console.WriteLine("Hi");
}));
tasks.Add(Task.Run(() => {
    Thread.Sleep(100000);
    Console.WriteLine("Hi2");
}));
tasks.Add(Task.Run(() => {
    Thread.Sleep(2000);
    Console.WriteLine("Hi3");
}));

Task task = Task.WhenAny(tasks.ToArray())
    .ContinueWith((t) => {
        Console.WriteLine("Finished");
    });

task.Wait();
```

#### Delay + ContinueWith
* Delay 與 Sleep 不同，會返回一個 Task，用來延遲後續事件。 
```c#
Task.Delay(2 * 1000).ContinueWith(() => { Console.WriteLine("Trigger"); })
```

### 控制線程數量
#### 全局
```c# 
ThreadPool.SetMaxThreads(12, 12);
ThreadPool.SetMinThreads(6, 6);
```
#### 局部
* 自己寫
```c#
public static void RunTaskAtMaxThreadsNum(List<Action> tasks, int maxThreads)
{
	// 避免卡主線程
	Task.Run(() => 
	{
		List<Task> taskList = new List<Task>();
		foreach (var t in tasks)
		{
			taskList.Add(t.Run(() => Action.Invoke());
			if(taskList.Count > maxThreads)
			{
				Task.WaitAny(taskList.ToArray());
				taskList = taskList.Where(t => t.Status != TaskStatus.RanToCompletion).ToList();
			}
		}
		Task.WhenAll(taskList.ToArray());
	}
}
```
# Parallel
---
有 `For` 與 `ForEach` 兩個方法。
### 使用

#### For
##### 簽名
```c# 
public static ParallelLoopResult For(int fromInclusive, int toExclusive, Action<int, ParallelLoopState> body) {
```
* body 的參數
	1. 為 For 迴圈的局部變量值也就是 i
	2. 為用於控制迴圈行為，e.g. Stop, Break，但經測試並不可靠。
1. 

* 這邊使用 Semaphore 的例子
```c#
Parallel.For(1, 1000, (i, state) => semo.DoWork());
```
#### ForEach
```c#
List<int> list = new() {12, 24, 52};

Parallel.Foreach(list, (num, state) => DoWork(num));
```

# Async / Await
async / awiat 是 C# 用於簡化異步編程的關鍵字，與 Task 一起使用。
### async
* 意義：為修飾符，用於標記方法、lambda 表達式或匿名方法為異步的
* 方法的返回值：
	1. 無法返回值： `Task`
	2. 有返回值： `Task<T>`
#### 底層概念
1. 轉為狀態機
	編譯器將 async 方法轉換成狀態機，這個狀態機能夠管理該異步操作的不同階段 ，像是 js Promise 的 Resolve, Reject, Pending。方法中的被分割程 await 之前與 await 之後。
2. 封裝成 Task	
3. 異常處理
	編譯器會將異步方法中的異常捕獲並封裝到返回的 Task 中。
### await
* 意義：用於異步方法的內部，用於暫停當前方法的執行，釋放線程去做別的事情，等到 Task 完成之後再繼續執行。
* 行為：當 await 一個 Task 時，當前方法的剩餘操作被封裝為繼續操作並返回，控制權交還給方法的調用者。
