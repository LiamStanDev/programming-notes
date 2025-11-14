# Sqlx
---
### 特點
* **Pure Rust**
* **Truly asynchronouse**
* **Runtime agnostic**: works on `async-std`, `tokio`, `actix`
	* TLS backend: `native-lts`, `rustls`
* **Built-in connection pooling**: `sqlx:Pool`
* **Auto statement preparation and caching** when using `sqlx::query`

### 操作
#### Connection
* Connection Pool
```rust
let pool = PgPoolOptions::new()
	.max_connections(5)
	.connect("postgres://postgres:password@localhost:5432/test")
	.await?;
```

* Single
```rust
let conn = PgConnection::connect("postgres://postgres:password@localhost:5432/test").await?;
```


#### Quering
SQL Query 分為:
* `prepared`(parameterized):  sqlx 中類型為 `Query`, `QueryAs`
	* 採用 query plan cache(資料庫不需要重新規劃該怎麼執行也就是不用重新解析)
	* 二進制模式溝通
	* 參數化避免 SQL injection
* `unprepared`:  `&str` 會被視為 `unprepared`
	* 簡單的語句
	* 適用於無法被 prepare 的指令（如：BEGIN、PRAGMA、SET 等）

> **Query Plan Caching 是什麼？**
> 每次執行 SQL 前，資料庫需經過：
        1. Parsing：解析語法
        2. Analyzing：分析參數、結構
        3. Planning：建立執行計畫（index、join、hash 等）
    故 prepared 查詢會 cahce query plan，之後相同查詢（不同參數）可直接重用，可以大幅節省 CPU 與提升效能，且避免 SQL injection

```rust
// low-level
use sqlx::Executor; // 需包含這個 trait
conn.execute("BEGIN").await?; // unprepared, simple query
pool.execute(sqlx::query("DELETE FROM table")).await?; // prepared


// high-level: 為了方便使用 query 都有 executor 不用引入 Executor trait
sqlx::query("DELETE FROM table").execute(&mut conn).await?;
sqlx::query("DELETE FROM table").execute(&pool).await?;
```

##### 返回值
| 方法                 | 回傳型別                                          | 說明                                                                    |
| ------------------ | --------------------------------------------- | --------------------------------------------------------------------- |
| `execute()` ✅ 常用   | `DB::QueryResult`                             | 執行非查詢型 SQL（如 `INSERT`、`UPDATE`、`DELETE`），可用 `.rows_affected()` 取得影響行數 |
| `fetch()` ✅ 常用     | `BoxStream<'_, Result<DB::Row, sqlx::Error>>` | 非同步串流，適合逐筆讀取大量結果（需搭配 `TryStreamExt` 使用）                               |
| `fetch_one()` ✅ 常用 | `DB::Row`                                     | 取得一筆結果，若沒有或多於一筆會拋出錯誤                                                  |
| `fetch_optional()` | `Option<DB::Row>`                             | 最多回傳一筆結果，沒有結果時回傳 `None`，比 `fetch_one()` 更安全                           |
| `fetch_all()`      | `Vec<DB::Row>`                                | 一次性載入所有結果為向量，適合小筆數查詢                                                  |

##### 逐筆獲取
```rust
use futures_utils::TryStreamExt; // 可以使用 try_next
use sqlx::Row; // 可以使用 try_get

let mut  rows = sqlx::query("SELECT * FROM users WHERE email = ?")
	.bind(email)
	.fetch(&mut conn); // 不需要 await

while let Some(row) = rows.try_next().await? {
	// map into user-defined domain type
	let email: &str = row.try_get("email")?;
}
```

##### 映射 row -> struct
* 方式一： `.map()`
```rust
let mut stream = sqlx::query("SELECT * FROM users")
	.map(|row: PgRow| {
		// mapping
	})
	.fetch(&mut conn);
```

* 方式二：`.query_as()`
```rust
#[derive(sqlx::FromRow)]
struct User { name: String, id: i64 }

let mut stream = sqlx::query_as::<_, User>("SELECT * FROM users WHERE email = ? OR name = ?")
    .bind(user_email)
    .bind(user_name)
    .fetch(&mut conn);
```


##### Compile-time 驗證：query!() & query_as!()
* 優點:
	* 編譯期型別檢查，防止 SQL 拼錯、欄位漏接等問題
	* 可推導出欄位型別與命名
* 缺點:
	* 需要 DATABASE_URL 環境變數，並能實際連線
	* 編譯時間略久

首先建立 `.env` sqlx 會自動獲取
```text
DATABASE_URL=mysql://localhost/my_database
```

* `query!()`
```rust
let countries = sqlx::query!(
        "
SELECT country, COUNT(*) as count
FROM users
GROUP BY country
WHERE organization = ?
        ",
        organization
    )
    .fetch_all(&pool) // -> Vec<{ country: String, count: i64 }>
    .await?;
```

* `query_as!()`
```rust
// 不需要添加 trait
struct Country { country: String, count: i64 }

let countries = sqlx::query_as!(Country,
        "
SELECT country, COUNT(*) as count
FROM users
GROUP BY country
WHERE organization = ?
        ",
        organization
    )
    .fetch_all(&pool) // -> Vec<Country>
    .await?;
```