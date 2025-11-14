### Promise
Promise 為一個異步操作的結果值，分為 fulfilled 與 rejected。可以使用 Promise 管理異步操作
#### 建立一個 Promise
```ts
const myPromise = new Promise((resolve, reject) => {
	// do something async e.g. request
	if (condition) {
		resolve("success message");
	} else {
		reject("error message")
	}
});
```

#### 進行後續處理
* 方式一: 使用 .then 與 .catch 進行鏈式處裡
* 方式二: 使用 await 與 try...catch.. 進行處裡。
```ts
// method 1:
myPromise
  .then((successMessage) => {
    console.log(successMessage); 
  })
  .catch((errorMessage) => {
    console.log(errorMessage); 
  });
  
// method 2:
try { // promise 成功
	const successMessage = await myPromise;
	console.log(successMessage); 
} catch (error) { // promise 失敗
	console.log(error); 
}
```