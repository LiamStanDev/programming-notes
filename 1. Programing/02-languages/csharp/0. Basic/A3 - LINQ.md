# 基本介紹
---
LINQ 主要用於實作了 `IEnumerable<T>` 或者 `IQueryable<T>` 的類型，LINQ 採用 Extension Method 的方始讓我們可以直接調用。

### 執行時間
* 返回單一數值 (Average, Sum...): 會立刻計算
* 返回 Sequence: 會延後到要使用時才會計算。
### IEnumerable vs IQueryable
* IEnumerable: 對象為 In-memory Collection
* IQueryable: 會依據 LINQ 使用 expression tree 生成對應的 query，交給數據庫執行。

### Storing result in memory
LINQ 返回的返回值是一系列對於數據的操作 (Instructions)，有以下語句可以使其真正執行並存放在 ram 中。
1. foreach statement
2. ToList
3. ToArray
4. ToDictionary
5. [ToLookup](https://www.javatpoint.com/linq-tolookup-method#:~:text=ToLookup%20operator%20in%20LINQ%20is,that%20matched%20with%20the%20Key.)

### 分類
* 篩選： `Where`
* 投影： `Select`, `SelectMany`, `Zip`
* 排序： `OrderBy`, `OrderByDescending`
* 聚合： `Sum`, `Average`, `Min`, `Max`
* 分組： `GroupBy`
* 連接： `Join`
* 分頁： `Skip` (psql: offset), `Take` (psql: limit)
* 集合： `Distinct`, `Except`, `Union`, `Intersect`

# 使用
---
### Filtering
也就是 Where 語句。
```c#
string[] words = { "the", "quick", "brown", "fox", "jumps" };  
  
var query1 = from word in words  
             where word.Length == 3  
             select word;  
  
var query2 = words.Where(w => w.Length == 3);
```

### Projection
分別有 Select, SelectMany 與 Zip 三種。
#### Select
```c#
class Bouquet
{
    public List<string> Flowers { get; set; }
}

static void SelectVsSelectMany()
{
    List<Bouquet> bouquets = new()
    {
        new Bouquet { Flowers = new List<string> { "sunflower", "daisy", "daffodil", "larkspur" }},
        new Bouquet { Flowers = new List<string> { "tulip", "rose", "orchid" }},
        new Bouquet { Flowers = new List<string> { "gladiolis", "lily", "snapdragon", "aster", "protea" }},
        new Bouquet { Flowers = new List<string> { "larkspur", "lilac", "iris", "dahlia" }}
    };

    IEnumerable<List<string>> query1 = bouquets.Select(bq => bq.Flowers);
    // equal to 
    // var query2 = from bq in Bouquet
	//              select bq.Flowers.

	// need two for loop because it's two-dimensional
    foreach (IEnumerable<String> collection in query1)
        foreach (string item in collection)
            Console.WriteLine(item);
}
```

#### SelectMany: 多進行一次 from 
```c#
    IEnumerable<List<string>> query1 = bouquets.SelectMany(bq => bq.Flowers);
    // equal to 
    // var query2 = from bq in Bouquet
	//              from flower in bq.Flowers
	//              select flower

	// need only one for-loop because it's one-dimensional
    foreach (IEnumerable<String> item in query1)
            Console.WriteLine(item);
```

#### Zip
其返回 `IEnumerable<TFirst, TSecond>`，函數簽名如下
```c#
IEnumerable<TFirst, TSecond> Enumerable.Zip<TFirst, TSecond>(IEnumerable<TFirst>, IEnumerable<TSecond>)
```
案例:
```c#
// An int array with 7 elements.
IEnumerable<int> numbers = new[]
{
    1, 2, 3, 4, 5, 6, 7
};
// A char array with 6 elements.
IEnumerable<char> letters = new[]
{
    'A', 'B', 'C', 'D', 'E', 'F'
};

foreach ((int num, char letter) in numbers.Zip(letters)) {
	Console.WriteLine($"Number: {num}, Letter: {letter}")
}
```

### Set Operations
集合操作有以下幾個
1. Distinct: 取唯一
2. Except: 差集
3. Intersect: 交集
4. Union: 聯集
以上方法都有以 By 結尾的方法，如 DistinctBy, ExceptBy，而結尾有 By 的需要傳入 KeySelector 是一個 Func，這個函數用在判斷元素是否相同 (comparative discriminator)。
```c#
string[] words = new[] { "an", "apple", "a", "day", "the", "quick", "brown", "fox" };
words.Distinct(); // 按照 string == string 的方式進行判斷相同
words.DistinctBy(w => w.Length); // 依照自己設定的方式進行判斷相同
```