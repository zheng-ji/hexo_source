---
title: NLP - 最小编辑距离
date: 2018-09-08 10:11:42
tags:
categories: NLP
---

最近工作中，需要求两个单词`最小编辑距离`，用于判断两个字符串的相近程度。
* [最小编辑距离是什么](#第一节)
* [解决思路](#第二节)
* [代码演示](#第三节)

### 最小编辑距离是什么

允许一个字符串 A ，通过以下三种操作，变换成另一个字符串 B
插入一个字符，例如：ab -> abc
删除一个字符，例如：abc -> ab
替换一个字符，例如：abc -> dbc
所需要的最短操作次数，称为最短编辑距离 D，又称 `Levenshtein 距离`
故两个字符串的相识度 S = D / Max(len(A)， len(B))
使用场景：搜索引擎的单词纠错，扩召回等。

### 解决思路

我发现这个问题，其实也是[LeetCode](https://leetcode.com/problems/edit-distance/)的一道题目， 以前有些枯燥的算法题，应用在工程领域上会显得如此可爱迷人。要解决这个问题，暴力枚举很快会让人绝望。直觉告诉我们要分而治之， 将复杂的问题分解为相似的子问题。

假设字符串 A 共 m 位，从 A[1] 到 A[m]；字符串 B 共 n 位，从 B[1] 到 B[n]；D[i][j] 表示字符串 A[1]-A[i] 转换为 B[1]-B[j] 的编辑距离。
* 规律一: 当 A[i] == B[j] 时，D[i][j] = D[i-1][j-1]，比如 hel -> hal 的编辑距离 = he -> ha 的编辑距离
* 规律二: 当 A[i] 不等于 B[j] 时，D[i][j] 等于如下 3 项的最小值:
    - D[i-1][j] + 1（删除 A[i]），比如 xyz  -> abc  的编辑距离 = xy -> abc 的编辑距离 + 1
    - D[i][j-1] + 1（插入 B[j])， 比如 xyz -> abc 的编辑距离 = xyzc -> abc 的编辑距离 + 1; 根据规律一， xyzc->abc 由于最后一个字符是一样的， 所以它的编辑距离等于 xyz->ab， 因此xyz -> abc 的编辑距离 = xyzc -> abc 的编辑距离 + 1 = xyz->ab 的编辑距离 + 1。
    - D[i-1][j-1] + 1（将 A[i] 替换为 B[j]）， 比如 xyz -> abc 的编辑距离 = xyc -> abc 的编辑距离 + 1 = xy -> ab 的编辑距离 + 1 (根据规律一)
* 边界：
    - A[i][0] = i， B 字符串为空，表示将 A[1]-A[i] 全部删除，所以编辑距离为 i
    - B[0][j] = j， A 字符串为空，表示 A 插入 B[1]-B[j]，所以编辑距离为 j
翻译成代码
```
int distance(char *a, char *b, int i, int j)
{
    if (j == 0) return i;
    else if (i == 0) return j;
    else if (a[i-1] == b[j-1]) 
        return edit_distance(a, b, i - 1, j - 1);
    else
        return Min3(edit_distance(a, b, i - 1, j) + 1，
                edit_distance(a, b, i, j - 1) + 1，
                edit_distance(a, b, i - 1, j - 1) + 1);
}
```

## 动态规划

递归被人诟病的就是性能不行，时间复杂度是指数增长的，很多相同的子问题其实是经过了多次求解，用动态规划可以解决这类问题。 而此问题需用一个二维数组来记忆状态图，俗称打表。
翻译成代码如下
```
int distance(char *a, char *b)
{
    int lena = strlen(a);
    int lenb = strlen(b);
    int d[lena+1][lenb+1];
    int i, j;
    for (i = 0; i <= lena; i++)
        d[i][0] = i;
    for (j = 0; j <= lenb; j++)
        d[0][j] = j;
    for (i = 1; i <= lena; i++) 
    {
        for (j = 1; j <= lenb; j++) 
        {
            if (a[i-1] == b[j-1]) 
            {
                d[i][j] = d[i-1][j-1];
            } 
            else 
            {
                d[i][j] = min_of_three(d[i-1][j]+1, d[i][j-1]+1, d[i-1][j-1]+1);
            }
        }
    }
    return d[lena][lenb];
}
```
