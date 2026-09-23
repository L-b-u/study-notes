# 删除链表的倒数第 N 个结点

LeetCode 19。

## 题意

给定链表头节点 `head` 和正整数 `n`，删除链表的倒数第 `n` 个节点，并返回链表头节点。

## 我的思路

使用虚拟头节点和快慢指针。

- `dummy.next = head`，让删除原头节点也能统一处理。
- `fast` 和 `slow` 都从 `dummy` 开始，先让 `fast` 前进 `n` 步。
- 然后两个指针同步前进，直到 `fast` 到达尾节点。
- 此时 `slow` 位于待删除节点的前一个位置，通过 `slow.next = slow.next.next` 删除目标节点。
- 返回 `dummy.next`。

## 标准模板

```python
dummy = ListNode(0)
dummy.next = head
fast = slow = dummy

for _ in range(n):
    fast = fast.next

while fast.next:
    fast = fast.next
    slow = slow.next

slow.next = slow.next.next
return dummy.next
```

## 复杂度

- 时间复杂度：O(n)，只遍历链表一次。
- 空间复杂度：O(1)。

## 踩坑点

- 两个指针从 `dummy` 开始，`fast` 先走 `n` 步；循环结束时 `fast` 在尾节点而非 `None`。
- 删除的是 `slow.next`，所以 `slow` 必须停在目标节点前一个位置。
- 使用 `dummy` 避免单独处理删除头节点的情况。
- 题目保证 `n` 合法，无需额外处理找不到目标节点的情况。

## 复习记录

- 第一次：独立写出虚拟头节点、先拉开 n 步间距、同步移动并删除目标节点的流程。
- 第二次：待复习。
- 第三次：待复习。
