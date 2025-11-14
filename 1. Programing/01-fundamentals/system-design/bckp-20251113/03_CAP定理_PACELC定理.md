# CAP 定理與 PACELC 定理

## 1. CAP 定理理論解釋與圖解

CAP 定理（Brewer's theorem）指出，分散式系統在「一致性（Consistency）」、「可用性（Availability）」與「分區容忍性（Partition Tolerance）」三者之間，最多只能同時滿足兩項，無法三者兼得。

- **一致性（Consistency, C）**：所有節點在同一時間看到的資料是一致的。
- **可用性（Availability, A）**：每個請求都能在有限時間內獲得回應（不保證是最新資料）。
- **分區容忍性（Partition Tolerance, P）**：系統能容忍網路分區（部分節點無法互通）而繼續運作。
- 
> 系統在任何網路異常下都會停掉，但現實中分散式系統「網路分區一定會發生」，所以 幾乎所有分散式系統都必須要有 P。
### 圖解（Venn Diagram）

```mermaid
venn
    title CAP 定理
    A: "一致性 (C)"
    B: "可用性 (A)"
    C: "分區容忍性 (P)"
    AB: "CA"
    AC: "CP"
    BC: "AP"
```

- CA：如傳統關聯式資料庫（單機）
- CP：如 HBase
- AP：如 Cassandra、DynamoDB

---

## 2. PACELC 定理理論解釋與圖解

PACELC 定理是 CAP 定理的延伸，補足了 CAP 在「無分區」情境下的權衡。其核心為：

- **若發生分區（Partition, P）**，系統必須在「一致性（Consistency, C）」與「可用性（Availability, A）」間取捨（即 CAP）。
- **否則（Else, E）**，系統需在「延遲（Latency, L）」與「一致性（Consistency, C）」間取捨。

### PACELC 決策樹圖解

```mermaid
graph TD
    A[分區發生？] -->|是| B[一致性 (C) 或 可用性 (A)]
    A -->|否| C[延遲 (L) 或 一致性 (C)]
```

- 例：DynamoDB 屬於 PA/EL（分區時偏可用，平時偏低延遲）

---

## 3. 真實世界分散式系統範例

| 系統        | CAP 選擇 | PACELC 分類 | 說明                 |
| --------- | ------ | --------- | ------------------ |
| Cassandra | AP     | PA/EL     | 偏重可用性與分區容忍，平時追求低延遲 |
| MongoDB   | CP/AP  | PA/EC     | 可配置，預設偏可用性，亦可調整一致性 |
| DynamoDB  | AP     | PA/EL     | 強調高可用與低延遲，犧牲部分一致性  |
| HBase     | CP     | PC/EC     | 偏重一致性與分區容忍，犧牲可用性   |
| Bigtable  | CP     | PC/EC     | 與 HBase 類似，強調一致性   |

---

## 4. 架構師實務建議與 trade-off 分析

- **需求導向選擇**：根據業務需求決定偏重一致性、可用性或延遲。例如金融交易系統需強一致性，社群貼文可接受最終一致性。
- **一致性 vs 可用性**：強一致性會犧牲部分可用性與效能，適合資料正確性要求高的場景。高可用性則適合對即時性要求高但可容忍資料短暫不一致的應用。
- **分區容忍性不可妥協**：現代分散式系統必須具備分區容忍性，否則網路異常時系統將無法運作。
- **延遲考量**：低延遲提升用戶體驗，但可能需犧牲一致性。可根據需求調整一致性等級（如 Cassandra 的 tunable consistency）。
- **混合策略**：可針對不同資料或服務採用不同策略，例如關鍵資料採強一致性，非關鍵資料採最終一致性。

### Trade-off 分析建議

- **評估業務容忍度**：明確界定哪些資料可容忍不一致，哪些必須即時同步。
- **設計可調參數**：選擇支援一致性、可用性、延遲可調整的系統，根據實際情境動態調整。
- **監控與測試**：持續監控系統一致性、可用性與延遲指標，並進行壓力測試，確保系統能應對各種異常情境。

---

> 參考資料：
> - Eric Brewer, "CAP Twelve Years Later: How the 'Rules' Have Changed"
> - Daniel Abadi, "Consistency Tradeoffs in Modern Distributed Database System Design: CAP is Only Part of the Story"
> - 各分散式資料庫官方文件