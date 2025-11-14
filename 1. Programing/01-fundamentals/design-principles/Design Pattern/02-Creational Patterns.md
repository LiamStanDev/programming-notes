創建型設計模式: 依據指定的要求或者控制流程來建立對象。
# 工廠模式
---
工廠模式本質就是使用工廠方法代替 new 操作創建實例對象的方式，**目的是依賴解耦**。分為簡單工廠、工廠方法與抽象工廠。
用於本類中要創建依賴類的實例，該實例可以有多種，我們將其創建隔離開來。

> 目的: **隔離變化**

### 引入概念
我們希望寫一個 SqlHelper 來實現對於數據庫的操作，目標是希望她可以用於不同的數據庫。
#### Version 1
* SqlHelper.cs
```cs
public class SqlHelper
{
	// 可以從配置文件讀取
	private static readonly string _connectionString;
    private static readonly string _dbType;
	  // 實現功能
    public int ExecuteNonQuery(string sql)
    {
        using (NpgsqlConnection conn = new NpgsqlConnection(_connectionString))
        {
            NpgsqlCommand cmd = new NpgsqlCommand(sql, conn);
  
            try
            {
                conn.Open();
                return cmd.ExecuteNonQuery();
            }
            catch (Exception ex)
            {
                Console.WriteLine(ex.Message);
                return 0;
            }
        }
   }
}
```
* 問題:
	* SqlHelper 高度依賴特定的數據庫實例。
#### Version 2
我們將尋找其基類進行替換，遵循依賴倒置原則。
* SqlHelper.cs
```cs
public int ExecuteNonQuery(string sql)
{
		// NpgsqlConnection -> DbConnection
		using (DbConnection conn = new NpgsqlConnection(_connectionString))
		{
				DbCommand cmd = conn.CreateCommand(); // NpgsqlCommand -> DbCommand
				cmd.CommandText = sql;

				try
				{
						conn.Open();
						return cmd.ExecuteNonQuery();
				}
				catch (Exception ex)
				{
						Console.WriteLine(ex.Message);
						return 0;
				}
		}
}
```
* 問題:
	* 我們在 new 時候還是會依賴 `NpgsqlConnection`

#### Version 3
我們將會變化的隔離開來，單獨用一個方法包裝。
```cs
public int ExecuteNonQuery(string sql)
{
    using (DbConnection conn = CreateDbConnection()) // 調用方法
    {
        DbCommand cmd = conn.CreateCommand();
        cmd.CommandText = sql;

        try
        {
            conn.Open();
            return cmd.ExecuteNonQuery();
        }
        catch (Exception ex)
        {
            Console.WriteLine(ex.Message);
            return 0;
        }
    }
}
// 利用方法將變化隔離開來
private DbConnection CreateDbConnection()
{
    if (_dbType == "Mysql")
    {
        return new MySqlConnection(_connectionString);
    }
    else if (_dbType == "Postgresql")
    {
        return new NpgsqlConnection(_connectionString);
    }
    else
    {
        return null;
    }
}
```
* 問題:
	* 這樣還是會因為要實現不同的數據庫而要修改 `SqlHelper` 類，這樣違反 OCP 原則。
#### Version 4
我們單獨建構一個類來建立數據庫連接。
* DbConnectionFactory.cs
```cs
public static class DbConnectionFactory
{
    private static readonly string _connectionString;
    private static readonly string _dbType;

    public static DbConnection CreateDbConnection()
    {
        if (_dbType == "Mysql")
        {
            return new MySqlConnection(_connectionString);
        }
        else if (_dbType == "Postgresql")
        {
            return new NpgsqlConnection(_connectionString);
        }
        else
        {
            return null;
        }
    }
}
```
* SqlHelper.cs
```cs
public class SqlHelper
{
    public int ExecuteNonQuery(string sql)
    {
        using (DbConnection conn = DbConnectionFactory.CreateDbConnection())
        {
            DbCommand cmd = conn.CreateCommand();
            cmd.CommandText = sql;

            try
            {
                conn.Open();
                return cmd.ExecuteNonQuery();
            }
            catch (Exception ex)
            {
                Console.WriteLine(ex.Message);
                return 0;
            }
        }
    }
}
```
* 大功告成，最後一版本就是簡單工廠的實現。

> 重點概念就是隔離變化。

## 簡單工廠 (Simple Factory)
又稱為靜態工廠，由上面所示我們為了調用方便將其使用 static 進行修飾，由工廠來決定該創建哪一個類。
#### 實作
* DbConnectionFactory
```cs
public static class DbConnectionFactory
{
		// 字段可以透過配置文件讀取
    private static readonly string _connectionString;
    private static readonly string _dbType;

    public static DbConnection CreateDbConnection()
    {
        if (_dbType == "Mysql")
        {
            return new MySqlConnection(_connectionString);
        }
        else if (_dbType == "Postgresql")
        {
            return new NpgsqlConnection(_connectionString);
        }
        else
        {
            return null;
        }
    }
}
```
* SqlHelper
```cs
public class SqlHelper
{
    public int ExecuteNonQuery(string sql)
    {
		    // 透過靜態工廠獲得對象
        using (DbConnection conn = DbConnectionFactory.CreateDbConnection())
        {
            DbCommand cmd = conn.CreateCommand();
            cmd.CommandText = sql;

            try
            {
                conn.Open();
                return cmd.ExecuteNonQuery();
            }
            catch (Exception ex)
            {
                Console.WriteLine(ex.Message);
                return 0;
            }
        }
    }
}


```
#### UML 類圖
![[Screenshot 2024-02-25 222901.png|800]]
* Client 依賴於抽象與工廠，由工廠建立實例，
#### 優缺點
* 優點:
	* 實現責任分離，並隔離變化。
	* 不修改客戶程式碼就能更換產品。
* 缺點:
	* 將所有實例創建放在同一個工廠類中，當需求變大工廠會膨脹難以維護，違反單一職責原則。
	* 擴展一定得修改工廠類，違反開閉原則。

## 工廠方法 (Factory Method)
工廠方法改進了簡單工廠集中將所有類型創建都放在工廠中，他將工廠本身進行抽象，**將對象創建延遲到其子類**。

> 目標是將需客製化的實例，也就是變化向外推到頂層 Program.cs 上。
* 將 DbConnectionFactory 進行抽象
```cs
public abstract class DbConnectionFactory
{
    public abstract DbConnection CreateDbConnection();
}
```

* 將 SqlHelper 依賴於抽象的工廠方法
```cs
public class SqlHelper
{
    private DbConnectionFactory _dbConnectionFactory; // 簡單工廠為靜態，但工廠方法需要實例

    public SqlHelper(DbConnectionFactory dbConnectionFactory)
    {
        _dbConnectionFactory = dbConnectionFactory;
    }
    public int ExecuteNoSql(string sql)
    {
        using (DbConnection conn = _dbConnectionFactory.CreateDbConnection())
        {
            DbCommand cmd = conn.CreateCommand();
            cmd.CommandText = sql;

            try
            {
                conn.Open();
                return cmd.ExecuteNonQuery();
            }
            catch (Exception ex)
            {
                Console.WriteLine(ex.Message);
            }
            return 0;
        }
    }
}
```

* 建立工廠方法實例類
```cs
public class PostgresDbConnectionFactory : DbConnectionFactory
{
    private static readonly string _connectionString;
    public override DbConnection CreateDbConnection()
    {
        return new NpgsqlConnection(_connectionString);
    }
}
```

* 使用方式 Program.cs
```cs
SqlHelper sqlHelper = new SqlHelper(new PostgresDbConnectionFactory());
int res = sqlHelper.ExecuteNoSql("select count(*) from employment");
Console.WriteLine(res);
```

#### UML 類圖
![[Screenshot 2024-02-26 104224.png | 800]]
#### 優缺點
* 優點: 符合設計原則
* 缺點: 類的數量會增加很快，因為每個工廠實例都要建立類。

## 抽象工廠 (Abstract Factory)
#### 概念引入
我們希望添加參數化查詢功能。
##### Version 1
* Program.cs
```cs
SqlHelper sqlHelper = new SqlHelper(new PostgresDbConnectionFactory());
int age = 18;
int res = sqlHelper.ExecuteNoSql($"select count(*) from employment where Age > {age};"); // SQL 注入風險
Console.WriteLine(res);
```

##### Version 2: 解決 SQL 注入問題
* SqlHelper 添加參數化查詢功能，並依賴於抽象 `DbParameter`
```cs
public int ExecuteNoSql(string sql, params DbParameter[] dbParameters)
{
		using (DbConnection conn = _dbConnectionFactory.CreateDbConnection())
		{
				DbCommand cmd = conn.CreateCommand();
				cmd.CommandText = sql;
				if (dbParameters is not null)
				{
						cmd.Parameters.AddRange(dbParameters);
				}

				try
				{
						conn.Open();
						return cmd.ExecuteNonQuery();
				}
				catch (Exception ex)
				{
						Console.WriteLine(ex.Message);
				}
				return 0;
		}
}
```
* 在 Program.cs 使用
```cs
SqlHelper sqlHelper = new SqlHelper(new PostgresDbConnectionFactory());
int age = 18;
int res = sqlHelper.ExecuteNoSql($"select count(*) from employment where Age > @Age;", new NpgsqlParameter("@Age", age));

Console.WriteLine(res);
```
###### 討論
我們好像並沒有違反設計模式，因為我們將建構拋置頂層，但注意不同 ExecuteNoSql 雖然參數使用抽象，但我需要傳入實例的 `NpgsqlParameter`，想像一下 ExecuteNoSql 並不一定在 Program.cs 中被調用，他可能散布於程式碼的各處，所以導致變化散步於代碼庫中，另外 `PostgresDbConnectionFactory` 需要強綁定 `NpgsqlParamter`，而不能使用 mysql 版本的，他們有一定的相依關係，我們應該實現高內聚。


##### Version 3: 解決依賴實例擴散問題
* 建立工廠方法來建立 `DbParameter` 實例，DbParameterFactory.cs
```cs
public abstract class DbParameterFactory
{
    public abstract DbParameter CreateDbParameter(string parameterName, object value);
}
```
* 建立實際工廠，PostgresDbParameterFactory.cs
```cs
public class PostgresDbParameterFactory : DbParameterFactory
{
    public override DbParameter CreateDbParameter(string parameterName, object value)
    {
        return new NpgsqlParameter(parameterName, value);
    }
}
```
* 聚合到 SqlHelper 中
```cs
public class SqlHelper
{
    private DbConnectionFactory _dbConnectionFactory;
    private DbParameterFactory _dbParameterFactory; // 聚合

    public SqlHelper(DbConnectionFactory dbConnectionFactory, DbParameterFactory dbParameterFactory)
    {
        _dbConnectionFactory = dbConnectionFactory;
        _dbParameterFactory = dbParameterFactory; // 聚合
    }
    public int ExecuteNoSql(string sql, params DbParameter[] dbParameters)
    {
        using (DbConnection conn = _dbConnectionFactory.CreateDbConnection())
        {
            DbCommand cmd = conn.CreateCommand();
            cmd.CommandText = sql;
            if (dbParameters is not null)
            {
                cmd.Parameters.AddRange(dbParameters);
            }

            try
            {
                conn.Open();
                return cmd.ExecuteNonQuery();
            }
            catch (Exception ex)
            {
                Console.WriteLine(ex.Message);
            }
            return 0;
        }
    }
		// 添加方法
    public DbParameter CreateDbParameter(string parameterName, object value)
    {
        return _dbParameterFactory.CreateDbParameter(parameterName, value);
    }
}
```

* Program.cs 中使用
```cs
DbConnectionFactory dbConnectionFactory = new PostgresDbConnectionFactory();
DbParameterFactory dbParameterFacotry = new PostgresDbParameterFactory();

SqlHelper sqlHelper = new SqlHelper(dbConnectionFactory, dbParameterFacotry);
string sql = $"select count(*) from employment where Age > @Age;";
int age = 18;
int res = sqlHelper.ExecuteNoSql(sql, sqlHelper.CreateDbParameter("@Age", age));

Console.WriteLine(res);
```

###### 問題
尚未解決 `PostgresDbConnectionFactory` 與 `PostgresDbParameterFactory` 之間的強依賴關係。

##### Version 4: 形成高內聚
* 將 DbConnectionFactory 與 DbParameterFactory 合成一個抽象工廠
```cs
public abstract class DbFactory
{
    public abstract DbConnection CreateDbConnection();
    public abstract DbParameter CreateDbParameter(string parameterName, object value);
}
```
* 實現抽象工廠
```cs
public class PostgreDbFactory : DbFactory
{
    private static readonly string _connectionString;
    public override DbConnection CreateDbConnection()
    {
        return new NpgsqlConnection(_connectionString);
    }

    public override DbParameter CreateDbParameter(string parameterName, object value)
    {
        return new NpgsqlParameter(parameterName, value);
    }
}
```
* 修改 SqlHelper
```cs
public class SqlHelper
{
    private DbFactory _dbFactory; // 只依賴於抽象工廠
    public SqlHelper(DbFactory dbFactory)
    {
        _dbFactory = dbFactory;
    }
    public int ExecuteNoSql(string sql, params DbParameter[] dbParameters)
    {
        using (DbConnection conn = _dbFactory.CreateDbConnection())
        {
            DbCommand cmd = conn.CreateCommand();
            cmd.CommandText = sql;
            if (dbParameters is not null)
            {
                cmd.Parameters.AddRange(dbParameters);
            }

            try
            {
                conn.Open();
                return cmd.ExecuteNonQuery();
            }
            catch (Exception ex)
            {
                Console.WriteLine(ex.Message);
            }
            return 0;
        }
    }

    public DbParameter CreateDbParameter(string parameterName, object value)
    {
        return _dbFactory.CreateDbParameter(parameterName, value);
    }
}
```
* 在 Prgram.cs 使用
```cs
DbFactory dbFactory = new PostgreDbFactory();
SqlHelper sqlHelper = new SqlHelper(dbFactory);

string sql = $"select count(*) from employment where Age > @Age;";
int age = 18;
int res = sqlHelper.ExecuteNoSql(sql, sqlHelper.CreateDbParameter("@Age", age));

Console.WriteLine(res);
```
以上就是抽象工廠。

#### UML 類圖
* 改造前
![[Screenshot 2024-02-26 115423.png|1200]]
* 改造後
![[Pasted image 20240226115530.png|1200]]
#### 結論與優缺點
> 抽象工廠為工廠方法的升級版，提供了**相互依賴**的對象提供統一的接口。

* 優點
	* 大幅度減少了工廠的數量
	* 高內聚，封裝性好
* 缺點
	* 發現新的依賴導致的擴展困難的困難，需要修改抽象基類與原有的實例，違背開閉原則

> 與工廠方法不同在於工廠方法只會有一個創建實例方法，而抽象工廠會有多個創建方法，且他們之間實例對象相互依賴。
> 使用上須先清楚依賴關係，確保抽象工廠的方法不會過多的添加新的功能，以保持穩定。


# 單例模式 (Singleton)
---
單例模式處理的問題是我們希望該類的實例在**當前進程中只被實例化一次**。
> 全局共享，全局唯一。

* 優點:
	* 共享唯一實體。
	* 節省系統資源。

### 實現一: 靜態類 
由於靜態類的性質他無法被創建，所以可以保證全局唯一。

```cs
public static class Singleton
{
	private static int _count = 0; 
	public static int Increase()
	{
			reutnr ++_count;
	}
}
```

* 問題
	* 靜態資源 always load，只要程式碼跑在 CLR 上靜態類無論是否使用都會被加載。(p.s. 除非真的導致 OOM 不然這點資源浪費，不必要理會)
	* 靜態類的特性其無法繼承抽象與接口化，導致我們無法對其進行擴展 (這才是關鍵問題)
> 這種方式基本上不稱為單例模式。


### 實現二: 私有構造器+靜態屬性+靜態方法
* Why?
	* 私有構造器避免外部 new 實例
	* 靜態屬性提供類加載時建立實例
	* 靜態方法需要提供外部獲得實例的方式，因為不能實例化只能透過靜態方法實現。
```cs
public class Singleton
{
		private static readonly Singleton _instance = new Singleton();
		// 私有構造器
		private Singleton() {};
		public static Singlton Instance { get {return _instance; } } // 這邊使用靜態 property
}
```

* 問題:
	* 雖然解決了可繼承與實現接口問題，但靜態資源還是 always loaded

### 實現三: 懶加載
* 在第一次調用時才進行加載
```cs
public class Singleton
{
		private static Singleton _instance; // 沒賦值
		// 私有構造器
		private Singleton() {};
		public static Singleton Instance
		{
				get 
				{
						return _instance ?? (_instance = new Singleton);
				}
		}
}
```

### 實現四: 多線程安全懶加載
```cs
public class Singleton
{
		// 細節: 鎖需要使用靜態修飾，因為不能讓多個實例有不同的鎖
		private static readonly object _locker = new object();
		private static Singleton _instance;
		// 私有構造器
		private Singleton() {};
		public static Singleton Instance
		{
				get 
				{
						if (_instance == null)
						{
								lock (_locker)
								{
										if (_instance == null)
										{
											_instance = new Singleton();
										}
								}
						}
						return _instance;
				}
		}
}
```

* 問題:
`_instance = new Singleton();` 這段順序應該為 1.開闢空間 2. 對象初始化 3. `_instance` 指向對象空間，但因為編譯器優化流程可能為 1. 開闢空間 2. `_instance` 指向對象空間 3. 對象初始化，若為後者在多線程情況下可能會為初始化的 `_instance` 給調用者。

### 實現四: 最安全版多線程安全懶加載
```cs
public class Singleton
{
		private static readonly object _locker = new object();
		// 添加 volatile 告訴編譯器，不要對其進行編譯器優化。
    private static volatile Singleton _instance;
		
		// 私有構造器
		private Singleton() {};

    public static Singleton Instance
    {
        get
        {
            if (_instance is null)
            {
                lock (_locker)
                {
                    if (_instance is null)
                    {
                        _instance = new Singleton();
                    }
                }
            }
            return _instance;
        }
    }
}
```
* 問題:
	* 代碼太多
### 實現五: C# 優雅版
參考: [[A11 - Performance#Lazy Initialize (延遲初始化)]]

```cs
public class Singleton
{
		private static readonly Lazy<Singleton> _instance = new Lazy<Singleton>(() => new Singleton());
		
		// 私有構造器
		private Singleton() {};
		
		public static Singleton Instance 
		{
        get { return _instance.Value; }
		}
}
```

# 建造者模式 (Builder)
---
Builder Pattern 是將一個複雜的對象建構拆分成一步一步的建造過程，使其可以靈活的建構不同的實例。與工廠模式不同建造者模式重點在於**配件組裝**。

> 抽象工廠也能做到一樣的事情，為甚麼要使用建造者模式?
> 因為**抽象工廠不能靈活搭配部件**，他會生成一個具體的產品，若想改造就得生成新的類，而**建造者模式可以客製化產品**。
##### 角色
* Builder: 為工人的統稱，抽象的概念
* ConcreteBuilder: 對應具體的工人。
* Director: 指導建構流程，有 Build 方法用來構建最終產品。
* Product: 最終建成的對象

### 標準建造者模式
#### UML 類圖
![[Pasted image 20240227115833.png | 800]]
> Builder 就是抽象工廠
> ConcreteBuilder 就是具體工廠，用來**組裝配件**
> Director 就是 Factory 使用 ConcreteBuilder 產出的配件**組裝成最後的產品**。
> 使用方式如下:

構造一個手機，包括 CPU, Memory, 螢幕等部分，要求手機配置可以靈活搭配。
```cs
PhonePartBuilder highPartBuilder = new HighPhonePartBuilder(); // Builder
PhoneFactory phoneFactory = new PhoneFactory(highPartBuilder); // Directory
Phone highPhone = phoneFactory.Build(); // Product
```

#### 優缺點
##### 優點
* 消除使用工廠會造成的類數量上升問題。
* 實現職責分離，Director 組合成實例，Builder 生成具體部件。
##### 缺點
* 配件搭配變得固定，因為它就是抽象工廠。

### 簡化建造者模式
* 我們並不需要有 Director 存在，因為它只做了 Build 的動作，我們可以將 Build 方法放在 Builer 上。

#### UML 類圖
![[Pasted image 20240229093512.png | 800]]

#### 實作
* PhoneBuilder.cs
```cs
public abstract class PhoneBuilder
{
    protected Phone Phone; // Product

    public PhoneBuilder()
    {
        Phone = new Phone();
    }
    // Part Builder
    protected abstract void BuildCpu();
    protected abstract void BuildMem();
    protected abstract void BuildScreen();

    // Director，這邊不允許子類修改，流程是固定的
    public Phone Build()
    {
        BuildCpu();
        BuildMem();
        BuildScreen();
        return Phone;
    }
}
```
* LowPhoneBuilder.cs
```cs
public class LowPhoneBuilder : PhoneBuilder
{
    protected override void BuildCpu()
    {
        Phone.Cpu = new LowCpu();
    }

    protected override void BuildMem()
    {
        Phone.Mem = new LowMem();
    }

    protected override void BuildScreen()
    {
        Phone.Screen = new LowScreen();
    }
}
```
* Program.cs
```cs
PhoneBuilder phoneBuilder = new LowPhoneBuilder();

var phone = phoneBuilder.Build();
phone.Show();
```

### 鏈式建造者模式
前兩版都無法將建造過程客製化，這版本會使用委託的方式，將客製化內容交由外部來設定，不藉由 Builder 子類來定義。

> 這種模式是在 c# 最優雅的實現方式，也是在框架中最常見到的方式。

#### 實作
* Program.cs
```cs
IPhoneBuilder builder = new PhoneBuilder();

builder.BuildCpu(cpuConfig => cpuConfig.Type = "4 cores")
       .BuildMem(memConfig => memConfig.Type = "4G")
       .BuidScreen(screenConfig => screenConfig.Type = "5'");

Phone phone = builder.Build();

phone.Show();
```

* IPhoneBuider.cs
```cs
public interface IPhoneBuilder
{
    IPhoneBuilder BuildCpu(Action<Cpu> cpuConfig);
    IPhoneBuilder BuildMem(Action<Mem> memConfig);
    IPhoneBuilder BuidScreen(Action<Screen> screenConfig);
    Phone Build();
}
```

* PhoneBuider.cs
```cs
public class PhoneBuilder : IPhoneBuilder
{
    private Phone? _phone; // product
    // parts
    private Cpu? _cpu; 
    private Mem? _mem;
    private Screen? _screen;

		// 使用委託將客製化內容移出
    public IPhoneBuilder BuildCpu(Action<Cpu> cpuConfig)
    {
        _cpu = new Cpu();
        cpuConfig.Invoke(_cpu);
        return this;
    }

    public IPhoneBuilder BuildMem(Action<Mem> memConfig)
    {
        _mem = new Mem();
        memConfig.Invoke(_mem);
        return this;
    }
    public IPhoneBuilder BuidScreen(Action<Screen> screenConfig)
    {
        _screen = new Screen();
        screenConfig(_screen);
        return this;
    }

		// Director
    public Phone Build()
    {
        _phone = new Phone { Cpu = _cpu, Mem = _mem, Screen = _screen };
        return _phone;
    }
}
```

> StringBuilder 是更簡化的建造者模式，簡潔在於它沒有實現接口，因為它非常穩定，不用我們的例子可能會想要實現不同的 Builder 實例。


# 原型模式 (Prototype)
---
先創建一個對象，通過 Clone 來創建新對象。用於要**創建大量對象**，**且保證性能** (也就是不使用 Constructor 建構)，創建出的對象，只需要作些許微調的情況。

### 案例
找工作時需要準備多個簡歷，但絕大部分信息相同，可以根據不同面試公司稍做微調。
#### 使用建造者模式實現
```cs
// Buider
public interface IResumeBuilder
{
    IResumeBuilder BuildBaseInfo(Action<BaseInfo> baseInfoConfig);
    IResumeBuilder BuildWorkExperence(Action<WorkExperence> workExperenceConfig);
    ResumeBase Build();
}
// Concrete Builder
public class ITResumeBuider : IResumeBuilder
{
    private readonly BaseInfo _baseInfo = new BaseInfo();
    private readonly WorkExperence _workExperence = new WorkExperence();
    public IResumeBuilder BuildBaseInfo(Action<BaseInfo> baseInfoConfig)
    {
        baseInfoConfig.Invoke(_baseInfo);
        return this;
    }

    public IResumeBuilder BuildWorkExperence(Action<WorkExperence> workExperenceConfig)
    {
        workExperenceConfig.Invoke(_workExperence);
        return this;
    }
    public ResumeBase Build()
    {
        ITResume resume = new ITResume
        {
            Name = _baseInfo.Name,
            Age = _baseInfo.Age,
            Gender = _baseInfo.Gender,
            ExpectedSalary = _baseInfo.ExpectedSalary,
            WorkExperence = new WorkExperence
            {
                Company = _workExperence.Company,
                Detail = _workExperence.Detail,
                StartDate = _workExperence.StartDate,
                EndDate = _workExperence.EndDate
            }
        };
        return resume;
    }
}
```

* Program.cs
```cs
ITResumeBuider resumeBuider = new ITResumeBuider();

resumeBuider.BuildBaseInfo(cfg =>
{
    cfg.Name = "Liam";
    cfg.Gender = "Male";
    cfg.Age = 26;
    cfg.ExpectedSalary = "100W";
}).BuildWorkExperence(cfg =>
{
    cfg.Company = "pxpay";
    cfg.Detail = "Backend";
    cfg.StartDate = DateTime.Parse("2023-3-1");
    cfg.EndDate = DateTime.Parse("2025-4-7");
});

ResumeBase resume = resumeBuider.Build();

ResumeBase resume1 = resumeBuider.BuildBaseInfo(cfg =>
{
    cfg.ExpectedSalary = "98W";
}).Build();

ResumeBase resume2 = resumeBuider.BuildWorkExperence(cfg =>
{
    cfg.Company = "Google";
}).Build();


resume.Display();
resume1.Display();
resume2.Display();
```
* 問題:
	* 所有資訊都在 `resume` 裡面，我們並不用每次都使用 Build，由他本身自己克隆就好。

#### 使用原型模式
```cs
public abstract class ResumeBase
{
    public string Name { get; set; }
    public string Gender { get; set; }
    public int Age { get; set; }
    public string ExpectedSalary { get; set; }
    public abstract void Display();

    public abstract ResumeBase Clone(); // 添加 Clone 方法
}

public class ITResume : ResumeBase
{
    public WorkExperence WorkExperence { get; set; }

    public override void Display()
    {
        Console.WriteLine($"Name: {Name}");
        Console.WriteLine($"Gender: {Gender}");
        Console.WriteLine($"Age: {Age}");
        Console.WriteLine($"ExpectedSalary: {ExpectedSalary}");
        Console.WriteLine("----------------------------");
        WorkExperence?.Display();
        Console.WriteLine("----------------------------");
    }
		// 實作 Clone 
    public override ResumeBase Clone()
    {
        ITResume resume = new ITResume
        {
            Name = this.Name,
            Age = this.Age,
            Gender = this.Gender,
            ExpectedSalary = this.ExpectedSalary,
            WorkExperence = new WorkExperence // 注意這邊不能直接賦值
            {
                Company = this.WorkExperence.Company,
                Detail = this.WorkExperence.Detail,
                StartDate = this.WorkExperence.StartDate,
                EndDate = this.WorkExperence.EndDate
            }
        };
        return resume;
    }
}
```
* 問題:
	* Clone 方法違反開閉原則，很容易需要修改。
	* 並沒有優化創建過程，性能相同

#### 原型模式優化
* 使用 `MemberwiseClone` 來使其不用重新構建，而是開闢一個新的區域將原本複製過來，並不會再進入一次構造函數。

```cs
// sealed 不允許被繼承
public sealed class ITResume : ResumeBase
{
    public WorkExperence WorkExperence { get; set; }

    public override void Display()
    {
        Console.WriteLine($"Name: {Name}");
        Console.WriteLine($"Gender: {Gender}");
        Console.WriteLine($"Age: {Age}");
        Console.WriteLine($"ExpectedSalary: {ExpectedSalary}");
        Console.WriteLine("----------------------------");
        if (WorkExperence is not null)
        {
            WorkExperence.Display();
        }
        Console.WriteLine("----------------------------");
    }

    public override ResumeBase Clone()
    {
        var resume = this.MemberwiseClone() as ITResume;
        resume.WorkExperence = this.WorkExperence.Clone();
        return resume;
    }
}

public class WorkExperence
{
    public string Company { get; set; }
    public string Detail { get; set; }
    public DateTime StartDate { get; set; }
    public DateTime EndDate { get; set; }
    public void Display()
    {
        Console.WriteLine("Working Experence:");
        Console.WriteLine($"{Company}\t{StartDate.ToShortDateString()}\t{EndDate.ToShortDateString()}");
        Console.WriteLine("Detail:");
        Console.WriteLine(Detail);
    }
    // 其依賴的引用類型也要使用 Clone 方法。
    public WorkExperence Clone()
    {
        return this.MemberwiseClone() as WorkExperence;
    }
}

```
* 注意點:
	* `WorkExperence` 需要實現 `Clone` 方法，不然會導致淺拷貝問題，也就是所有的應用對象都需要實現 `Clone` 方法
	* 原型模式的類建議要使用 `sealed` 關鍵字，因為子類繼承若添加新的引用對象，又沒有重寫 `Clone`，會導致淺拷貝問題，故乾脆不要使其被繼承。
* 問題:
	* 違反開閉原則，因為只要添加新的依賴引用對象，就要修改 `Clone` 方法

#### 使用序列化與反序列化來優化原型模式
* 有 xml, json, binary(已禁用) 等方式
```cs
public override ResumeBase Clone()
{
		using var stream = new MemoryStream();
		JsonSerializer.Serialize(stream, this);
		stream.Position = 0;
		return JsonSerializer.Deserialize<ITResume>(stream);
}
```
* 雖然效率會比 `MemberwiseClone()`差，但是可以避免淺拷貝與可以遵守 OCP。
##### 為甚麼不使用 IClonable 接口?
> 來自於 Effective C# 原則 27
* 優點:
	* 不需要建立抽象基類
* 缺點:
	* 返回類型為 object，調用者完全不知道是甚麼類型。
	* .Net 自從支持泛型後，幾乎所有接口都有泛型參數，但 IClonable 卻沒有，可能被 .Net 遺棄了
	* IClonable 作為方法接收參數幾乎沒有任何意義。
	* 多數類庫實現 IClonable 內部都直接 private 然後拋出異常。

### UML 類圖
![[Pasted image 20240301163509.png|600]]
* 在抽象基類要聲明 Clone 方法，由子類實現。
## 總結
* 工廠模式: 代替 new 來實例化對象，降地 new 造成的耦合關係。
	* 簡單工廠: 靜態工廠，使用靜態方法實現，將所有實例集中創建。
	* 工廠方法: 將實例延遲到工廠子類完成，由子類來決定實例化哪個類。
	* 抽象工廠: 將有相依性的對象統一在一起創建，提供統一的接口。
* 單例模式: 只允許對象被創建一次(固定個數也行)，現在更流行的方式會使用 IOC 容器實現。
* 建造者模式: 創建複雜的、具有複合屬性的對象，標準的建造者模式就是抽象工廠(Builder)+簡單工廠(Director)的結構，並約束建造流程。
* 原型模式: 用於重複創建包含大量公共屬性，或者需要消耗大量資源的對象時，讓他採用自我複製實現新對象。