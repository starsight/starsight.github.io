---
layout: post
title: "LeetCode【82. Remove Duplicates from Sorted List II】"
date: 2018-01-11 11:17:02 +0800
comments: true
categories: 2018-01
tags: [LeetCode,链表]
---
本文的题目是一道中等难度题（82题），之前没有注意删除头结点时的技巧。<!--more-->

#### 题目内容
Given a sorted linked list, delete all nodes that have duplicate numbers, leaving only distinct numbers from the original list.

For example,
Given 1->2->3->3->4->4->5, return 1->2->5.
Given 1->1->1->2->3, return 2->3.

#### 思路
这道题在做了好久，开始希望先保证头结点唯一，但后来继续实现发现，保证头结点唯一和保证其他结点唯一方法类似。看各种判断语句感觉这代码不能这么写啊！
然后去百度，看到有人用添加头结点的方式构造。学了这一手，然后自己写了代码。代码有什么不懂的可以留言讨论。

#### 实现
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
    public ListNode deleteDuplicates(ListNode head) {  
        if(head==null||head.next==null)  
            return head;  
      
        ListNode cur =head;  
        ListNode newHead =new ListNode(head.next.val);//小技巧，第一个结点，让左边等于右边咯 
        ListNode pre =newHead;//当前结点的前一个结点  
        ListNode preSuccess =newHead;//前一个唯一结点  
        pre.next =head;//连接虚拟头结点到链表  
          
        while(cur!=null&&cur.next!=null){  
            if(pre.val==cur.val||cur.val==cur.next.val){  
                pre =cur;  
                cur =cur.next;  
                if(cur!=null&&cur.next==null){//只剩下最后一个（如果倒数第二个已经加入链表，那么最后一个也默认加入了）  
                    if(pre.val!=cur.val)//pre和preSuccess不一定相等哦 preSuccess不一定连接着cur  
                        preSuccess.next =cur;  
                    else  
                        preSuccess.next=null;  
                }  
            }else{  
                preSuccess.next =cur;  
                preSuccess =cur;  
                pre =cur;  
                cur =cur.next;           
            }     
        }  
  
        return newHead.next;  
    }  
  
} 
```


#### 参考链接
原题链接：[82. Remove Duplicates from Sorted List II](https://leetcode.com/problems/remove-duplicates-from-sorted-list-ii/description/)  
参考思路：[Remove Duplicates from Sorted List II -- LeetCode](http://blog.csdn.net/linhuanmars/article/details/24389429)
