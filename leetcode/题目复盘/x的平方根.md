# x 的平方根

LeetCode 69。

## 题意

给定非负整数 `x`，返回 `x` 的算术平方根的整数部分，不能使用内置平方根函数。

```text
x = 8

返回 2，因为 2 * 2 <= 8，而 3 * 3 > 8。
```

## 我的思路

这次二分的不是数组下标，而是答案范围 `[0, x]`。

- 如果 `middle * middle <= x`，说明 `middle` 是合法答案，先记录，再向右寻找更大的合法值。
- 如果 `middle * middle > x`，说明 `middle` 太大，向左缩小范围。
- 循环结束后，`best` 保存最大的合法整数。

## 标准模板

```python
left = 0
right = x
best = 0

while left <= right:
    middle = left + (right - left) // 2

    if middle * middle <= x:
        best = middle
        left = middle + 1
    else:
        right = middle - 1

return best
```

## 复杂度

- 时间复杂度：O(log x)。
- 空间复杂度：O(1)。

## 踩坑点

- 目标是最大的合法值，不是找到一个合法值后就返回。
- 条件成立时要继续向右搜索，因此更新 `left = middle + 1`。
- 条件不成立时更新 `right = middle - 1`。
- 不需要特殊处理 `x == 1`；`x == 0` 也能由同一模板返回 `0`。
- 使用 `middle * middle` 比 `x / middle` 更直接，也避免了 `middle == 0` 时的除零问题。

## 复习记录

- 第一次：理解了二分答案范围，并完成“记录合法值后继续向右寻找最大值”的模板。
- 第二次：待复习。
- 第三次：待复习。
