## Document Object Model
---
是一個編程接口，用於表示和與網頁互動，將網頁轉化為一個包含各種對象的樹狀結構，當瀏覽器加載網頁時，它會解析HTML文檔，創建一個DOM，這樣程序和腳本就能動態訪問和修改頁面的內容、結構和樣式。
```html
<head>...</head>
<body>
	<div id="block">
	</div>
	<div id="container">
		<p>...</p>
		<p class="item">...</p>
		<p>...</p>
	</div>
	<script src=".../index.js"></script>
</body>
```
### 獲取元素
```js
// 單一元素
var block = document.getElementById("block") // 較少使用
var block2 = document.querySelector(".item") // HTML5 新方法

// 多個元素
var contents = document.querySelectorAll("#container p"); // 取得 #container 下的所有 p 標籤

// 獲取該元素相鄰標籤
var prevBlock = block.previousElementSibling;

var nextBlock = block.nextElementSibling;

var father = block.parentNode;

var children = block.children;
```

### 元素操作
```js
// 文本內容修改
// 1. 輸出文本內容 
block.textContent = "Hi"; // 會覆蓋裡面的所有p標籤
// 2. 可以輸出文本內容與標籤 (性能損耗)
block.innerHTML = 'normal content<span class="bold-text">bold text</span>'

// 修改樣式
// 1. 直接修改css內容 (少用)
block.style.width = "80px";
block.style.backgroundColor = "tomato"
// 2. 更改 class
block.className = "changeStyle" // 不用 .changeStyle

// 事件處裡
// 1. 傳統方式
block.onclick = () => {
	alert("suprise");
}
// 2. 添加事件監聽 (可以重複添加避免覆蓋，更好用)
block.addEventListener("click", () => {
	alert("suprise agina");
});
```

* 常見事件處裡

|Event|Description|
|--|--|
|onchange|An HTML element has been changed|
|onclick|The user clicks an HTML element|
|onmouseover|The user moves the mouse over an HTML element|
|onmouseout|The user moves the mouse away from an HTML element|
|onkeydown|The user pushes a keyboard key|
|onload|The browser has finished loading the page|

## Browser Object Model
---
是指與瀏覽器進行交互的一組對象，關注的主要是瀏覽器而非文本內容，主要有以下對象:
1. window
	1. 窗口開啟關閉
		1. window.open()
		2. window.close()
	2. 調整窗口大小
		1. window.resizeTo()
	3. 定時器
		1. setTimeout()
		2. setInterval()
	4. 獲取窗口尺寸
		1. window.innerWidth
		2. window.innerHeight
2. location
	1. 獲取或設置當前頁面url
		1. location.href
	2. 重定向
		1. location.replace()
		2. 或修改 location.href 
	3. 獲取ulr的細節
		1. location.hostname
		2. location.pathname
3. navigator
	1. 瀏覽器檢測(獲得瀏覽器用戶代理字符號串)
		1. navigator.userAgent
	2. 獲取操作系統
		1. navigator.platform
4. screen
	1. 屏幕信息: 
		1. screen.width
		2. screen.height
	2. 顏色深度:
		1. screen.colorDepth
5. history
	1. 導覽歷史
		1. history.back()
		2. history.forward()