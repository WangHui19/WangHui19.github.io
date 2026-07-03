---
title: q2-两数相加
date: 2022-10-27 10:55:22
tags: LeetCode
categories: 
    - LeetCode 热题 HOT 100
    - 中等题
---
原题链接：https://leetcode.cn/problems/add-two-numbers/

### 问题描述
给你两个 **非空** 的链表，表示两个非负的整数。它们每位数字都是按照 **逆序** 的方式存储的，并且每个节点只能存储 **一位** 数字。

请你将两个数相加，并以相同形式返回一个表示和的链表。

你可以假设除了数字 0 之外，这两个数都不会以 0 开头。
<!--more-->
#### 示例1：
![图](/img/addtwonumber1.jpg)

    输入：l1 = [2,4,3], l2 = [5,6,4]
    输出：[7,0,8]
    解释：342 + 465 = 807.
#### 示例2:
    输入：l1 = [0], l2 = [0]
    输出：[0]
#### 示例3：
    输入：l1 = [9,9,9,9,9,9,9], l2 = [9,9,9,9]
    输出：[8,9,9,9,0,0,0,1]

### 思路
这里可以同时遍历两个链表的节点n1,n2，逐一计算它们的和，然后与当前位置的进位相加,n1+n2+carry。然后对应位置的值为当前和取余10，(n1+n2+carry) mod 10。下一个位置的进位是当前和/10，(n1+n2+carry) / 10。

### 代码（Java）
```
/**
 * Definition for singly-linked list.
 * public class ListNode {
 *     int val;
 *     ListNode next;
 *     ListNode() {}
 *     ListNode(int val) { this.val = val; }
 *     ListNode(int val, ListNode next) { this.val = val; this.next = next; }
 * }
 */
class Solution {
    public ListNode addTwoNumbers(ListNode l1, ListNode l2) {
        ListNode p1 = l1, p2 = l2;
        // 设置虚拟头结点便于添加第一个数字节点
        ListNode dummy = new ListNode(-1);
        ListNode p = dummy;
        int carry = 0;  // 进位
        while(p1 != null || p2 != null || carry != 0) {
            int val = carry;
            if(p1 != null) {
                val += p1.val;
                p1 = p1.next;
            }
            if(p2 != null) {
                val += p2.val;
                p2 = p2.next;
            }
            p.next = new ListNode(val % 10);
            p = p.next;
            carry = val / 10;
        }
        return dummy.next;
    }
}
```
### 复杂度分析
* 时间复杂度：O(max(m,n))，其中 mm 和 nn 分别为两个链表的长度。对于每一个元素 x ，采用O(1)的时间计算数值。
* 空间复杂度：O(1)。返回值不计入空间复杂度。