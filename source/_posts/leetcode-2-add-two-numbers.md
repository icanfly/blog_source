title: leetcode算法题2_两数相加
date: 2017-09-07
tags:
 - leetcode
 - 算法
categories:
 - leetcode

---

给定两个非空的链表，表示两个非负整数。 数字以相反的顺序存储，每个节点包含一个数字。 添加两个数字并将其作为链表返回。

您可以假设两个数字不包含任何前导零，除了数字0本身。

<!-- more -->

思路：

参考常规的两数相加算式以及进位思想，两数相加与10相除得到该位相加后数值，两数相加与10取余得到该位相加后进位数。

方案：

```java
/**
 * Definition for singly-linked list.
 * public class ListNode {
 *     int val;
 *     ListNode next;
 *     ListNode(int x) { val = x; }
 * }
 */
class Solution {
    public ListNode addTwoNumbers(ListNode l1, ListNode l2) {

        ListNode root = null;
        ListNode parent = null;

        ListNode n1 = l1;
        ListNode n2 = l2;
        int val = 0;
        int carry = 0;

        while(true){
            if(n1 == null && n2 == null){
            	//在此情况下两个数各个位数都已经相加完成，只剩最后的进位数
                if(carry != 0){
                    ListNode child = new ListNode(carry);
                    parent.next = child;
                    parent = child;
                }
                break;
            }else if(n1 != null && n2 != null){
                val = n1.val + n2.val + carry;
                carry = val / 10;
                val = val % 10;
                n1 = n1.next;
                n2 = n2.next;
            }else if(n1 == null && n2 != null){
                val = n2.val + carry;
                carry = val / 10;
                val = val % 10;
                n1 = null;
                n2 = n2.next;
            }else if(n1 != null && n2 == null){
                val = n1.val + carry;
                carry = val / 10;
                val = val % 10;
                n1 = n1.next;
                n2 = null;
            }


            if(root == null){
                root = new ListNode(val);
                parent = root;
            }else{
                ListNode child = new ListNode(val);
                parent.next = child;
                parent = child;
            }
        }

        return root;
    }

}

```

原文地址：[2. Add Two Numbers](https://leetcode.com/problems/add-two-numbers/description/)
