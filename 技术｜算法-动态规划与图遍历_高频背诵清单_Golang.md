# 动态规划与图遍历：高频背诵清单（Golang）

> 适用范围：此前确定的 70 道后端算法面试题。  
> 目标：只背最有代表性的题目，用少量母题覆盖动态规划、图遍历和回溯中的高频模板。

---

# 一、动态规划：建议背 5 题

## 第一优先级：必须默写

---

## 1. 53 最大子数组和

### 代表模板

```text
以当前位置结尾的最优值
```

### 核心状态

```text
dp[i] = max(nums[i], dp[i-1] + nums[i])
```

### Golang 模板

```go
func maxSubArray(nums []int) int {
	current := nums[0]
	answer := nums[0]

	for i := 1; i < len(nums); i++ {
		current = max(nums[i], current+nums[i])
		answer = max(answer, current)
	}

	return answer
}
```

### 必须掌握的点

```text
current 表示：以当前位置结尾的最大子数组和
answer 表示：全局最大子数组和
```

---

## 2. 198 打家劫舍

### 代表模板

```text
选或不选
```

### 核心状态

```text
选当前：dp[i-2] + nums[i]
不选当前：dp[i-1]
```

### Golang 模板

```go
func rob(nums []int) int {
	previousTwo := 0
	previousOne := 0

	for _, value := range nums {
		current := max(previousOne, previousTwo+value)
		previousTwo = previousOne
		previousOne = current
	}

	return previousOne
}
```

### 必须掌握的点

```text
previousTwo：dp[i-2]
previousOne：dp[i-1]
current：dp[i]
```

---

## 3. 322 零钱兑换

### 代表模板

```text
完全背包
每种硬币可以重复使用
求最少数量
```

### 核心状态

```text
dp[x]：凑出金额 x 所需的最少硬币数
dp[x] = min(dp[x], dp[x-coin] + 1)
dp[0] = 0
```

### Golang 模板

```go
func coinChange(coins []int, amount int) int {
	const inf = int(^uint(0) >> 1)

	dp := make([]int, amount+1)

	for current := 1; current <= amount; current++ {
		dp[current] = inf

		for _, coin := range coins {
			if current >= coin && dp[current-coin] != inf {
				dp[current] = min(
					dp[current],
					dp[current-coin]+1,
				)
			}
		}
	}

	if dp[amount] == inf {
		return -1
	}

	return dp[amount]
}
```

---

## 4. 139 单词拆分

### 代表模板

```text
字符串分割 DP
枚举最后一段
```

### 识别关键词

```text
能否拆分
前缀是否合法
最后一段是什么
```

### Golang 模板

```go
func wordBreak(s string, wordDict []string) bool {
	dictionary := make(map[string]struct{})

	for _, word := range wordDict {
		dictionary[word] = struct{}{}
	}

	dp := make([]bool, len(s)+1)
	dp[0] = true

	for end := 1; end <= len(s); end++ {
		for start := 0; start < end; start++ {
			if !dp[start] {
				continue
			}

			if _, exists := dictionary[s[start:end]]; exists {
				dp[end] = true
				break
			}
		}
	}

	return dp[len(s)]
}
```

### 必须掌握的点

```text
dp[end] 表示：s[0:end] 能否被字典中的单词拆分
枚举 start，相当于枚举最后一个单词
```

---

## 5. 1143 最长公共子序列

### 代表模板

```text
两个序列的二维 DP
```

### 核心状态

```text
dp[i][j]：
first 前 i 个字符与 second 前 j 个字符的最长公共子序列长度
```

### 状态转移

```text
字符相等：看左上角
字符不等：看上方和左方
```

### Golang 模板

```go
func longestCommonSubsequence(first, second string) int {
	m, n := len(first), len(second)

	dp := make([][]int, m+1)
	for i := range dp {
		dp[i] = make([]int, n+1)
	}

	for i := 1; i <= m; i++ {
		for j := 1; j <= n; j++ {
			if first[i-1] == second[j-1] {
				dp[i][j] = dp[i-1][j-1] + 1
			} else {
				dp[i][j] = max(
					dp[i-1][j],
					dp[i][j-1],
				)
			}
		}
	}

	return dp[m][n]
}
```

---

# 二、动态规划第二优先级

## 6. 300 最长递增子序列

### 建议掌握程度

记住核心思想和 `sort.Search` 写法，不必把它和普通二维 DP 混在一起背。

### 核心思想

```text
tails[i]：
长度为 i+1 的递增子序列中，最小的结尾值
```

### Golang 模板

```go
import "sort"

func lengthOfLIS(nums []int) int {
	tails := make([]int, 0)

	for _, value := range nums {
		index := sort.Search(
			len(tails),
			func(i int) bool {
				return tails[i] >= value
			},
		)

		if index == len(tails) {
			tails = append(tails, value)
		} else {
			tails[index] = value
		}
	}

	return len(tails)
}
```

---

# 三、图遍历：建议背 4 题

## 第一优先级：必须默写

---

## 1. 200 岛屿数量

### 代表模板

```text
网格 DFS
连通分量计数
```

### 必须背的结构

```text
越界判断
当前格子是否合法
标记已访问
向四个方向递归
```

### Golang 模板

```go
func numIslands(grid [][]byte) int {
	if len(grid) == 0 {
		return 0
	}

	rows, cols := len(grid), len(grid[0])
	answer := 0

	var dfs func(row, col int)

	dfs = func(row, col int) {
		if row < 0 || row >= rows ||
			col < 0 || col >= cols ||
			grid[row][col] != '1' {
			return
		}

		grid[row][col] = '0'

		dfs(row-1, col)
		dfs(row+1, col)
		dfs(row, col-1)
		dfs(row, col+1)
	}

	for row := 0; row < rows; row++ {
		for col := 0; col < cols; col++ {
			if grid[row][col] == '1' {
				answer++
				dfs(row, col)
			}
		}
	}

	return answer
}
```

---

## 2. 994 腐烂的橘子

### 代表模板

```text
多源 BFS
多个起点同时扩散
按层统计时间
```

### 核心区别

```text
开始时，把所有起点一次性加入队列
```

### Golang 模板

```go
func orangesRotting(grid [][]int) int {
	rows, cols := len(grid), len(grid[0])
	queue := make([][2]int, 0)
	fresh := 0

	for row := 0; row < rows; row++ {
		for col := 0; col < cols; col++ {
			if grid[row][col] == 2 {
				queue = append(queue, [2]int{row, col})
			} else if grid[row][col] == 1 {
				fresh++
			}
		}
	}

	directions := [][2]int{
		{-1, 0},
		{1, 0},
		{0, -1},
		{0, 1},
	}

	minutes := 0

	for len(queue) > 0 && fresh > 0 {
		size := len(queue)

		for i := 0; i < size; i++ {
			current := queue[0]
			queue = queue[1:]

			for _, direction := range directions {
				nextRow := current[0] + direction[0]
				nextCol := current[1] + direction[1]

				if nextRow < 0 || nextRow >= rows ||
					nextCol < 0 || nextCol >= cols ||
					grid[nextRow][nextCol] != 1 {
					continue
				}

				grid[nextRow][nextCol] = 2
				fresh--
				queue = append(queue, [2]int{nextRow, nextCol})
			}
		}

		minutes++
	}

	if fresh > 0 {
		return -1
	}

	return minutes
}
```

---

## 3. 207 课程表

### 代表模板

```text
有向图
依赖关系
拓扑排序
判断是否有环
```

### 必须背的四步

```text
1. 建邻接表
2. 统计入度
3. 入度为 0 的节点入队
4. 处理节点并减少后继节点入度
```

### Golang 模板

```go
func canFinish(numCourses int, prerequisites [][]int) bool {
	graph := make([][]int, numCourses)
	indegree := make([]int, numCourses)

	for _, edge := range prerequisites {
		course := edge[0]
		prerequisite := edge[1]

		graph[prerequisite] = append(graph[prerequisite], course)
		indegree[course]++
	}

	queue := make([]int, 0)

	for course := 0; course < numCourses; course++ {
		if indegree[course] == 0 {
			queue = append(queue, course)
		}
	}

	visited := 0

	for len(queue) > 0 {
		current := queue[0]
		queue = queue[1:]
		visited++

		for _, next := range graph[current] {
			indegree[next]--

			if indegree[next] == 0 {
				queue = append(queue, next)
			}
		}
	}

	return visited == numCourses
}
```

---

## 4. 79 单词搜索

### 代表模板

```text
网格 DFS + 回溯
访问后还要恢复
```

### 和岛屿数量的区别

```text
岛屿数量：标记后不恢复
单词搜索：当前路径结束后必须恢复
```

### Golang 模板

```go
func exist(board [][]byte, word string) bool {
	rows, cols := len(board), len(board[0])

	var dfs func(row, col, index int) bool

	dfs = func(row, col, index int) bool {
		if index == len(word) {
			return true
		}

		if row < 0 || row >= rows ||
			col < 0 || col >= cols ||
			board[row][col] != word[index] {
			return false
		}

		original := board[row][col]
		board[row][col] = '#'

		found := dfs(row-1, col, index+1) ||
			dfs(row+1, col, index+1) ||
			dfs(row, col-1, index+1) ||
			dfs(row, col+1, index+1)

		board[row][col] = original

		return found
	}

	for row := 0; row < rows; row++ {
		for col := 0; col < cols; col++ {
			if dfs(row, col, 0) {
				return true
			}
		}
	}

	return false
}
```

---

# 四、图遍历第二优先级

## 5. 547 省份数量

### 代表模板

```text
并查集
连通分量
集合合并
```

### 建议掌握程度

建议背 `Find` 和 `Union` 两个函数。  
时间极度有限时，也可以直接用 DFS 解，不一定必须使用并查集。

### Golang 模板

```go
type UnionFind struct {
	parent []int
	count  int
}

func NewUnionFind(n int) *UnionFind {
	parent := make([]int, n)

	for i := range parent {
		parent[i] = i
	}

	return &UnionFind{
		parent: parent,
		count:  n,
	}
}

func (u *UnionFind) Find(x int) int {
	if u.parent[x] != x {
		u.parent[x] = u.Find(u.parent[x])
	}

	return u.parent[x]
}

func (u *UnionFind) Union(a, b int) {
	rootA := u.Find(a)
	rootB := u.Find(b)

	if rootA == rootB {
		return
	}

	u.parent[rootB] = rootA
	u.count--
}
```

---

# 五、回溯：建议额外背 2 题

---

## 1. 46 全排列

### 代表模板

```text
used 数组
每一层从所有元素中选择
顺序不同算不同答案
```

### Golang 模板

```go
func permute(nums []int) [][]int {
	answer := make([][]int, 0)
	path := make([]int, 0, len(nums))
	used := make([]bool, len(nums))

	var backtrack func()

	backtrack = func() {
		if len(path) == len(nums) {
			answer = append(answer, append([]int(nil), path...))
			return
		}

		for i := 0; i < len(nums); i++ {
			if used[i] {
				continue
			}

			used[i] = true
			path = append(path, nums[i])

			backtrack()

			path = path[:len(path)-1]
			used[i] = false
		}
	}

	backtrack()

	return answer
}
```

---

## 2. 39 组合总和

### 代表模板

```text
start 下标
同一个元素可以重复使用
排序后剪枝
```

### Golang 模板

```go
import "sort"

func combinationSum(candidates []int, target int) [][]int {
	sort.Ints(candidates)

	answer := make([][]int, 0)
	path := make([]int, 0)

	var backtrack func(start, remain int)

	backtrack = func(start, remain int) {
		if remain == 0 {
			answer = append(answer, append([]int(nil), path...))
			return
		}

		for i := start; i < len(candidates); i++ {
			value := candidates[i]

			if value > remain {
				break
			}

			path = append(path, value)

			backtrack(i, remain-value)

			path = path[:len(path)-1]
		}
	}

	backtrack(0, target)

	return answer
}
```

### 和 78 子集的关系

```text
78 子集不需要单独死背
只需把“每到一个节点都保存一次 path”加入组合模板即可
```

---

# 六、最终最小背诵清单

## 动态规划必须背

```text
53   最大子数组和
198  打家劫舍
322  零钱兑换
139  单词拆分
1143 最长公共子序列
```

## 动态规划第二优先级

```text
300 最长递增子序列
```

## 图遍历必须背

```text
200 岛屿数量
994 腐烂的橘子
207 课程表
79  单词搜索
```

## 回溯必须背

```text
46 全排列
39 组合总和
```

## 图遍历第二优先级

```text
547 省份数量
```

---

# 七、最小背诵数量

优先背熟 11 道代表题：

```text
DP：5 道
图遍历：4 道
回溯：2 道
```

这 11 道题覆盖：

```text
一维 DP
选或不选 DP
完全背包
字符串分割 DP
二维序列 DP
普通网格 DFS
带恢复的网格 DFS
多源 BFS
拓扑排序
排列回溯
组合回溯
```

时间极度有限时，先练到闭眼能写：

```text
200 岛屿数量
994 腐烂的橘子
207 课程表
53  最大子数组和
198 打家劫舍
322 零钱兑换
```

---

# 八、背诵标准

## 动态规划

每道题必须能说清楚：

```text
1. 状态定义
2. 状态转移方程
3. 初始化
4. 遍历顺序
5. 最终答案
```

## 图遍历

每道题必须能说清楚：

```text
1. 节点是什么
2. 边是什么
3. 何时标记 visited
4. 队列或递归中保存什么
5. 什么时候结束
```

## 回溯

每道题必须能说清楚：

```text
1. 当前有哪些选择
2. 什么时候结束
3. 如何做选择
4. 如何递归
5. 如何撤销选择
```

---

# 九、面试现场识别口诀

```text
连续最优：先想一维 DP
不能相邻选：选或不选
可以重复凑目标：完全背包
字符串能否拆分：枚举最后一段
两个字符串比较：二维 DP
岛屿和连通区域：网格 DFS
多个起点同时扩散：多源 BFS
依赖和先修关系：拓扑排序
网格路径要恢复：DFS + 回溯
所有排列：used 数组
组合不看顺序：start 下标
```
