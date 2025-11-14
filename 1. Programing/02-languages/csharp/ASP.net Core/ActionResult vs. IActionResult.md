### 簡介
---
作為 Controller 的返回值有兩種選擇，抽象類 ActionResult 與接口 IActionResult。
#### IActionResult
* `IActionResult` 是接口
* 定義操作的結果，允許返回各種不同類型的 HTTP 響應，如 Ok(), NotFound(), BadRequest() 等
> 不用指名返回值類型，也就是前端只知道 body 是一個文本，常用於 Anonymous types 如以下情況

```c#
public IActionResult filter() {
	var brand = new List<string>();
	var type = new List<string>();
	return Ok(new {brand, type});
}
```

* Swagger 下顯示情況![[20231123_10h51m35s_grim.png]]

#### ActionResult
* `ActionResult<T>` 是抽象類
* 許多響應類型（如 ViewResult, FileResult 等）的基類
* 實現 IActionResult 接口
> 需要指定泛型 T，也就是 body 的類型，故前端會知道返回是什麼。

* Swagger 下顯示情況![[20231123_10h50m25s_grim.png]]
