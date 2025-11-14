## Client API
---
官網推薦使用 [Npgsql ADO.NET data provider](https://www.nuget.org/packages/Npgsql)，允許您使用 ADO.NET 訪問 PostgreSQL 數據庫。

## 使用
---
* 幾乎所有方法都有 Async 版本
### 建立連接
```c#
// 數據庫連接字符串
string connectionString = "Host=localhost;Port=5432;Username=postgres;Password=870107;Database=sales;";
using (var connection = new NpgsqlConnection(connectionString)) {
	// 開始連接
    await connection.OpenAsync();
    // .. 
}
```
### 執行 SQL 命令
#### 查詢
```c#
using Npgsql;

string query = "SELECT * FROM projects";
using (NpgsqlCommand command = new NpgsqlCommand(query, connection)) {

	using (NpgsqlDataReader dataReader = await command.ExecuteReaderAsync()) {
		while (dataReader.Read()) {
			// 0, 1, 2 表示第幾個 Column
			Console.Write(dataReader.GetInt32(0) + " ");
			Console.Write(dataReader.GetString(1) + " ");
			Console.Write(dataReader.GetInt32(2));
			Console.WriteLine();
		}
	}
}
// 執行 SQL 語句
```
```text
結果：
21 Project Alpha 1
22 Project Beta 2
23 Project Gamma 3
24 Project Delta 4
25 Project Epsilon 5
26 Project Zeta 6
27 Project Eta 7
28 Project Theta 8
29 Project Iota 9
30 Project Kappa 10
31 Project Lambda 11
```
#### 插入、更新與刪除
```c#
query = "INSERT INTO projects (name, employee_id) VALUES (@n, @e_id)";

using (NpgsqlCommand command = new NpgsqlCommand(query, connection)) {
	command.Parameters.AddWithValue("@n", "Project Flight");
	command.Parameters.AddWithValue("@e_id", 4);
	int result = await command.ExecuteNonQueryAsync();
	Console.WriteLine($"result: {result}");
}

```


### 事物
> 注意：IsolationLevel 是來自 `System.Data` 而不是 `System.Transaction`
```c#
// 在連接上建立事物，且設定隔離等級
using (NpgsqlTransaction transaction = await connection.BeginTransactionAsync(IsolationLevel.ReadCommitted)) {
	try {

		query = "INSERT INTO projects (name, employee_id) VALUES (@n, @e_id)";
		using (NpgsqlCommand command = new NpgsqlCommand(query, connection, transaction)) {
			command.Parameters.AddWithValue("@n", "Project Flight");
			command.Parameters.AddWithValue("@e_id", 5);
			int result = await command.ExecuteNonQueryAsync();
			Console.WriteLine($"result: {result}");
		}
		// 以下 query 會失敗
		query = "INSERT INTO projects (nam, employee_id) VALUES (@n, @e_id)";
		using (NpgsqlCommand command = new NpgsqlCommand(query, connection, transaction)) {
			command.Parameters.AddWithValue("@n", "Project Flight");
			command.Parameters.AddWithValue("@e_id", 4);
			int result = await command.ExecuteNonQueryAsync();
			Console.WriteLine($"result: {result}");
		}
	} catch (Exception ex) {
		Console.WriteLine(ex.Message);
		Console.WriteLine(ex.StackTrace);
		await transaction.RollbackAsync();
	}
}

```