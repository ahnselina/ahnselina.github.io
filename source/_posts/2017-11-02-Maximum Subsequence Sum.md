---
layout: post
title: Maximum Subsequence Sum
categories: 练习题
tags: [数据结构,算法,练习]

---

本题为上一篇文章中最大子列和的改进版题目，要求在输出最大子列和的同时输出最大子列的第一个元素和最后一个元素，具体要求见题目。

***

### Maximum Subsequence Sum
Given a sequence of K integers { N​1, N​2, ..., N​K}. A continuous subsequence is defined to be { N​i, Ni+1, ..., N​j} where 1≤i≤j≤K. The Maximum Subsequence is the continuous subsequence which has the largest sum of its elements. For example, given sequence { -2, 11, -4, 13, -5, -2 }, its maximum subsequence is { 11, -4, 13 } with the largest sum being 20.
Now you are supposed to find the largest sum, together with the first and the last numbers of the maximum subsequence.

#### Input Specification:

Each input file contains one test case. Each case occupies two lines. The first line contains a positive integer K (≤10000). The second line contains K numbers, separated by a space.

#### Output Specification:

For each test case, output in one line the largest sum, together with the first and the last numbers of the maximum subsequence. The numbers must be separated by one space, but there must be no extra space at the end of a line. In case that the maximum subsequence is not unique, output the one with the smallest indices i and j (as shown by the sample case). If all the K numbers are negative, then its maximum sum is defined to be 0, and you are supposed to output the first and the last numbers of the whole sequence.

#### Sample Input:

10
-10 1 2 3 4 -5 -23 3 7 -21

#### Sample Output:

10 1 4

时间限制: 200ms
内存限制: 64MB
代码长度限制: 16KB


### 思路分析：

题目要求大致就是求出给出数列的最大子列和，同时给出所求子列的第一项和最后一项。对于全为负数的数列，最大子列和为0，并且输出整个数列的第一项和最后一项。需要注意的时候是，最大子列和有多个的时候，输出序号最小的第一个元素和最后一个元素。如题目输出的是10 1 4，而非10 3 7。

```c
#include <stdio.h>

void max_subsquence_sum(int n);

int main(void)
{
    int n;
    scanf("%d", &n);
    max_subsquence_sum(n);

    return 0;
}

void max_subsquence_sum(int N)
{
    int i, this_sum, max_sum, flag, start1, start, end;
    flag = start1= start= end = 0;
    this_sum = max_sum = 0;
    int a[N];
    for(i = 0; i < N; i++)
    {
        scanf("%d", &a[i]);
    }

    for(i=0; i < N; i++)
    {
        if(a[i] >= 0)
        {
            flag = 1;//标记，如果有非负数，flag就置为1
        }
        this_sum += a[i];
        if(this_sum > max_sum)
        {
            start = start1;
            max_sum = this_sum;
            end = i;
        }
        else if(this_sum < 0)
        {
            this_sum = 0;
            start1 = i + 1;
        }
    }

    if (0 == max_sum)
    {
        if(0 == flag)
        {
            printf("%d %d %d\n", max_sum, a[0], a[N - 1]);
        }
        else
        {
            printf("0 0 0");
        }
    }
    else
    {
        printf("%d %d %d\n", max_sum, a[start], a[end]);
    }

    return;
}

```

```c
#include <stdio.h>

int main(void)
{
    int n, flag = 0, max_sum = -1, this_sum = 0, first = 0, last = 0, index = 0;
    scanf("%d", &n);
    int array[n];
    int i=0;
    for(; i < n; i++)
    {
        scanf("%d", &array[i]);
        if (array[i] >= 0)
        {
            flag = 1;
        }
        this_sum += array[i];
        if(this_sum > max_sum)
        {
            max_sum = this_sum;
            first = index;
            last = i;
        }
        else if(this_sum < 0)
        {
            this_sum = 0;
            index = i + 1;
        }
    }

    if (0 == flag)
    {
        printf("0 %d %d\n", array[0], array[n-1]);
    }
    else
    {
        printf("%d %d %d\n", max_sum, array[first], array[last]);
    }

    return 0;
}

```

### 思考
如果该题目不要求输出最小的最大子列和的第一个元素和最后一个元素，反而要求输出最大的最大子列和的第一个元素和最后一个元素呢？


### 参考资料

[浙江大学PTA]
