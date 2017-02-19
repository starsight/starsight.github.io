---
layout: post
title: "Relative Ranks问题的两种实现"
date: 2017-02-19 16:17:02 +0800
comments: true
categories: 2017-02
tags: [LeetCode,排序]
---
本文的题目是一道基础题（506题），基于排序问题，借此熟悉一下快排和冒泡及其使用，同时也能很明显地看出快排的优势。<!--more-->

#### 题目内容
Description:
>Given scores of N athletes, find their relative ranks and the people with the top three highest scores, who will be awarded medals: "Gold Medal", "Silver Medal" and "Bronze Medal".

Example:
>Input: [5, 4, 3, 2, 1]
Output: ["Gold Medal", "Silver Medal", "Bronze Medal", "4", "5"]
Explanation: The first three athletes got the top three highest scores, so they got "Gold Medal", "Silver Medal" and "Bronze Medal". 
For the left two athletes, you just need to output their relative ranks according to their scores.

#### 我的思路
此问题基本就是个排序问题，当不能光排序，最后还需要给他们按未排序的数组分配名次。所以需要一个数组来建立**排序后的位置与排序前的位置关系**。即排序后的位置与原数组位置的信息丢失了，因此我们在排序的同时就需要保留这份信息。

#### 冒泡排序实现
```java
        String[] result = new String[nums.length]; //存储结果的字符串数组
        int [] numsTmp = new int[nums.length]; //因为排序直接对原来的数组操作，最后需要对排序后的数组分配奖牌及名次。如下标0，对应的值就是排序后nums[0]原来的位置

        for(int i=0;i<nums.length;i++) {//初始化次序
            numsTmp[i] = i;
        }

        int temp = 0;
        for(int i=0;i<nums.length;i++){//冒泡排序
            for(int j=0;j<nums.length-i-1;j++){
                if(nums[j+1]>nums[j]){
                    temp = nums[j];//冒泡中的交换
                    nums[j] = nums[j+1];
                    nums[j+1] = temp;
					
                    temp = numsTmp[j];//因为nums中交换了，对应地，numsTmp就要交换。
                    numsTmp[j] = numsTmp[j+1] ;//注意，这里交换的应该是numsTmp中的值
                    numsTmp[j+1] = temp;
                }
            }
        }

        

        for(int i=0;i<numsTmp.length;i++){//名次是从1开始的
            result[numsTmp[i]] = i+1+"";
        }

        if(result.length==1){//这里要注意，是不是可能人数少于3人
            result[numsTmp[0]] = "Gold Medal";
        }
        else if(result.length==2){
		    result[numsTmp[0]] = "Gold Medal";
            result[numsTmp[1]] = "Silver Medal";
        }else{
            result[numsTmp[0]] = "Gold Medal";			
            result[numsTmp[1]] = "Silver Medal";
            result[numsTmp[2]] = "Bronze Medal";
        }

        return result;
```

#### 快速排序实现
```java
public class Solution {
    public String[] findRelativeRanks(int[] nums) {
        int[] numsTmp = new int[nums.length];
        String[] result = new String[nums.length];
        
        for(int i=0;i<numsTmp.length;i++) {
            numsTmp[i] = i;
        }
        
        quickSort(nums,numsTmp,0,nums.length-1);//快排
        
        for(int i=0;i<numsTmp.length;i++){//同冒泡
            result[numsTmp[i]] = i+1+"";
        }

        if(result.length==1){
            result[numsTmp[0]] = "Gold Medal";
        }
        else if(result.length==2){
            result[numsTmp[0]] = "Gold Medal";			
            result[numsTmp[1]] = "Silver Medal";
        }else{
            result[numsTmp[0]] = "Gold Medal";			
            result[numsTmp[1]] = "Silver Medal";
            result[numsTmp[2]] = "Bronze Medal";
        }

        return result;
    }
    
    	public static void quickSort(int[] nums,int numsTmp[],int start,int end){

		if(start<end){
			int temp = nums[start];
			int cc = numsTmp[start];//记录下起始位置中的值
			int startTmp= start,endTmp = end;
			while(start<end) {
				while (start < end && nums[end] < temp) {
					--end;
				}
				if(start < end) {
				    //int Tmp = 0;
				    //Tmp = numsTmp[start];
				    numsTmp[start] = numsTmp[end];//把nums中更换时，同时也要更换numsTmp中
				    //numsTmp[end] = Tmp;
					nums[start++] = nums[end];
				}

				while (start < end && nums[start] >= temp) {
					++start;
				}
				if(start < end) {
				    numsTmp[end] = numsTmp[start];//把nums中更换时，同时也要更换numsTmp中
                    nums[end--] = nums[start];
                }

			}
			nums[start] = temp;//把最开始的值存在start位置，然后递归
			numsTmp[start] = cc;//把numsTmp的目前start在的位置赋值成初始值，对应于快排算法
			quickSort(nums,numsTmp,startTmp,start-1);
			quickSort(nums,numsTmp,start+1,endTmp);
		}
	}
}
```
再贴一张快排思路图，来自[pzhtpf](http://blog.csdn.net/pzhtpf/article/details/7560294)，但除了这幅图其他的讲的过于混乱。需要完整看快排算法可以看参考链接。

{% img /images/relative-ranks/quick_sort_example.jpg%} 

#### 实现时遇到的问题
1.排序算法不够熟练，所以一开始用冒泡做。
2.如何存储关系不明确，也就是思路不够明晰，其实只用把numsTMp中的内容做一个交换（冒泡）或者直接给新位置赋上旧位置的值（快排）即可。

#### 两种排序的运行时间对比
其实差距还是很明显的，快排大概能击败95%，冒泡则只有20%不到。

冒泡：
{% img /images/relative-ranks/bubble_sort.jpg%} 

快排：
{% img /images/relative-ranks/quick_sort.jpg%} 

#### 参考链接
原题链接：[relative-ranks](https://leetcode.com/problems/relative-ranks/?tab=Description)  
快排实现：[白话经典算法系列之六 快速排序 快速搞定](http://blog.csdn.net/morewindows/article/details/6684558)
