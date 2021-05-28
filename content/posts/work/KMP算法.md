---
title: "KMP算法"
date: 2019-11-30T21:55:33+08:00
description: "KMP算法，是一种解决string.LastIndex(string)问题的优化算法，本文以形象易理解为目标，给你介绍它的基本原理。希望可以帮你更好地理解它。"
tags: ["算法"]
draft: false
---

问题描述为：

> 在字符串`P`(Primary)中找跟目标字符串`T`(Target)完全相同的子串(index)。

## 暴力解法

对于解决这种长字符串中包含小字符串的问题，写代码最简单的当然就是`暴力解法`：

遍历`P`的所有字符`P(i)`开头的字串是否跟`T`相等，逻辑清晰，代码简单，复杂度高(`O(m*n)`)。

暴力法的性能问题出在：每次匹配失败都返回两个字符串的头部（`P`返回`P(i+1)`的位置，`T`是返回`T(0)`）重新进行匹配。

```java
P:[ABCABCABDE]
T:[ABCABD]
   |
   P(i=0) == T(j=0), i++, j++
    |
    P(i=1) == T(j=1), i++, j++
     |
     P(i=2) == T(j=2), i++, j++
      .
      .
      .
        |至P(i=5) != T(j=5)匹配失败

// 出错时P和T都放弃已经获取的结果, 重置i=1, j=0, 重新匹配

P:[ABCABCABDE]
T:>[ABCABD]
    |
    P(i=1) == T(j=0)
```

但是实际上观察`T`可以发现，已经匹配过的局部字符串`ABCAB`中拥有一个最大的相同前缀和后缀`AB`，那么直观地将T的位置往右移动变为：

```java
P:[ABCAB|CABDE]
T:   [AB|CABD]
```

移动之后，显然可以忽略`AB`（因为AB是相同的前尾缀），然后继续P从`P(i=5)`，T从`T(i=2)`开始匹配，这显然比暴力法直接回退为i=1,j=0要高效。

## KMP原理

继续来看下一个例子，我修改了`P`，令`T`保持不变：

```java
P:[EFABCABCABDE]
T:[ABCABD]

// 首先很显然P开头两个字符EF都跟T(i=1)不匹配，向右移动T(本质上是增加P(i)的下标i)

P:[EFABCABCABDE]
T:  [ABCABD]

// 然后从P(i=2),T(j=0)继续匹配

P:[EFABCABCABDE]
T:  [ABCABD]
     |
     P(i=2) == T(j=0), i++, j++
      |
      P(i=3) == T(j=1), i++, j++
       |
       P(i=4) == T(j=2), i++, j++
      .
      .
      .
          |至P(i=7) != T(j=5)匹配失败          
```

和上一个例子一样，因为相同前后缀`AB`的存在，可以将T的位置右移变为:

```java
P:[EFABCAB|CABDE]
T:     [AB|CABD]
```

这样移动之后，继续P从`P(i=7)`，T从`T(j=2)`开始匹配！

给你三秒钟，观察两次的移动操作，可以发现什么规律么？

sleep(1)

sleep(2)

sleep(3)

### next函数 **!!划重点**

没错！就是两次移动之后，

> P都是从`P(i=i' | i'为上一次匹配的旧值)`继续匹配，即P未移动。
>
> T则都是从`T(j=2)`继续匹配, 而`2`恰恰是T中`已匹配的字符串`的`最大相同前后缀`：`AB`的长度！！

*这是其实一个普适的规律，可以在数学上证明。本文偏应用(数学渣)，有兴趣看数学推导的同学可以自行google*

这里我换个表述方式更方便你理解：

在P(i)==T(j)匹配失败的时候，

P继续匹配当前字符P(i)

T则需要回滚(下标j回滚)到T(n)继续匹配

- 其中n等于T已匹配过的`字符串T[0,j)`的`最大相同前后缀长度`，即n满足一个只跟j有关的函数，称之为`next(j)`
- 换言之，`next(j)`函数在T确定之后就得到确定，只需要计算一次！！

你可能对`最大相同前后缀长度`的概念有点迷惑，我再举几个例子:

- 字符串`ABD`的最大前后缀长度为`0`，没有相同的
- 字符串`ABA`的最大前后缀长度为`1`，有相同的`A`
- 字符串`ABCAB`的最大前后缀长度为`2`，有相同的`AB`
- 字符串`AAA`的最大前后缀长度为`1`，有相同的`A`(而不是AA或者AAA)
- 字符串`ABABA`的最大前后缀长度为`2`，有相同的`AB`(而不是ABA或ABABA)

而目标串`T`中，每个字符`T(j)`对应的已匹配过的子串`T[0,j)`都是随T固定的，因此每个字符上的`已匹配过的字符串的最大前后缀长度`在T上都是固定的。已匹配的串`T[0,j)`的最大前后缀并不会随着主串P而改变，只要T的检查位置来到j，它的最大前后缀(长度)就是一个确定的值。

还是举个例子，假设有字符串`T=[ABCEAB]`：

```java
j   T(j)   已匹配的串T[0, j)    最大前后缀(长度)
0   A      []                   无(0) "强制令其等于0，方便编程"
1   B      [A]                  [](0)
2   C      [AB]                 [](0)
3   E      [ABC]                [](0)
4   A      [ABCE]               [](0)
5   B      [ABCEA]              [A](1)

// PS. 为什么在已匹配的串T[0,j)为空时令最大前后缀长度等于0？ 
// 是因为将这个长度要代替next()作为匹配失败时T应该回滚的位置。在已匹配的串为空时，显然应该回到头部，所以令next(j=0)=0
```

于是获得T的`next(j)`函数:

```java
next(j)={
    0|j=0; 
    0|j=1; 
    0|j=2; 
    0|j=3; 
    0|j=4; 
    1|j=5;
}
```

### 计算next函数

现在你已经知道kmp算法的核心思路，最后剩下的只是如何根据一个给定的字符串T计算他的next(j)函数，通常使用一个数组来保存这个映射关系，因此求`next(j)`的方法可以被定义为:

```java
int[] getNext(String t){
    
}
```

事实上如果前面的逻辑你都清楚，那么这个方法的实现应该是水到渠成的，你可以试着先写一遍。再对比下面我给出的demo，看看跟你的实现有什么不同？（思路最重要，实现其次）

```java
func getNext(tArr []byte) []int {
    next := make([]int, len(tArr))
    next[0] = 0 // force 0 to start a check round at front of tArr
    if len(tArr)<=1 {
        return next // single byte
    }
    next[1] = 0 // set the first, than each othor depends on it
    for j := 2; j < len(tArr); j++ {
        if tArr[j-1] == tArr[next[j-1]] { // depends on the number of same bytes of the prefix and suffix 
            next[j] = next[j-1] + 1
        }
        // else default to zero
    }
    return next
}
```

最后，有了next(j)函数之后，依据上面的思路实现一个kmp算法相信也难不倒你。

```java
func KMP(p, t string) int {
    if len(t) < 1{
        return -1 // empty t return -1
    }
    tArr, pArr := []byte(t), []byte(p)
    i, j := 0, 0
    next := getNext(tArr)
    for ;i < len(pArr); {
        for j < len(tArr) && i < len(pArr) {
            if tArr[j] != pArr[i] {
                if j == 0 {
                    i++ // T(j) has reset to front, let P(i) move next
                    break
                }
                j = next[j]
                break
            }
            // P(i) = T(i), so both go to next
            i += 1
            j += 1
        }
        if j+1 >= len(tArr) { // reach end of j
            return i - j
        }
    }
    return -1
}
```

