# 删除有序数组中的重复项 II

LeetCode 80。

## 题意

给定升序数组，原地修改数组，使每个数字最多出现两次，返回保留后的元素数量 `k`。

## 我的思路

保留前两个元素后，使用同向快慢指针。

- `fast` 遍历尚未处理的元素。
- `slow` 表示已保留的元素数量，也表示下一个可写入的位置。
- 当前 `nums[fast]` 与 `nums[slow - 2]` 不同时，才可以写入；这保证同一数字不会被保留超过两次。

## 标准模板

```python
if len(nums) <= 2:
    return len(nums)

slow = 2

for fast in range(2, len(nums)):
    if nums[fast] != nums[slow - 2]:
        nums[slow] = nums[fast]
        slow += 1

return slow
```

## 复杂度

- 时间复杂度：O(n)。
- 空间复杂度：O(1)。

## 踩坑点

- 长度不超过 2 时，直接返回原长度。
- 此处的 `slow` 是下一个写入位置，不是最后一个有效元素的下标。
- 比较对象是 `nums[slow - 2]`，不是 `nums[slow]`。

## 复习记录

- 第一次：提示后能独立写出核心循环。
- 第二次：待复习。
- 第三次：待复习。
