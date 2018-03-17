---
layout: post
title: "阿里笔试测试题（零售通App中使用消息数的红点引导用户点击）"
date: 2018-03-15 20:31:42 +0800
comments: true
categories: 2018-03
tags: [笔试题]
---
题目如下<!--more-->

{% img /images/interview/1.png%}

{% img /images/interview/2.png%}


这道题当时没做出来，一个是因为时间短，一个是因为思路太不成熟。
构建这个树比较麻烦，数据结构选好了其实就比较简单。
大致要求就是求得一级节点消息数的最大值。
构建这棵树，采用HashMap<String,Node>这样可以快速根据字符串找到对应地Node，同时拿一个ArrayList<Node>存储下根节点连接的一级节点，便于后期求一级节点。

```java
public class choiceRedDot {  
    /*** 
r->a,r->b,r->c,a->e,b->g,c->g,c->h,g->f 
e:15,f:10,h:6 
     */  
  
    static class Node{  
        String label;  
        int val=0;  
        ArrayList<Node> next;  
        Node(){  
            next = new ArrayList<>();  
        }  
    }  
  
    static String choiceRedDot(String nodePath, String nodeCount) {  
        String[] link = nodePath.split(",");  
        String[] count = nodeCount.split(",");  
  
        String[] temp;  
        //Node root =new Node();  
        Node node1 =null;  
        Node node2 =null;  
        HashMap<String,Node> hashMap = new HashMap<>();  
        ArrayList<Node> list =new ArrayList<>();  
        for (String aLink : link) {  
            temp = aLink.split("->");  
            if (hashMap.containsKey(temp[0])) {  
                node1 = hashMap.get(temp[0]);  
            } else {  
                node1 = new Node();  
                node1.label =temp[0];  
                hashMap.put(temp[0], node1);  
            }  
  
            if (hashMap.containsKey(temp[1])) {  
                node2 = hashMap.get(temp[1]);  
            } else {  
                node2 = new Node();  
                node2.label =temp[1];  
                hashMap.put(temp[1], node2);  
            }  
  
            node1.next.add(node2);  
  
            if (temp[0].equals("r")) {  
                list.add(node2);  
            }  
        }  
  
        for(String cCount :count){  
            temp = cCount.split(":");  
            node1 = hashMap.get(temp[0]);  
            node1.val = Integer.valueOf(temp[1]);  
        }  
  
        int max =-1;  
  
        for(Node node:list){  
            int sum =calculate(node);  
            if(max<sum){  
                max = sum;  
                node1 =node;  
            }  
        }  
        return node1.label;  
  
    }  
  
    public static int calculate(Node node){  
        ArrayList<Node> list = node.next;  
  
        if(node.val!=0)  
            return node.val;  
  
        int sum =node.val;  
        for (Node n: list) {  
            sum+=calculate(n);  
        }  
        return sum;  
    }  
  
    public static void main(String[] args){  
        Scanner in = new Scanner(System.in);  
        String res;  
  
        String _nodePath;  
        try {  
            _nodePath = in.nextLine();  
  
        } catch (Exception e) {  
            _nodePath = null;  
        }  
  
        String _nodeCount;  
        try {  
            _nodeCount = in.nextLine();  
        } catch (Exception e) {  
            _nodeCount = null;  
        }  
  
        res = choiceRedDot(_nodePath, _nodeCount);  
        System.out.println(res);  
    }  
}  
```

完整代码如上，只经过基本的case测试，大型case效率估计有问题，calculate做了一些避免重复计算的优化，但感觉这方案肯定不是最优设计。

欢迎大家交流~

