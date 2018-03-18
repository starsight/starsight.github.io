---
layout: post
title: "二进制中0的个数"
date: 2018-03-18 14:26:53 +0800
comments: true
categories: 2018-03
tags: [剑指Offer]
---
师兄的一道网易面试题
问题：求二进制0的个数，要求从第一位为1往后统计个数?

#### 方法一
主要考虑正负数情况。<!--more-->
对于正数，在不为0的情况下，计算其二进制，容易得0的个数。
对于负数，因为已知第一位是1的，所以不用担心有高位0，直接取1，然后依次左移，看&原值是不是0。停止条件是这个数为0。

```java
public static int numberOf0(int num){  
    int count=0;  
    if(num<0){  
        int i=1;  
        while(i!=0){  
            if((i&num)==0){  
                count++;  
            }  
            i=i<<1;  
        }  
        return  count;  
    }  
  
    while(num!=0){  
        int mod = num%2;  
        if(mod==0)  
            count++;  
        num = num/2;  
    }  
    return count;  
}  
```

#### 方法二
另一种技巧性强的方法。从低位开始，遇到0则正常计数，遇到1则需要判断，在更高位是否还有1，没有则可以跳出循环。

判断方法在于：

先记录左移次数，在当前次数下的次（第二）大值，如左移3次，为0b1000，次大值为0b0111；

把原数当前为1的位变成0，再比较这个数和次大值，如果高位没有1，则这个数不会超过次大值，这时就可以跳出循环了，注意绝对值。
```java
public static int NumberOf0(int num){  
    int count =0;  
  
    int flag0 = 0;  
    int flag1 = 1;  
  
    int movecount =0;  
    int max =0;  
  
    for(;;){  
        if((num&flag1)==0){  
            count++;  
        }else{  
            flag0 =~flag1;  
            max = 1<<movecount;//构造次大值  
            max--;  
            if(Math.abs(num&flag0)<=max)//不要忘了绝对值  
                break;  
        }  
        flag1 = flag1<<1;  
        movecount++;  
    }  
  
    return count;  
}  
```
