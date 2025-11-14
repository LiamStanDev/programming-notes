# 回答框架
---
## 框架說明
### Step 1: Clarification and Edge Cases
這邊將題目複述，並邊講邊想問題，比如說 input/output 的形式，以及某些過程中的重要計算公式

| 步驟                 | 句型                                                                                                                                 |
| ------------------ | ---------------------------------------------------------------------------------------------------------------------------------- |
| 重複問題               | So, to make sure I understand the problem correctly, we need to...                                                                 |
| 確認 input/output 格式 | What are the constraints on the input size, N?<br>Can the input array contain negative numbers?                                    |
| 詢問 Edge Cases      | What should I return if the input is null or an empty list?<br>Are the elements guaranteed to be unique or are duplicates allowed? |


### Step 2: High-Level Approach (Brute-force)
提出一個不高效的解決方案，作爲討論起點

| 步驟      | 句型                                                                            |
| ------- | ----------------------------------------------------------------------------- |
| 提出最初想法  | My initial thouths are to use brute-force approach.                           |
| 描述邏輯    | This would involve iterating through all possible subsets/pairs/combinations. |
| 初步複雜度分析 | With this approch, the time complexity would be O(N^2).                       |
| 指出問題    | This will likely result in a Time Limit Exceeded error for large inputs.      |


### Step 3: Core Idea (Optimized Solution)

| 步驟                                                                                                                                                                                                                                                                               | 句型                                                                                                          |
| -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| 過度到優化解                                                                                                                                                                                                                                                                           | To optimaze the time complexity, we can leverage a data structure like a Hash Map / a Two Pointers techniqu |
| 具體步驟  We will maintain two pointers, left and right, both initialized to be head.<br><br>We will advance the right pointer to expand the window until the condition is violated.<br><br>Then, we will retreat the left pointer to shrink the window and restore the invariant to |                                                                                                             |

### Step 4: Complexity Analysis

| 步驟          | 句型                                                                                                      |
| ----------- | ------------------------------------------------------------------------------------------------------- |
| Final time  | With this approach, the time complexity is reduce to O(N) because each element is visited at most once. |
| Final space | The space complexity is O(1) since we only use a few extra variables for the pointers.                  |

### Step 5: Implementation

| 步驟     |                                                                                                                                                             |
| ------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 開始撰寫提醒 | I will now proceed with the implementation                                                                                                                  |
| 編寫邊闡述  | I initialize three vairables ...<br><br>Here, I'm setting the base case for the recursion<br><br>This line is for hanlding the edge case of an empty input. |


# 常用英文
### Two Poiner 類型
| **動作句片語 (Verb Phrase)**                          | **中文翻譯 (Chinese Translation)** |
| ------------------------------------------------ | ------------------------------ |
| 1. Initialize `left` and `right` pointers.       | 初始化 `left` 和 `right` 指針。       |
| 2. Move the pointers inwards/towards each other. | 將指針相向/向內移動。                    |
| 3. Advance the left pointer.                     | 推進左指針（向右移動）。                   |
| 4. Decrement the right pointer.                  | 遞減右指針（向左移動）。                   |
| 5. Compare the values pointed to by $L$ and $R$. | 比較 $L$ 和 $R$ 指針所指的值。           |
| 6. Swap the elements at the pointer positions.   | 交換指針位置的元素。                     |
| 7. Set $L$ to $R$'s current position.            | 將 $L$ 設定為 $R$ 當前的位置。           |
| 8. Iterate until the pointers cross.             | 循環直到指針交叉。                      |
| 9. Skip over any duplicate elements.             | 跳過任何重複的元素。                     |
| 10. Find the shortest distance between them.     | 找到它們之間的最短距離。                   |
cccccccccc