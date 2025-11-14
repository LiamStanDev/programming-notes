# 簡介
---
### RAPIDS
RAPIDS 是一系列開放原始碼軟體函式庫和 API，可在 GPU 上執行完整的資料科學流程。
* RAPIDS 實際運用 NVIDIA CUDA
* 可以用於工作站與叢集中 e.g. Spark, Kubernetes
* 提供 GPU 的平行計算功能與高速的記憶體存取透過 GPU DataFrame。

### Spark with RAPIDS
Apache Spark 3.x 提供了 API 來請求利用 GPU 資源，並在內部增加了 GPU 調度，允許叢集管理器 (YARN, K8S) 整合，使用戶能更容易使用 GPU。

![[Pasted image 20240701095917.png]]

#### Apache Arrow
RAPIDS 提供 GPU DataFrame，它是基於 Apache Arrow (language-independent, **columnar memory format**)，如下，
![[Pasted image 20240701101206.png]]
增加 data locality，也就是將相關的數據放置在 momory 中相近的位置，以利用 CPU/GPU 的 Cache (虛擬地址轉換可以透過 MMU 中的快表)，來進行加速，減少 Cahce miss 的機會。

### DataFrame 
* 尚未使用 RAPIDS 前： DataFrame 會被轉成 RDD 的 Row 格式，會一行一行進行處理
* Spark 2.x 改進：能透過 Vectorized Parquet 與 ORC 資料格式進行批量的 columnar 處理
* RAPIDS: 透過使用 GPU 來批量處理 columnar data，使得 GPU 可以更有效的進行數據處理
> GPU 處理 columnar data　會顯著快於 row-by-row 處理

另外 Spark Shuffle 也可以透過 OpenUCX 庫來進行高效率的數據傳遞，Spark shuffle 會發生在數據在節點時進行移動的時候，透過 OpenUCX 可以使 GPU 的數據
* 保持儘可能多的數據在GPU上
* 尋找數據在節點間移動的最快路徑
* 繞過 CPU 進行 GPU 到 GPU 的記憶體傳輸
![[Pasted image 20240701102906.png]]

# 使用
---
Spark 3.x 可以將執行 spark SQL 與 Spark DataFrame 操作改為使用 GPU 進行，但我們不需要修改程式碼，在背後會自行判斷是否該由 GPU 進行，若無法使用 GPU 會回退到使用 CPU 進行計算。
* 注意：RDD 操作是無法使用 GPU 加速的

#### Requirment
* Spark 3.1+
* 配置 Spark cluster with GPU
* RAPIDS 加速插件 .jar 包
* 設定 `spark.plugins` 為 `com.nvidia.spark.SQLPlugin`


#### 範例\
ref: https://docs.nvidia.com/spark-rapids/user-guide/23.10/tuning-guide.html
```python
from pyspark.sql import SparkSession

# 建立 SparkSession，並啟用 RAPIDS 加速
spark = SparkSession.builder \
    .appName("Spark with RAPIDS Example") \
    .config("spark.rapids.sql.enabled", "true") \ # 啟用 Rapid 來處理 SQL 
    .config("spark.executor.memory", "4G") \ # 每個計算節點分配 4G Ram 來處理該任務
    .config("spark.executor.cores", "4") \ # 每個計算節點分配 4 核心
    .config("spark.task.resource.gpu.amount", "0.125") \ # 表示使用 計算節點上的 1/8 個 gpu，每個節點至多只有一個 GPU
    .getOrCreate()

# 讀取 CSV 文件
df = spark.read.csv("/path/to/your/file.csv", header=True, inferSchema=True)

# 執行一些簡單的數據處理操作
df = df.select("column1", "column2") \
       .filter(df["column1"] > 100) \
       .groupBy("column2") \
       .count()

# 將結果寫入一個新的 CSV 文件
df.write.csv("/path/to/output/file.csv", header=True)

# 結束 SparkSession
spark.stop()
```

* `spark.task.resource.gpu.amout` 佔用 GPU 的比例，如：
```python
# 每個任務要完整的一個 GPU，表示每個 executor (計算節點) 只能同時運行一個任務
spark.task.resource.gpu.amount=1
# 希望一個 GPU 可以處理多個任務，如下為同時處理 8 個任務
spark.task.resource.gpu.amount=0.125
```
# Reference
---
[RAPIDS Accelerator for Apache Spark - User Guide](https://docs.nvidia.com/spark-rapids/user-guide/latest/index.html)
[NVIDIA/spark-rapids github repo](https://github.com/NVIDIA/spark-rapids)
