在不使用 ORM 之前，會使用 `DataTable`, `DateSet` 取資料，但會這些做法不是強類型的，也就是 `Rows[0]["ID"]` 我們不知道 Key 的類型也不知道取出的值是甚麼類型。
* ORM 提供的功能:
	* 自動生成 SQL 語句。
	* 容易操作且有高可讀性。
	* 進行必要處理，e.g. 避免 SQL Injection，使用 transtation 等。
* ORM 問題
	* 前置作業過多。
	* 複雜操作 SQL 效能變差。
	* 較難客製化。
# Dapper
---
Dapper 是一個用於 .Net 平台的輕量級 Object Relational Mapping，由 Stack Overflow 開發，提供與資料庫的溝通，**把表對應到 Class，把資料對應到 Object**，讓我們依然可以使用強類型開發，又**可自行撰寫 SQL 語法**。

#### Dapper vs. EntityFramework core
* 性能: Dapper 相比 EF core 更輕量，減少抽象性能更好。
* 控制: 提供直接 SQL 語句編寫，而 EF core 使用 LINQ。
* 複雜性: EF Core 用於複雜數據模型和關係，而使用 Dapper 會比較困難。

