# 簡介
---
### 應用場景
給定 pattern 後可以
1. 尋找符合的字串
2. 驗證文字格式是否符合
3. 擷取、編輯、取代或刪除子字串

### 運作方式
透過 .Net 提供的 `System.Text.RegularExpressions.Regex` 作為工具庫，要進行文字處理需要提供：
1. Pattern
2. 待剖稀的字符串

#### 常用方法

| 方法簽名                             | 功能       |
| -------------------------------- | -------- |
| `Regex.IsMatch`                  | 返回批配是否成功 |
| `Regex.Match` or `Regex.Matches` | 批配後返回結果  |
| `Regex.Replace`                  | 用來取代與刪除  |

# 正則表達式
---
### 中介字元 (Metacharacters)

| 中介字元    | 說明           |
| ------- | ------------ |
| `[]`    | 字元集合         |
| `\`     | 轉意符號         |
| `.`     | 除了`\n`以外的符號  |
| `^`     | 字串開頭         |
| `$`     | 字串結尾         |
| `*`     | 出現任意次數(包含0)  |
| `?`     | 出現 0 或 1 次   |
| `+`     | 出現至少一次以上     |
| `{m,n}` | 指定出現次數 m~n 次 |
| `\`     | 表示 or        |
| `()`    | 將括弧內形成一體     |
### 特別序列 (Special Squences)

| 特別序列 | 說明                              |
| ---- | ------------------------------- |
| `\d` | 數字                              |
| `\D` | 非數字                             |
| `\w` | 任意文字，包含數字與下劃線                   |
| `\W` | 非文字，如空格、標點符號、特殊字符 `@` `#` `$` 等 |
| `\s` | 任意空白符號如空白與換行                    |
| `\S` | 非空白符號                           |
| `\b` | 單詞邊界，表示 `\b` 與 `\b` 包圍的是一個完整的單詞 |

# 使用
---
##### 範例一：提取郵件地址
```cs
string input = "Contact us at support@example.com or sales@example.com.";
// 其中 \b 劃分出單詞邊界
string pattern = @"\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,7}\b";

MatchCollection matches = Regex.Matches(input, pattern);

foreach (Match match in matches) {
	Console.WriteLine("Found Email: " + match.Value);
}
```

##### 範例二：驗證字符串是否為有效電話號碼
```cs
string input = "+1-800-555-0123";
string pattern = @"^\+?[1-9]\d{1,14}$";

bool isValid = Regex.IsMatch(input, pattern);
Console.WriteLine("Is valid phone number: " + isValid); // False
```

##### 範例三：替換所有空格為 `_`
```cs
string input = "This is a sample text.";
string pattern = @"\s";
string replacement = "_";

string result = Regex.Replace(input, pattern, replacement);
Console.WriteLine(result);
 ```
 
# Reference
---
https://hackmd.io/@aaronlife/regular-expression