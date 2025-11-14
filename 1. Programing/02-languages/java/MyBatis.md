# 簡介
---
MyBatis原本是Apache的iBatis後來前移到Google Code更名為MyBatis，是一個基於Java的持久層框架，包含SQL Maps與Data Access Object(DAO)。
### MyBatis的特性
1. 支持可訂製化SQL查詢
2. 避免幾乎所有的JDBC代碼
3. 透過XML與註解來配置映射
4. 為一個半自動的ORM: JDBC為全手動，Hibernet不用自己寫SQL所以為全自動。

### MyBatis vs. Hibernate
* Hibernate: 
	* 操作簡單，開發效率高
	* 全自動: 不需要寫SQL
	* 大量使用反射，執行效率低
* MyBatis: 
	* 輕量級，高性能
	* SQL與Java分開: SQL寫在xml
	* 開發效率稍弱於Hibernate
