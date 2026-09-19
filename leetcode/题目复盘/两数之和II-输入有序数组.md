# 两数之和 II - 输入有序数组

LeetCode 167。

## 题意

给定升序整数数组 `numbers` 与 `target`，找出两个不同元素，使两数之和等于 `target`。返回从 1 开始的下标，题目保证答案唯一。

## 我的思路

数组有序，使用相向双指针：`left` 从开头、`right` 从末尾开始。

- 当前和等于目标，直接返回。
- 当前和过小，移动 `left` 以增大和。
- 当前和过大，移动 `right` 以减小和。

## 标准模板

```python
left = 0
right = len(numbers) - 1

while left < right:
    current_sum = numbers[left] + numbers[right]

    if current_sum == target:
        return [left + 1, right + 1]
    elif current_sum < target:
        left += 1
    else:
        right -= 1
```

## 复杂度

- 时间复杂度：O(n)。两个指针都只会单向移动。
- 空间复杂度：O(1)。

## 踩坑点

- 返回下标从 1 开始。
- `left < right` 保证两个下标不同。
- 当前和小于目标时，必须移动 `left`；当前和大于目标时，必须移动 `right`。

## 复习记录

- 第一次：独立写出并通过测试。
- 第二次：待复习。
- 第三次：待复习。
