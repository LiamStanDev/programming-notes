# 用 Go 刷 LeetCode - 資料結構完全指南

> 涵蓋 LeetCode 必備的所有資料結構實現與典型題目解析

## 目錄

- [1. Stack（堆疊）](#1-stack堆疊)
- [2. Queue（佇列）](#2-queue佇列)
- [3. Deque（雙端佇列）](#3-deque雙端佇列)
- [4. LinkedList（鏈表）](#4-linkedlist鏈表)
- [5. Binary Tree（二叉樹）](#5-binary-tree二叉樹)
- [6. Binary Search Tree（BST）](#6-binary-search-treebst)
- [7. Heap（堆）](#7-heap堆)
- [8. Trie（字典樹）](#8-trie字典樹)
- [9. Graph（圖）](#9-graph圖)
- [10. Union-Find（並查集）](#10-union-find並查集)
- [11. LRU Cache（綜合應用）](#11-lru-cache綜合應用)
- [12. 資料結構選擇指南](#12-資料結構選擇指南)

---

## 1. Stack（堆疊）

### 1.1 基於 Slice 實現

```go
package main

import "fmt"

type Stack struct {
    items []interface{}
}

// Push - 入棧 O(1)
func (s *Stack) Push(item interface{}) {
    s.items = append(s.items, item)
}

// Pop - 出棧 O(1)
func (s *Stack) Pop() interface{} {
    if s.IsEmpty() {
        return nil
    }
    index := len(s.items) - 1
    item := s.items[index]
    s.items = s.items[:index]
    return item
}

// Peek - 查看棧頂 O(1)
func (s *Stack) Peek() interface{} {
    if s.IsEmpty() {
        return nil
    }
    return s.items[len(s.items)-1]
}

// IsEmpty - 判斷是否為空 O(1)
func (s *Stack) IsEmpty() bool {
    return len(s.items) == 0
}

// Size - 獲取大小 O(1)
func (s *Stack) Size() int {
    return len(s.items)
}
```

### 1.2 泛型版本（Go 1.18+）

```go
type Stack[T any] struct {
    items []T
}

func (s *Stack[T]) Push(item T) {
    s.items = append(s.items, item)
}

func (s *Stack[T]) Pop() (T, bool) {
    if len(s.items) == 0 {
        var zero T
        return zero, false
    }
    index := len(s.items) - 1
    item := s.items[index]
    s.items = s.items[:index]
    return item, true
}

func (s *Stack[T]) Peek() (T, bool) {
    if len(s.items) == 0 {
        var zero T
        return zero, false
    }
    return s.items[len(s.items)-1], true
}

func (s *Stack[T]) IsEmpty() bool {
    return len(s.items) == 0
}

func (s *Stack[T]) Size() int {
    return len(s.items)
}
```

### 1.3 LeetCode 經典題目

#### 題目 1：有效的括號（LeetCode 20）

```go
// 給定一個只包括 '(',')','{','}','[',']' 的字符串 s，判斷字符串是否有效
func isValid(s string) bool {
    if len(s)%2 != 0 {
        return false
    }
    
    pairs := map[rune]rune{
        ')': '(',
        '}': '{',
        ']': '[',
    }
    
    stack := &Stack[rune]{}
    
    for _, ch := range s {
        // 如果是右括號
        if open, ok := pairs[ch]; ok {
            top, hasTop := stack.Pop()
            if !hasTop || top != open {
                return false
            }
        } else {
            // 左括號入棧
            stack.Push(ch)
        }
    }
    
    return stack.IsEmpty()
}
```

#### 題目 2：每日溫度（LeetCode 739 - 單調棧）

```go
// 給定每日溫度列表，返回需要等待多少天才能等到更高的溫度
func dailyTemperatures(temperatures []int) []int {
    n := len(temperatures)
    result := make([]int, n)
    stack := &Stack[int]{} // 存儲索引
    
    for i := 0; i < n; i++ {
        // 當前溫度比棧頂索引的溫度高
        for !stack.IsEmpty() {
            top, _ := stack.Peek()
            if temperatures[i] <= temperatures[top] {
                break
            }
            idx, _ := stack.Pop()
            result[idx] = i - idx
        }
        stack.Push(i)
    }
    
    return result
}
```

#### 題目 3：最小棧（LeetCode 155）

```go
type MinStack struct {
    stack    []int
    minStack []int // 維護最小值
}

func Constructor() MinStack {
    return MinStack{
        stack:    []int{},
        minStack: []int{},
    }
}

func (s *MinStack) Push(val int) {
    s.stack = append(s.stack, val)
    if len(s.minStack) == 0 || val <= s.minStack[len(s.minStack)-1] {
        s.minStack = append(s.minStack, val)
    }
}

func (s *MinStack) Pop() {
    if len(s.stack) == 0 {
        return
    }
    top := s.stack[len(s.stack)-1]
    s.stack = s.stack[:len(s.stack)-1]
    
    if top == s.minStack[len(s.minStack)-1] {
        s.minStack = s.minStack[:len(s.minStack)-1]
    }
}

func (s *MinStack) Top() int {
    return s.stack[len(s.stack)-1]
}

func (s *MinStack) GetMin() int {
    return s.minStack[len(s.minStack)-1]
}
```

---

## 2. Queue（佇列）

### 2.1 基於 Slice 實現

```go
type Queue[T any] struct {
    items []T
}

// Enqueue - 入隊 O(1) 均攤
func (q *Queue[T]) Enqueue(item T) {
    q.items = append(q.items, item)
}

// Dequeue - 出隊 O(n)（需要移動元素）
func (q *Queue[T]) Dequeue() (T, bool) {
    if len(q.items) == 0 {
        var zero T
        return zero, false
    }
    item := q.items[0]
    q.items = q.items[1:]
    return item, true
}

// Front - 查看隊首 O(1)
func (q *Queue[T]) Front() (T, bool) {
    if len(q.items) == 0 {
        var zero T
        return zero, false
    }
    return q.items[0], true
}

func (q *Queue[T]) IsEmpty() bool {
    return len(q.items) == 0
}

func (q *Queue[T]) Size() int {
    return len(q.items)
}
```

### 2.2 環形佇列（高效版本）

```go
type CircularQueue[T any] struct {
    items []T
    head  int
    tail  int
    size  int
    cap   int
}

func NewCircularQueue[T any](capacity int) *CircularQueue[T] {
    return &CircularQueue[T]{
        items: make([]T, capacity),
        cap:   capacity,
    }
}

// Enqueue - O(1)
func (q *CircularQueue[T]) Enqueue(item T) bool {
    if q.IsFull() {
        return false
    }
    q.items[q.tail] = item
    q.tail = (q.tail + 1) % q.cap
    q.size++
    return true
}

// Dequeue - O(1)
func (q *CircularQueue[T]) Dequeue() (T, bool) {
    if q.IsEmpty() {
        var zero T
        return zero, false
    }
    item := q.items[q.head]
    q.head = (q.head + 1) % q.cap
    q.size--
    return item, true
}

func (q *CircularQueue[T]) Front() (T, bool) {
    if q.IsEmpty() {
        var zero T
        return zero, false
    }
    return q.items[q.head], true
}

func (q *CircularQueue[T]) IsEmpty() bool {
    return q.size == 0
}

func (q *CircularQueue[T]) IsFull() bool {
    return q.size == q.cap
}
```

### 2.3 LeetCode 經典題目

#### 題目 1：用棧實現隊列（LeetCode 232）

```go
type MyQueue struct {
    inStack  []int
    outStack []int
}

func Constructor() MyQueue {
    return MyQueue{}
}

func (q *MyQueue) Push(x int) {
    q.inStack = append(q.inStack, x)
}

func (q *MyQueue) Pop() int {
    q.transfer()
    val := q.outStack[len(q.outStack)-1]
    q.outStack = q.outStack[:len(q.outStack)-1]
    return val
}

func (q *MyQueue) Peek() int {
    q.transfer()
    return q.outStack[len(q.outStack)-1]
}

func (q *MyQueue) Empty() bool {
    return len(q.inStack) == 0 && len(q.outStack) == 0
}

func (q *MyQueue) transfer() {
    if len(q.outStack) == 0 {
        for len(q.inStack) > 0 {
            val := q.inStack[len(q.inStack)-1]
            q.inStack = q.inStack[:len(q.inStack)-1]
            q.outStack = append(q.outStack, val)
        }
    }
}
```

#### 題目 2：二叉樹層序遍歷（LeetCode 102）

```go
type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

func levelOrder(root *TreeNode) [][]int {
    if root == nil {
        return [][]int{}
    }
    
    result := [][]int{}
    queue := []*TreeNode{root}
    
    for len(queue) > 0 {
        levelSize := len(queue)
        level := []int{}
        
        for i := 0; i < levelSize; i++ {
            node := queue[0]
            queue = queue[1:]
            level = append(level, node.Val)
            
            if node.Left != nil {
                queue = append(queue, node.Left)
            }
            if node.Right != nil {
                queue = append(queue, node.Right)
            }
        }
        
        result = append(result, level)
    }
    
    return result
}
```

---

## 3. Deque（雙端佇列）

### 3.1 基於 Slice 實現

```go
type Deque[T any] struct {
    items []T
}

func (d *Deque[T]) PushFront(item T) {
    d.items = append([]T{item}, d.items...)
}

func (d *Deque[T]) PushBack(item T) {
    d.items = append(d.items, item)
}

func (d *Deque[T]) PopFront() (T, bool) {
    if len(d.items) == 0 {
        var zero T
        return zero, false
    }
    item := d.items[0]
    d.items = d.items[1:]
    return item, true
}

func (d *Deque[T]) PopBack() (T, bool) {
    if len(d.items) == 0 {
        var zero T
        return zero, false
    }
    item := d.items[len(d.items)-1]
    d.items = d.items[:len(d.items)-1]
    return item, true
}

func (d *Deque[T]) Front() (T, bool) {
    if len(d.items) == 0 {
        var zero T
        return zero, false
    }
    return d.items[0], true
}

func (d *Deque[T]) Back() (T, bool) {
    if len(d.items) == 0 {
        var zero T
        return zero, false
    }
    return d.items[len(d.items)-1], true
}

func (d *Deque[T]) IsEmpty() bool {
    return len(d.items) == 0
}

func (d *Deque[T]) Size() int {
    return len(d.items)
}
```

### 3.2 LeetCode 經典題目

#### 題目：滑動窗口最大值（LeetCode 239 - 單調隊列）

```go
// 給定數組和滑動窗口大小 k，返回每個窗口的最大值
func maxSlidingWindow(nums []int, k int) []int {
    if len(nums) == 0 || k == 0 {
        return []int{}
    }
    
    result := []int{}
    deque := &Deque[int]{} // 存儲索引，保持遞減順序
    
    for i := 0; i < len(nums); i++ {
        // 移除不在窗口內的元素
        front, hasFront := deque.Front()
        if hasFront && front < i-k+1 {
            deque.PopFront()
        }
        
        // 維護遞減隊列
        for !deque.IsEmpty() {
            back, _ := deque.Back()
            if nums[back] < nums[i] {
                deque.PopBack()
            } else {
                break
            }
        }
        
        deque.PushBack(i)
        
        // 窗口形成後，記錄最大值
        if i >= k-1 {
            front, _ := deque.Front()
            result = append(result, nums[front])
        }
    }
    
    return result
}
```

---

## 4. LinkedList（鏈表）

### 4.1 單向鏈表

```go
type ListNode struct {
    Val  int
    Next *ListNode
}

type LinkedList struct {
    Head *ListNode
    Size int
}

// InsertAtHead - 頭部插入 O(1)
func (ll *LinkedList) InsertAtHead(val int) {
    newNode := &ListNode{Val: val, Next: ll.Head}
    ll.Head = newNode
    ll.Size++
}

// InsertAtTail - 尾部插入 O(n)
func (ll *LinkedList) InsertAtTail(val int) {
    newNode := &ListNode{Val: val}
    if ll.Head == nil {
        ll.Head = newNode
        ll.Size++
        return
    }
    
    current := ll.Head
    for current.Next != nil {
        current = current.Next
    }
    current.Next = newNode
    ll.Size++
}

// InsertAtIndex - 指定位置插入 O(n)
func (ll *LinkedList) InsertAtIndex(index, val int) bool {
    if index < 0 || index > ll.Size {
        return false
    }
    
    if index == 0 {
        ll.InsertAtHead(val)
        return true
    }
    
    newNode := &ListNode{Val: val}
    current := ll.Head
    for i := 0; i < index-1; i++ {
        current = current.Next
    }
    newNode.Next = current.Next
    current.Next = newNode
    ll.Size++
    return true
}

// DeleteAtIndex - 刪除指定位置 O(n)
func (ll *LinkedList) DeleteAtIndex(index int) bool {
    if index < 0 || index >= ll.Size {
        return false
    }
    
    if index == 0 {
        ll.Head = ll.Head.Next
        ll.Size--
        return true
    }
    
    current := ll.Head
    for i := 0; i < index-1; i++ {
        current = current.Next
    }
    current.Next = current.Next.Next
    ll.Size--
    return true
}

// DeleteByValue - 刪除第一個匹配的值 O(n)
func (ll *LinkedList) DeleteByValue(val int) bool {
    if ll.Head == nil {
        return false
    }
    
    if ll.Head.Val == val {
        ll.Head = ll.Head.Next
        ll.Size--
        return true
    }
    
    current := ll.Head
    for current.Next != nil {
        if current.Next.Val == val {
            current.Next = current.Next.Next
            ll.Size--
            return true
        }
        current = current.Next
    }
    return false
}

// Search - 查找節點 O(n)
func (ll *LinkedList) Search(val int) *ListNode {
    current := ll.Head
    for current != nil {
        if current.Val == val {
            return current
        }
        current = current.Next
    }
    return nil
}

// Reverse - 反轉鏈表 O(n)
func (ll *LinkedList) Reverse() {
    var prev *ListNode
    current := ll.Head
    
    for current != nil {
        next := current.Next
        current.Next = prev
        prev = current
        current = next
    }
    
    ll.Head = prev
}

// Print - 打印鏈表 O(n)
func (ll *LinkedList) Print() {
    current := ll.Head
    for current != nil {
        fmt.Printf("%d -> ", current.Val)
        current = current.Next
    }
    fmt.Println("nil")
}
```

### 4.2 雙向鏈表

```go
type DoublyNode struct {
    Val  int
    Prev *DoublyNode
    Next *DoublyNode
}

type DoublyLinkedList struct {
    Head *DoublyNode
    Tail *DoublyNode
    Size int
}

func (dll *DoublyLinkedList) InsertAtHead(val int) {
    newNode := &DoublyNode{Val: val}
    if dll.Head == nil {
        dll.Head = newNode
        dll.Tail = newNode
    } else {
        newNode.Next = dll.Head
        dll.Head.Prev = newNode
        dll.Head = newNode
    }
    dll.Size++
}

func (dll *DoublyLinkedList) InsertAtTail(val int) {
    newNode := &DoublyNode{Val: val}
    if dll.Tail == nil {
        dll.Head = newNode
        dll.Tail = newNode
    } else {
        newNode.Prev = dll.Tail
        dll.Tail.Next = newNode
        dll.Tail = newNode
    }
    dll.Size++
}

func (dll *DoublyLinkedList) DeleteAtHead() bool {
    if dll.Head == nil {
        return false
    }
    
    if dll.Head == dll.Tail {
        dll.Head = nil
        dll.Tail = nil
    } else {
        dll.Head = dll.Head.Next
        dll.Head.Prev = nil
    }
    dll.Size--
    return true
}

func (dll *DoublyLinkedList) DeleteAtTail() bool {
    if dll.Tail == nil {
        return false
    }
    
    if dll.Head == dll.Tail {
        dll.Head = nil
        dll.Tail = nil
    } else {
        dll.Tail = dll.Tail.Prev
        dll.Tail.Next = nil
    }
    dll.Size--
    return true
}
```

### 4.3 LeetCode 經典題目

#### 題目 1：反轉鏈表（LeetCode 206）

```go
// 迭代法
func reverseList(head *ListNode) *ListNode {
    var prev *ListNode
    current := head
    
    for current != nil {
        next := current.Next
        current.Next = prev
        prev = current
        current = next
    }
    
    return prev
}

// 遞歸法
func reverseListRecursive(head *ListNode) *ListNode {
    if head == nil || head.Next == nil {
        return head
    }
    
    newHead := reverseListRecursive(head.Next)
    head.Next.Next = head
    head.Next = nil
    
    return newHead
}
```

#### 題目 2：環形鏈表（LeetCode 141 - 快慢指針）

```go
func hasCycle(head *ListNode) bool {
    if head == nil || head.Next == nil {
        return false
    }
    
    slow := head
    fast := head.Next
    
    for slow != fast {
        if fast == nil || fast.Next == nil {
            return false
        }
        slow = slow.Next
        fast = fast.Next.Next
    }
    
    return true
}
```

#### 題目 3：合併兩個有序鏈表（LeetCode 21）

```go
func mergeTwoLists(l1 *ListNode, l2 *ListNode) *ListNode {
    dummy := &ListNode{}
    current := dummy
    
    for l1 != nil && l2 != nil {
        if l1.Val < l2.Val {
            current.Next = l1
            l1 = l1.Next
        } else {
            current.Next = l2
            l2 = l2.Next
        }
        current = current.Next
    }
    
    if l1 != nil {
        current.Next = l1
    }
    if l2 != nil {
        current.Next = l2
    }
    
    return dummy.Next
}
```

#### 題目 4：鏈表的中間節點（LeetCode 876）

```go
func middleNode(head *ListNode) *ListNode {
    slow := head
    fast := head
    
    for fast != nil && fast.Next != nil {
        slow = slow.Next
        fast = fast.Next.Next
    }
    
    return slow
}
```

---

## 5. Binary Tree（二叉樹）

### 5.1 樹節點定義

```go
type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

type BinaryTree struct {
    Root *TreeNode
}
```

### 5.2 遍歷方法

#### 前序遍歷（根-左-右）

```go
// 遞歸版本
func preorderTraversal(root *TreeNode) []int {
    result := []int{}
    var traverse func(*TreeNode)
    traverse = func(node *TreeNode) {
        if node == nil {
            return
        }
        result = append(result, node.Val)
        traverse(node.Left)
        traverse(node.Right)
    }
    traverse(root)
    return result
}

// 迭代版本
func preorderTraversalIterative(root *TreeNode) []int {
    if root == nil {
        return []int{}
    }
    
    result := []int{}
    stack := []*TreeNode{root}
    
    for len(stack) > 0 {
        node := stack[len(stack)-1]
        stack = stack[:len(stack)-1]
        result = append(result, node.Val)
        
        // 先壓右子樹，再壓左子樹
        if node.Right != nil {
            stack = append(stack, node.Right)
        }
        if node.Left != nil {
            stack = append(stack, node.Left)
        }
    }
    
    return result
}
```

#### 中序遍歷（左-根-右）

```go
// 遞歸版本
func inorderTraversal(root *TreeNode) []int {
    result := []int{}
    var traverse func(*TreeNode)
    traverse = func(node *TreeNode) {
        if node == nil {
            return
        }
        traverse(node.Left)
        result = append(result, node.Val)
        traverse(node.Right)
    }
    traverse(root)
    return result
}

// 迭代版本
func inorderTraversalIterative(root *TreeNode) []int {
    result := []int{}
    stack := []*TreeNode{}
    current := root
    
    for current != nil || len(stack) > 0 {
        // 一路向左
        for current != nil {
            stack = append(stack, current)
            current = current.Left
        }
        
        // 處理節點
        current = stack[len(stack)-1]
        stack = stack[:len(stack)-1]
        result = append(result, current.Val)
        
        // 轉向右子樹
        current = current.Right
    }
    
    return result
}
```

#### 後序遍歷（左-右-根）

```go
// 遞歸版本
func postorderTraversal(root *TreeNode) []int {
    result := []int{}
    var traverse func(*TreeNode)
    traverse = func(node *TreeNode) {
        if node == nil {
            return
        }
        traverse(node.Left)
        traverse(node.Right)
        result = append(result, node.Val)
    }
    traverse(root)
    return result
}

// 迭代版本
func postorderTraversalIterative(root *TreeNode) []int {
    if root == nil {
        return []int{}
    }
    
    result := []int{}
    stack := []*TreeNode{root}
    
    for len(stack) > 0 {
        node := stack[len(stack)-1]
        stack = stack[:len(stack)-1]
        result = append([]int{node.Val}, result...) // 插入到前面
        
        if node.Left != nil {
            stack = append(stack, node.Left)
        }
        if node.Right != nil {
            stack = append(stack, node.Right)
        }
    }
    
    return result
}
```

#### 層序遍歷（BFS）

```go
func levelOrder(root *TreeNode) [][]int {
    if root == nil {
        return [][]int{}
    }
    
    result := [][]int{}
    queue := []*TreeNode{root}
    
    for len(queue) > 0 {
        levelSize := len(queue)
        level := []int{}
        
        for i := 0; i < levelSize; i++ {
            node := queue[0]
            queue = queue[1:]
            level = append(level, node.Val)
            
            if node.Left != nil {
                queue = append(queue, node.Left)
            }
            if node.Right != nil {
                queue = append(queue, node.Right)
            }
        }
        
        result = append(result, level)
    }
    
    return result
}
```

### 5.3 常見操作

#### 計算樹高度

```go
func maxDepth(root *TreeNode) int {
    if root == nil {
        return 0
    }
    leftDepth := maxDepth(root.Left)
    rightDepth := maxDepth(root.Right)
    return max(leftDepth, rightDepth) + 1
}

func max(a, b int) int {
    if a > b {
        return a
    }
    return b
}
```

#### 查找節點

```go
func searchBST(root *TreeNode, val int) *TreeNode {
    if root == nil || root.Val == val {
        return root
    }
    
    if val < root.Val {
        return searchBST(root.Left, val)
    }
    return searchBST(root.Right, val)
}
```

#### 插入節點（BST）

```go
func insertIntoBST(root *TreeNode, val int) *TreeNode {
    if root == nil {
        return &TreeNode{Val: val}
    }
    
    if val < root.Val {
        root.Left = insertIntoBST(root.Left, val)
    } else {
        root.Right = insertIntoBST(root.Right, val)
    }
    
    return root
}
```

### 5.4 LeetCode 經典題目

#### 題目 1：對稱二叉樹（LeetCode 101）

```go
func isSymmetric(root *TreeNode) bool {
    if root == nil {
        return true
    }
    return isMirror(root.Left, root.Right)
}

func isMirror(left, right *TreeNode) bool {
    if left == nil && right == nil {
        return true
    }
    if left == nil || right == nil {
        return false
    }
    return left.Val == right.Val &&
        isMirror(left.Left, right.Right) &&
        isMirror(left.Right, right.Left)
}
```

#### 題目 2：路徑總和（LeetCode 112）

```go
func hasPathSum(root *TreeNode, targetSum int) bool {
    if root == nil {
        return false
    }
    
    // 葉子節點
    if root.Left == nil && root.Right == nil {
        return root.Val == targetSum
    }
    
    return hasPathSum(root.Left, targetSum-root.Val) ||
        hasPathSum(root.Right, targetSum-root.Val)
}
```

#### 題目 3：翻轉二叉樹（LeetCode 226）

```go
func invertTree(root *TreeNode) *TreeNode {
    if root == nil {
        return nil
    }
    
    root.Left, root.Right = root.Right, root.Left
    invertTree(root.Left)
    invertTree(root.Right)
    
    return root
}
```

---

## 6. Binary Search Tree（BST）

### 6.1 BST 定義與性質

- 左子樹所有節點值 < 根節點值
- 右子樹所有節點值 > 根節點值
- 左右子樹也是 BST
- 中序遍歷結果是有序的

### 6.2 核心操作

#### 搜索節點 O(log n) 平均

```go
func searchBST(root *TreeNode, val int) *TreeNode {
    if root == nil || root.Val == val {
        return root
    }
    
    if val < root.Val {
        return searchBST(root.Left, val)
    }
    return searchBST(root.Right, val)
}
```

#### 插入節點 O(log n) 平均

```go
func insertIntoBST(root *TreeNode, val int) *TreeNode {
    if root == nil {
        return &TreeNode{Val: val}
    }
    
    if val < root.Val {
        root.Left = insertIntoBST(root.Left, val)
    } else {
        root.Right = insertIntoBST(root.Right, val)
    }
    
    return root
}
```

#### 刪除節點 O(log n) 平均

```go
func deleteNode(root *TreeNode, key int) *TreeNode {
    if root == nil {
        return nil
    }
    
    if key < root.Val {
        root.Left = deleteNode(root.Left, key)
    } else if key > root.Val {
        root.Right = deleteNode(root.Right, key)
    } else {
        // 找到要刪除的節點
        // 情況 1: 無子節點
        if root.Left == nil && root.Right == nil {
            return nil
        }
        // 情況 2: 只有一個子節點
        if root.Left == nil {
            return root.Right
        }
        if root.Right == nil {
            return root.Left
        }
        // 情況 3: 有兩個子節點
        // 找到右子樹的最小節點（後繼節點）
        minNode := findMin(root.Right)
        root.Val = minNode.Val
        root.Right = deleteNode(root.Right, minNode.Val)
    }
    
    return root
}

func findMin(node *TreeNode) *TreeNode {
    for node.Left != nil {
        node = node.Left
    }
    return node
}
```

### 6.3 LeetCode 經典題目

#### 題目 1：驗證 BST（LeetCode 98）

```go
func isValidBST(root *TreeNode) bool {
    return validate(root, nil, nil)
}

func validate(node *TreeNode, min, max *int) bool {
    if node == nil {
        return true
    }
    
    if (min != nil && node.Val <= *min) || (max != nil && node.Val >= *max) {
        return false
    }
    
    return validate(node.Left, min, &node.Val) &&
        validate(node.Right, &node.Val, max)
}
```

#### 題目 2：第 K 小元素（LeetCode 230）

```go
func kthSmallest(root *TreeNode, k int) int {
    count := 0
    var result int
    
    var inorder func(*TreeNode)
    inorder = func(node *TreeNode) {
        if node == nil {
            return
        }
        
        inorder(node.Left)
        count++
        if count == k {
            result = node.Val
            return
        }
        inorder(node.Right)
    }
    
    inorder(root)
    return result
}
```

---

## 7. Heap（堆）

### 7.1 使用 container/heap 實現

```go
import "container/heap"

// 最小堆
type MinHeap []int

func (h MinHeap) Len() int           { return len(h) }
func (h MinHeap) Less(i, j int) bool { return h[i] < h[j] }
func (h MinHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }

func (h *MinHeap) Push(x interface{}) {
    *h = append(*h, x.(int))
}

func (h *MinHeap) Pop() interface{} {
    old := *h
    n := len(old)
    x := old[n-1]
    *h = old[0 : n-1]
    return x
}

// 使用示例
func exampleMinHeap() {
    h := &MinHeap{3, 1, 4, 1, 5, 9}
    heap.Init(h)
    heap.Push(h, 2)
    fmt.Println(heap.Pop(h)) // 輸出: 1
}
```

```go
// 最大堆
type MaxHeap []int

func (h MaxHeap) Len() int           { return len(h) }
func (h MaxHeap) Less(i, j int) bool { return h[i] > h[j] } // 注意這裡是 >
func (h MaxHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }

func (h *MaxHeap) Push(x interface{}) {
    *h = append(*h, x.(int))
}

func (h *MaxHeap) Pop() interface{} {
    old := *h
    n := len(old)
    x := old[n-1]
    *h = old[0 : n-1]
    return x
}
```

### 7.2 自定義類型的堆

```go
type Item struct {
    Value    string
    Priority int
}

type PriorityQueue []*Item

func (pq PriorityQueue) Len() int { return len(pq) }

func (pq PriorityQueue) Less(i, j int) bool {
    return pq[i].Priority > pq[j].Priority // 優先級高的在前
}

func (pq PriorityQueue) Swap(i, j int) {
    pq[i], pq[j] = pq[j], pq[i]
}

func (pq *PriorityQueue) Push(x interface{}) {
    item := x.(*Item)
    *pq = append(*pq, item)
}

func (pq *PriorityQueue) Pop() interface{} {
    old := *pq
    n := len(old)
    item := old[n-1]
    *pq = old[0 : n-1]
    return item
}
```

### 7.3 LeetCode 經典題目

#### 題目 1：數組中第 K 大元素（LeetCode 215）

```go
func findKthLargest(nums []int, k int) int {
    h := &MinHeap{}
    heap.Init(h)
    
    for _, num := range nums {
        heap.Push(h, num)
        if h.Len() > k {
            heap.Pop(h)
        }
    }
    
    return (*h)[0]
}
```

#### 題目 2：前 K 個高頻元素（LeetCode 347）

```go
type Pair struct {
    num   int
    count int
}

type FreqHeap []Pair

func (h FreqHeap) Len() int           { return len(h) }
func (h FreqHeap) Less(i, j int) bool { return h[i].count < h[j].count }
func (h FreqHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }

func (h *FreqHeap) Push(x interface{}) {
    *h = append(*h, x.(Pair))
}

func (h *FreqHeap) Pop() interface{} {
    old := *h
    n := len(old)
    x := old[n-1]
    *h = old[0 : n-1]
    return x
}

func topKFrequent(nums []int, k int) []int {
    // 統計頻率
    freqMap := make(map[int]int)
    for _, num := range nums {
        freqMap[num]++
    }
    
    // 使用最小堆
    h := &FreqHeap{}
    heap.Init(h)
    
    for num, count := range freqMap {
        heap.Push(h, Pair{num, count})
        if h.Len() > k {
            heap.Pop(h)
        }
    }
    
    // 提取結果
    result := make([]int, k)
    for i := k - 1; i >= 0; i-- {
        result[i] = heap.Pop(h).(Pair).num
    }
    
    return result
}
```

#### 題目 3：合併 K 個有序鏈表（LeetCode 23）

```go
type ListHeap []*ListNode

func (h ListHeap) Len() int           { return len(h) }
func (h ListHeap) Less(i, j int) bool { return h[i].Val < h[j].Val }
func (h ListHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }

func (h *ListHeap) Push(x interface{}) {
    *h = append(*h, x.(*ListNode))
}

func (h *ListHeap) Pop() interface{} {
    old := *h
    n := len(old)
    x := old[n-1]
    *h = old[0 : n-1]
    return x
}

func mergeKLists(lists []*ListNode) *ListNode {
    h := &ListHeap{}
    heap.Init(h)
    
    // 將所有鏈表的頭節點加入堆
    for _, list := range lists {
        if list != nil {
            heap.Push(h, list)
        }
    }
    
    dummy := &ListNode{}
    current := dummy
    
    for h.Len() > 0 {
        node := heap.Pop(h).(*ListNode)
        current.Next = node
        current = current.Next
        
        if node.Next != nil {
            heap.Push(h, node.Next)
        }
    }
    
    return dummy.Next
}
```

---

## 8. Trie（字典樹）

### 8.1 Trie 節點定義

```go
type TrieNode struct {
    children map[rune]*TrieNode
    isEnd    bool
}

type Trie struct {
    root *TrieNode
}

func NewTrie() *Trie {
    return &Trie{
        root: &TrieNode{
            children: make(map[rune]*TrieNode),
        },
    }
}
```

### 8.2 核心操作

#### 插入字符串 O(m) - m 是字符串長度

```go
func (t *Trie) Insert(word string) {
    node := t.root
    for _, ch := range word {
        if _, exists := node.children[ch]; !exists {
            node.children[ch] = &TrieNode{
                children: make(map[rune]*TrieNode),
            }
        }
        node = node.children[ch]
    }
    node.isEnd = true
}
```

#### 搜索字符串 O(m)

```go
func (t *Trie) Search(word string) bool {
    node := t.root
    for _, ch := range word {
        if _, exists := node.children[ch]; !exists {
            return false
        }
        node = node.children[ch]
    }
    return node.isEnd
}
```

#### 前綴匹配 O(m)

```go
func (t *Trie) StartsWith(prefix string) bool {
    node := t.root
    for _, ch := range prefix {
        if _, exists := node.children[ch]; !exists {
            return false
        }
        node = node.children[ch]
    }
    return true
}
```

#### 刪除字符串

```go
func (t *Trie) Delete(word string) bool {
    return t.deleteHelper(t.root, word, 0)
}

func (t *Trie) deleteHelper(node *TrieNode, word string, index int) bool {
    if node == nil {
        return false
    }
    
    // 到達字符串末尾
    if index == len(word) {
        if !node.isEnd {
            return false
        }
        node.isEnd = false
        return len(node.children) == 0
    }
    
    ch := rune(word[index])
    childNode, exists := node.children[ch]
    if !exists {
        return false
    }
    
    shouldDeleteChild := t.deleteHelper(childNode, word, index+1)
    
    if shouldDeleteChild {
        delete(node.children, ch)
        return len(node.children) == 0 && !node.isEnd
    }
    
    return false
}
```

#### 獲取所有以某前綴開頭的單詞

```go
func (t *Trie) WordsWithPrefix(prefix string) []string {
    node := t.root
    
    // 找到前綴對應的節點
    for _, ch := range prefix {
        if _, exists := node.children[ch]; !exists {
            return []string{}
        }
        node = node.children[ch]
    }
    
    // DFS 收集所有單詞
    results := []string{}
    t.dfs(node, prefix, &results)
    return results
}

func (t *Trie) dfs(node *TrieNode, current string, results *[]string) {
    if node.isEnd {
        *results = append(*results, current)
    }
    
    for ch, child := range node.children {
        t.dfs(child, current+string(ch), results)
    }
}
```

### 8.3 LeetCode 經典題目

#### 題目 1：實現 Trie（LeetCode 208）

```go
type Trie struct {
    root *TrieNode
}

type TrieNode struct {
    children map[rune]*TrieNode
    isEnd    bool
}

func Constructor() Trie {
    return Trie{
        root: &TrieNode{
            children: make(map[rune]*TrieNode),
        },
    }
}

func (t *Trie) Insert(word string) {
    node := t.root
    for _, ch := range word {
        if _, exists := node.children[ch]; !exists {
            node.children[ch] = &TrieNode{
                children: make(map[rune]*TrieNode),
            }
        }
        node = node.children[ch]
    }
    node.isEnd = true
}

func (t *Trie) Search(word string) bool {
    node := t.root
    for _, ch := range word {
        if _, exists := node.children[ch]; !exists {
            return false
        }
        node = node.children[ch]
    }
    return node.isEnd
}

func (t *Trie) StartsWith(prefix string) bool {
    node := t.root
    for _, ch := range prefix {
        if _, exists := node.children[ch]; !exists {
            return false
        }
        node = node.children[ch]
    }
    return true
}
```

#### 題目 2：單詞搜索 II（LeetCode 212）

```go
func findWords(board [][]byte, words []string) []string {
    // 構建 Trie
    trie := NewTrie()
    for _, word := range words {
        trie.Insert(word)
    }
    
    result := make(map[string]bool)
    rows, cols := len(board), len(board[0])
    
    var dfs func(i, j int, node *TrieNode, path string)
    dfs = func(i, j int, node *TrieNode, path string) {
        if i < 0 || i >= rows || j < 0 || j >= cols || board[i][j] == '#' {
            return
        }
        
        ch := rune(board[i][j])
        if _, exists := node.children[ch]; !exists {
            return
        }
        
        node = node.children[ch]
        path += string(ch)
        
        if node.isEnd {
            result[path] = true
        }
        
        // 標記已訪問
        temp := board[i][j]
        board[i][j] = '#'
        
        // 四個方向
        dfs(i+1, j, node, path)
        dfs(i-1, j, node, path)
        dfs(i, j+1, node, path)
        dfs(i, j-1, node, path)
        
        // 回溯
        board[i][j] = temp
    }
    
    for i := 0; i < rows; i++ {
        for j := 0; j < cols; j++ {
            dfs(i, j, trie.root, "")
        }
    }
    
    // 轉換為切片
    res := []string{}
    for word := range result {
        res = append(res, word)
    }
    return res
}
```

---

## 9. Graph（圖）

### 9.1 圖的表示

#### 鄰接表（常用）

```go
type Graph struct {
    vertices int
    adjList  map[int][]int
}

func NewGraph(vertices int) *Graph {
    return &Graph{
        vertices: vertices,
        adjList:  make(map[int][]int),
    }
}

func (g *Graph) AddEdge(u, v int) {
    g.adjList[u] = append(g.adjList[u], v)
    // 無向圖需要雙向添加
    // g.adjList[v] = append(g.adjList[v], u)
}
```

#### 鄰接矩陣

```go
type GraphMatrix struct {
    vertices int
    matrix   [][]int
}

func NewGraphMatrix(vertices int) *GraphMatrix {
    matrix := make([][]int, vertices)
    for i := range matrix {
        matrix[i] = make([]int, vertices)
    }
    return &GraphMatrix{
        vertices: vertices,
        matrix:   matrix,
    }
}

func (g *GraphMatrix) AddEdge(u, v int) {
    g.matrix[u][v] = 1
    // 無向圖
    // g.matrix[v][u] = 1
}
```

### 9.2 深度優先搜索（DFS）

#### 遞歸實現

```go
func (g *Graph) DFS(start int) []int {
    visited := make(map[int]bool)
    result := []int{}
    
    var dfs func(int)
    dfs = func(v int) {
        visited[v] = true
        result = append(result, v)
        
        for _, neighbor := range g.adjList[v] {
            if !visited[neighbor] {
                dfs(neighbor)
            }
        }
    }
    
    dfs(start)
    return result
}
```

#### 迭代實現（使用棧）

```go
func (g *Graph) DFSIterative(start int) []int {
    visited := make(map[int]bool)
    result := []int{}
    stack := []int{start}
    
    for len(stack) > 0 {
        v := stack[len(stack)-1]
        stack = stack[:len(stack)-1]
        
        if visited[v] {
            continue
        }
        
        visited[v] = true
        result = append(result, v)
        
        // 逆序添加鄰居（保證順序一致）
        neighbors := g.adjList[v]
        for i := len(neighbors) - 1; i >= 0; i-- {
            if !visited[neighbors[i]] {
                stack = append(stack, neighbors[i])
            }
        }
    }
    
    return result
}
```

### 9.3 廣度優先搜索（BFS）

```go
func (g *Graph) BFS(start int) []int {
    visited := make(map[int]bool)
    result := []int{}
    queue := []int{start}
    visited[start] = true
    
    for len(queue) > 0 {
        v := queue[0]
        queue = queue[1:]
        result = append(result, v)
        
        for _, neighbor := range g.adjList[v] {
            if !visited[neighbor] {
                visited[neighbor] = true
                queue = append(queue, neighbor)
            }
        }
    }
    
    return result
}
```

### 9.4 檢測環（有向圖）

```go
func (g *Graph) HasCycle() bool {
    visited := make(map[int]bool)
    recStack := make(map[int]bool)
    
    var dfs func(int) bool
    dfs = func(v int) bool {
        visited[v] = true
        recStack[v] = true
        
        for _, neighbor := range g.adjList[v] {
            if !visited[neighbor] {
                if dfs(neighbor) {
                    return true
                }
            } else if recStack[neighbor] {
                return true
            }
        }
        
        recStack[v] = false
        return false
    }
    
    for v := 0; v < g.vertices; v++ {
        if !visited[v] {
            if dfs(v) {
                return true
            }
        }
    }
    
    return false
}
```

### 9.5 拓撲排序

```go
func (g *Graph) TopologicalSort() []int {
    inDegree := make(map[int]int)
    
    // 計算入度
    for v := 0; v < g.vertices; v++ {
        for _, neighbor := range g.adjList[v] {
            inDegree[neighbor]++
        }
    }
    
    // 將入度為 0 的節點加入隊列
    queue := []int{}
    for v := 0; v < g.vertices; v++ {
        if inDegree[v] == 0 {
            queue = append(queue, v)
        }
    }
    
    result := []int{}
    
    for len(queue) > 0 {
        v := queue[0]
        queue = queue[1:]
        result = append(result, v)
        
        for _, neighbor := range g.adjList[v] {
            inDegree[neighbor]--
            if inDegree[neighbor] == 0 {
                queue = append(queue, neighbor)
            }
        }
    }
    
    // 如果結果長度不等於頂點數，說明有環
    if len(result) != g.vertices {
        return []int{} // 有環，無法拓撲排序
    }
    
    return result
}
```

### 9.6 LeetCode 經典題目

#### 題目 1：克隆圖（LeetCode 133）

```go
type Node struct {
    Val       int
    Neighbors []*Node
}

func cloneGraph(node *Node) *Node {
    if node == nil {
        return nil
    }
    
    visited := make(map[*Node]*Node)
    
    var clone func(*Node) *Node
    clone = func(n *Node) *Node {
        if copied, exists := visited[n]; exists {
            return copied
        }
        
        newNode := &Node{Val: n.Val, Neighbors: []*Node{}}
        visited[n] = newNode
        
        for _, neighbor := range n.Neighbors {
            newNode.Neighbors = append(newNode.Neighbors, clone(neighbor))
        }
        
        return newNode
    }
    
    return clone(node)
}
```

#### 題目 2：課程表（LeetCode 207 - 檢測有向圖環）

```go
func canFinish(numCourses int, prerequisites [][]int) bool {
    // 構建圖
    graph := make(map[int][]int)
    for _, pre := range prerequisites {
        graph[pre[0]] = append(graph[pre[0]], pre[1])
    }
    
    visited := make(map[int]bool)
    recStack := make(map[int]bool)
    
    var hasCycle func(int) bool
    hasCycle = func(course int) bool {
        visited[course] = true
        recStack[course] = true
        
        for _, pre := range graph[course] {
            if !visited[pre] {
                if hasCycle(pre) {
                    return true
                }
            } else if recStack[pre] {
                return true
            }
        }
        
        recStack[course] = false
        return false
    }
    
    for i := 0; i < numCourses; i++ {
        if !visited[i] {
            if hasCycle(i) {
                return false
            }
        }
    }
    
    return true
}
```

#### 題目 3：島嶼數量（LeetCode 200 - DFS/BFS）

```go
func numIslands(grid [][]byte) int {
    if len(grid) == 0 {
        return 0
    }
    
    rows, cols := len(grid), len(grid[0])
    count := 0
    
    var dfs func(i, j int)
    dfs = func(i, j int) {
        if i < 0 || i >= rows || j < 0 || j >= cols || grid[i][j] == '0' {
            return
        }
        
        grid[i][j] = '0' // 標記為已訪問
        
        dfs(i+1, j)
        dfs(i-1, j)
        dfs(i, j+1)
        dfs(i, j-1)
    }
    
    for i := 0; i < rows; i++ {
        for j := 0; j < cols; j++ {
            if grid[i][j] == '1' {
                count++
                dfs(i, j)
            }
        }
    }
    
    return count
}
```

---

## 10. Union-Find（並查集）

### 10.1 基本實現

```go
type UnionFind struct {
    parent []int
    rank   []int
}

func NewUnionFind(size int) *UnionFind {
    parent := make([]int, size)
    rank := make([]int, size)
    for i := range parent {
        parent[i] = i
        rank[i] = 1
    }
    return &UnionFind{
        parent: parent,
        rank:   rank,
    }
}

// Find - 查找根節點（路徑壓縮）O(α(n)) ≈ O(1)
func (uf *UnionFind) Find(x int) int {
    if uf.parent[x] != x {
        uf.parent[x] = uf.Find(uf.parent[x]) // 路徑壓縮
    }
    return uf.parent[x]
}

// Union - 合併集合（按秩合併）O(α(n)) ≈ O(1)
func (uf *UnionFind) Union(x, y int) bool {
    rootX := uf.Find(x)
    rootY := uf.Find(y)
    
    if rootX == rootY {
        return false // 已經在同一集合
    }
    
    // 按秩合併
    if uf.rank[rootX] < uf.rank[rootY] {
        uf.parent[rootX] = rootY
    } else if uf.rank[rootX] > uf.rank[rootY] {
        uf.parent[rootY] = rootX
    } else {
        uf.parent[rootY] = rootX
        uf.rank[rootX]++
    }
    
    return true
}

// Connected - 判斷是否在同一集合 O(α(n))
func (uf *UnionFind) Connected(x, y int) bool {
    return uf.Find(x) == uf.Find(y)
}
```

### 10.2 擴展操作

```go
// 獲取連通分量數量
func (uf *UnionFind) Count() int {
    roots := make(map[int]bool)
    for i := range uf.parent {
        roots[uf.Find(i)] = true
    }
    return len(roots)
}

// 獲取集合大小
type UnionFindWithSize struct {
    parent []int
    size   []int
}

func NewUnionFindWithSize(n int) *UnionFindWithSize {
    parent := make([]int, n)
    size := make([]int, n)
    for i := range parent {
        parent[i] = i
        size[i] = 1
    }
    return &UnionFindWithSize{parent: parent, size: size}
}

func (uf *UnionFindWithSize) Find(x int) int {
    if uf.parent[x] != x {
        uf.parent[x] = uf.Find(uf.parent[x])
    }
    return uf.parent[x]
}

func (uf *UnionFindWithSize) Union(x, y int) {
    rootX, rootY := uf.Find(x), uf.Find(y)
    if rootX == rootY {
        return
    }
    
    if uf.size[rootX] < uf.size[rootY] {
        uf.parent[rootX] = rootY
        uf.size[rootY] += uf.size[rootX]
    } else {
        uf.parent[rootY] = rootX
        uf.size[rootX] += uf.size[rootY]
    }
}

func (uf *UnionFindWithSize) GetSize(x int) int {
    return uf.size[uf.Find(x)]
}
```

### 10.3 LeetCode 經典題目

#### 題目 1：朋友圈（LeetCode 547）

```go
func findCircleNum(isConnected [][]int) int {
    n := len(isConnected)
    uf := NewUnionFind(n)
    
    for i := 0; i < n; i++ {
        for j := i + 1; j < n; j++ {
            if isConnected[i][j] == 1 {
                uf.Union(i, j)
            }
        }
    }
    
    return uf.Count()
}
```

#### 題目 2：冗餘連接（LeetCode 684）

```go
func findRedundantConnection(edges [][]int) []int {
    n := len(edges)
    uf := NewUnionFind(n + 1)
    
    for _, edge := range edges {
        u, v := edge[0], edge[1]
        if !uf.Union(u, v) {
            return edge // 發現環
        }
    }
    
    return []int{}
}
```

#### 題目 3：島嶼數量（LeetCode 200 - Union-Find 解法）

```go
func numIslands(grid [][]byte) int {
    if len(grid) == 0 {
        return 0
    }
    
    rows, cols := len(grid), len(grid[0])
    uf := NewUnionFind(rows * cols)
    count := 0
    
    // 統計初始島嶼數量
    for i := 0; i < rows; i++ {
        for j := 0; j < cols; j++ {
            if grid[i][j] == '1' {
                count++
            }
        }
    }
    
    directions := [][]int{{1, 0}, {0, 1}}
    
    for i := 0; i < rows; i++ {
        for j := 0; j < cols; j++ {
            if grid[i][j] == '1' {
                for _, dir := range directions {
                    ni, nj := i+dir[0], j+dir[1]
                    if ni < rows && nj < cols && grid[ni][nj] == '1' {
                        id1 := i*cols + j
                        id2 := ni*cols + nj
                        if uf.Union(id1, id2) {
                            count--
                        }
                    }
                }
            }
        }
    }
    
    return count
}
```

---

## 11. LRU Cache（綜合應用）

### 11.1 使用 HashMap + 雙向鏈表實現

```go
type LRUNode struct {
    key   int
    value int
    prev  *LRUNode
    next  *LRUNode
}

type LRUCache struct {
    capacity int
    cache    map[int]*LRUNode
    head     *LRUNode // 虛擬頭節點
    tail     *LRUNode // 虛擬尾節點
}

func Constructor(capacity int) LRUCache {
    head := &LRUNode{}
    tail := &LRUNode{}
    head.next = tail
    tail.prev = head
    
    return LRUCache{
        capacity: capacity,
        cache:    make(map[int]*LRUNode),
        head:     head,
        tail:     tail,
    }
}

// Get - O(1)
func (lru *LRUCache) Get(key int) int {
    if node, exists := lru.cache[key]; exists {
        lru.moveToHead(node)
        return node.value
    }
    return -1
}

// Put - O(1)
func (lru *LRUCache) Put(key int, value int) {
    if node, exists := lru.cache[key]; exists {
        node.value = value
        lru.moveToHead(node)
    } else {
        newNode := &LRUNode{key: key, value: value}
        lru.cache[key] = newNode
        lru.addToHead(newNode)
        
        if len(lru.cache) > lru.capacity {
            removed := lru.removeTail()
            delete(lru.cache, removed.key)
        }
    }
}

// 移動到頭部（最近使用）
func (lru *LRUCache) moveToHead(node *LRUNode) {
    lru.removeNode(node)
    lru.addToHead(node)
}

// 添加到頭部
func (lru *LRUCache) addToHead(node *LRUNode) {
    node.prev = lru.head
    node.next = lru.head.next
    lru.head.next.prev = node
    lru.head.next = node
}

// 移除節點
func (lru *LRUCache) removeNode(node *LRUNode) {
    node.prev.next = node.next
    node.next.prev = node.prev
}

// 移除尾部（最久未使用）
func (lru *LRUCache) removeTail() *LRUNode {
    node := lru.tail.prev
    lru.removeNode(node)
    return node
}
```

### 11.2 使用示例

```go
func main() {
    cache := Constructor(2)
    
    cache.Put(1, 1)
    cache.Put(2, 2)
    fmt.Println(cache.Get(1))    // 返回 1
    cache.Put(3, 3)              // 移除 key 2
    fmt.Println(cache.Get(2))    // 返回 -1 (未找到)
    cache.Put(4, 4)              // 移除 key 1
    fmt.Println(cache.Get(1))    // 返回 -1 (未找到)
    fmt.Println(cache.Get(3))    // 返回 3
    fmt.Println(cache.Get(4))    // 返回 4
}
```

---

## 12. 資料結構選擇指南

### 12.1 時間複雜度對比表

| 資料結構        | 訪問       | 搜索       | 插入       | 刪除       | 空間複雜度                    |
| ----------- | -------- | -------- | -------- | -------- | ------------------------ |
| Array       | O(1)     | O(n)     | O(n)     | O(n)     | O(n)                     |
| LinkedList  | O(n)     | O(n)     | O(1)*    | O(1)*    | O(n)                     |
| Stack       | O(n)     | O(n)     | O(1)     | O(1)     | O(n)                     |
| Queue       | O(n)     | O(n)     | O(1)     | O(1)     | O(n)                     |
| Hash Table  | -        | O(1)**   | O(1)**   | O(1)**   | O(n)                     |
| Binary Tree | O(n)     | O(n)     | O(n)     | O(n)     | O(n)                     |
| BST (平衡)    | O(log n) | O(log n) | O(log n) | O(log n) | O(n)                     |
| Heap        | O(1)***  | O(n)     | O(log n) | O(log n) | O(n)                     |
| Trie        | O(m)     | O(m)     | O(m)     | O(m)     | O(ALPHABET_SIZE * N * M) |

\* 假設已知位置  
\** 平均情況  
\*** 僅堆頂元素

### 12.2 LeetCode 題型與資料結構映射

| 題型 | 推薦資料結構 | 典型題目 |
|------|------------|---------|
| 括號匹配 | Stack | LC 20, 32, 678 |
| 滑動窗口最大值 | Deque (單調隊列) | LC 239 |
| BFS 遍歷 | Queue | LC 102, 107, 199 |
| 鏈表操作 | LinkedList | LC 206, 141, 21 |
| 路徑和 | Binary Tree (DFS) | LC 112, 113, 437 |
| Top K 問題 | Heap | LC 215, 347, 373 |
| 前綴匹配 | Trie | LC 208, 212, 648 |
| 圖遍歷 | Graph (DFS/BFS) | LC 200, 133, 207 |
| 連通性問題 | Union-Find | LC 547, 684, 990 |
| LRU/LFU Cache | HashMap + LinkedList | LC 146, 460 |

### 12.3 常見陷阱與最佳實踐

#### ❌ 常見陷阱

1. **使用 Slice 模擬 Queue 時頻繁的記憶體重分配**
   - 解決：使用環形佇列或雙端佇列
   
2. **忘記 BST 刪除節點的三種情況**
   - 無子節點、一個子節點、兩個子節點
   
3. **Heap 操作後忘記調用 `heap.Init()` 或 `heap.Fix()`**
   
4. **Union-Find 未實現路徑壓縮導致性能下降**
   
5. **Trie 內存佔用過大**
   - 解決：考慮使用壓縮 Trie (Radix Tree)

#### ✅ 最佳實踐

1. **使用泛型（Go 1.18+）提升代碼復用性**
   ```go
   type Stack[T any] struct { items []T }
   ```

2. **預分配容量避免擴容**
   ```go
   slice := make([]int, 0, expectedSize)
   ```

3. **使用 `container/heap` 而非手動實現堆**
   
4. **鏈表操作使用虛擬頭節點簡化邊界處理**
   ```go
   dummy := &ListNode{}
   dummy.Next = head
   ```

5. **圖遍歷時使用 `visited` 集合避免重複訪問**

6. **並查集必須實現路徑壓縮和按秩合併**

---

## 附錄：完整測試範例

```go
package main

import "fmt"

func main() {
    // Stack 測試
    stack := &Stack[int]{}
    stack.Push(1)
    stack.Push(2)
    fmt.Println(stack.Pop()) // 2
    
    // Queue 測試
    queue := &Queue[string]{}
    queue.Enqueue("first")
    queue.Enqueue("second")
    fmt.Println(queue.Dequeue()) // first
    
    // LinkedList 測試
    ll := &LinkedList{}
    ll.InsertAtHead(1)
    ll.InsertAtTail(2)
    ll.Print() // 1 -> 2 -> nil
    
    // Heap 測試
    h := &MinHeap{3, 1, 4, 1, 5}
    heap.Init(h)
    fmt.Println(heap.Pop(h)) // 1
    
    // Trie 測試
    trie := NewTrie()
    trie.Insert("apple")
    fmt.Println(trie.Search("apple"))   // true
    fmt.Println(trie.StartsWith("app")) // true
    
    // Union-Find 測試
    uf := NewUnionFind(5)
    uf.Union(0, 1)
    uf.Union(1, 2)
    fmt.Println(uf.Connected(0, 2)) // true
    
    // LRU Cache 測試
    cache := Constructor(2)
    cache.Put(1, 1)
    cache.Put(2, 2)
    fmt.Println(cache.Get(1)) // 1
}
```

---

## 總結

這份指南涵蓋了 LeetCode 刷題必備的所有核心資料結構，包括：

✅ **線性結構**: Stack, Queue, Deque, LinkedList  
✅ **樹結構**: Binary Tree, BST, Heap, Trie  
✅ **圖結構**: Graph (DFS/BFS), Union-Find  
✅ **綜合應用**: LRU Cache  

每個資料結構都包含：
- 完整的 Go 實現
- 時間/空間複雜度分析
- LeetCode 真題範例
- 最佳實踐建議

建議學習路徑：
1. 先掌握基礎線性結構（Stack, Queue, LinkedList）
2. 再學習樹結構（Binary Tree, BST, Heap）
3. 最後挑戰圖結構和綜合應用

祝刷題順利！🚀
