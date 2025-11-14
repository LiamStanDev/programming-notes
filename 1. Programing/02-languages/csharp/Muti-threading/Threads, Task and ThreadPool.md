### Thread vs. Task
#### Thread
* 為 OS 線程的包裝
* 需要手動的管理其生命週期
* 線程間同步需要小心死鎖問題

#### Task
* Task 可以獨立於其他 Task 運行。
* 非常輕量
* 可以併發的透過 OS 提供的 Thread 執行
* 非常適合異步編程

#### Key differences
* Threading Model:
	* Thread: 由 OS 進行管理
	* Task: 由 .net 運行時進行管理，故對 OS 來說 Task 是透明的，OS 不會感知到 Task 的存在
* Resource Management: 
	* Thread: 需要顯示手動的管理
	* Task: 會自動管理
* Exception Handling: 
	* Thread: 極難管理
	* Task: 提供結構化的方式進行管理

### Task Parallel Library (TPL)
* `Parallel.ForEach`, `Parallel.For`
```c#
Chef[] chefs = { new Chef("Alice"), new Chef("Bob"), new Chef("Charlie"), new Chef("David") };

// 使用 ForEach
Parallel.ForEach(chefs, chef => 
{
	chef.Cook();
});

// 使用 For
Parallel.For(0, chefs.Length, i => 
{
	chefs[i].Cook();
});
```

### PLINQ
* 將 LINQ 轉為平行運算的版本
```c#
Chef[] chefs = { new Chef("Alice"), new Chef("Bob"), new Chef("Charlie"), new Chef("David") };

var result = chefs.AsParallel().Select(chef => 
{
	chef.Cook();
	return chef.Name;
}).ToList();
```