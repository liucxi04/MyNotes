## Go 小点

- 定义数组：`dp := make([]int, n)`

- 定义变量：`pre2, pre1 := 0, nums[0]`

- 数组长度：`l := length(nums)`

- 定义二维数组：

```go
dp := make([][]int, m)
for i := 0; i < m; i++ {
	dp[i] = make([]int, n)
}
```

- 用 if else 代替三元运算符

- min 函数：

```go
func min(a, b int) int {
    if a > b {
        return b
    } else {
        return a
    }
}
```

- 最值：Math.MaxInt16。。。
- 范围 for 循环：`for k, v := range wordDict`

- strings.Count(str, "0")
