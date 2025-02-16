---
title: 由LeetCode一题窥探GO程序的性能测试及性能优化
date: 2019-03-21 14:54:08
categories:
- LeetCode
tags: [Go, 性能测试, 性能优化]
---


![](http://s3.51cto.com/oss/201710/24/0e74679601d45a92e0a5b55934cfd47b.jpeg-wh_651x-s_2760864022.jpeg)
<!-- more -->
本文将通过一道LeetCode的题目介绍GO语言程序的性能测试及性能优化方法。  
题目是[无重复字符的最长子串](https://leetcode-cn.com/problems/longest-substring-without-repeating-characters/)

### LeetCode题目

#### 题目描述
给定一个字符串，请你找出其中不含有重复字符的 最长子串 的长度。

示例 1:

输入: "abcabcbb"
输出: 3 
解释: 因为无重复字符的最长子串是 "abc"，所以其长度为 3。
示例 2:

输入: "bbbbb"
输出: 1
解释: 因为无重复字符的最长子串是 "b"，所以其长度为 1。
示例 3:

输入: "pwwkew"
输出: 3
解释: 因为无重复字符的最长子串是 "wke"，所以其长度为 3。
     请注意，你的答案必须是 子串 的长度，"pwke" 是一个子序列，不是子串。

本题有多种解法：题解请参考:  
[无重复字符的最长子串题解](https://leetcode-cn.com/problems/longest-substring-without-repeating-characters/solution/)  
本文实现参考第三种解法


### GO程序性能测试
假设go程序实现如下：
```go
// longestNotRpeatedSubstring project main.go
package main

import (
	"fmt"
)


func longestNotRpeatedSubstring(s string) int {
	lastOccurred := make(map[byte]int)

	start := 0
	maxLength := 0
	for i, ch := range []byte(s) {
		//lastI, ok := lastOccurred[ch]
		lastI := lastOccurred[ch]
		if lastI >= start {
			start = lastI + 1
		}
		if i-start+1 > maxLength {
			maxLength = i - start + 1
		}
		lastOccurred[ch] = i
	}

	return maxLength
}

func main() {
	fmt.Println(longestNotRpeatedSubstring("abcadefg"))
	fmt.Println(longestNotRpeatedSubstring("abcabcbb"))
}

```
注意上面的程序不支持中文字符，其实可以稍微修改让程序支持中文字符，比如使用rune类型，这是GO语言为了用于支持各种语言而设定的类型。
golang中string底层是通过byte数组实现的。中文字符在unicode下占2个字节，在utf-8编码下占3个字节
* byte 等同于int8，常用来处理ascii字符
* rune 等同于int32,常用来处理unicode或utf-8字符  

通用版解法：
```go
package main

import (
	"fmt"
)

func longestNotRpeatedSubstring(s string) int {
	lastOccurred := make(map[rune]int)

	start := 0
	maxLength := 0
	for i, ch := range []rune(s) {
		lastI, ok := lastOccurred[ch]
		if ok && lastI >= start {
			start = lastI + 1
		}
		if i-start+1 > maxLength {
			maxLength = i - start + 1
		}
		lastOccurred[ch] = i
	}

	return maxLength
}

func main() {
	fmt.Println(longestNotRpeatedSubstring("abcadefg"))
	fmt.Println(longestNotRpeatedSubstring("abcabcbb"))
	fmt.Println(longestNotRpeatedSubstring("黑化肥发灰挥发会花飞灰化肥挥发发黑会飞花"))
}
```
GO语言由于提供了一系列测试相关的工具，所以其性能测试做起来相对其他语言而言来说是相当的方便。
另外命名一个文件：比如longestNotRpeatedSubstring_test.go  
**注意必须要以xxxx_test.go的方式命名**  
longestNotRpeatedSubstring_test.go内容如下
```GO
func BenchmarkSubstring(b *testing.B) {
	s, ans := "灰化肥挥发发黑会飞花", 9
	for i := 0; i < 11; i++ {
		s = s + s
	}
	b.Logf("len(s) = %d", len(s))
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		actual := longestNotRpeatedSubstring(s)
		if actual != ans {
			b.Errorf("got %d for input %s; "+
				"expected %d", actual, s, ans)
		}
	}
}
```


做性能测试命令如下：
```shell
go test -bench .
go test -bench . -cpuprofile cpu.out
go tool pprof cpu.out
```
执行效果如下图：
![](http://wx2.sinaimg.cn/mw690/71c65545ly1g1aevh6j5qj20jh0dv0t7.jpg)
go test -bench . -cpuprofile cpu.out生成的cpu.out文件是二进制文件，需要借助go的pprof工具进行查看，然后使用go tool pprof cpu.out命令进入到交互式命令行，可以敲入help进行查看，执行效果如下图：
![](http://wx3.sinaimg.cn/mw690/71c65545ly1g1aevhqr4dj20j70ewq3j.jpg)

这里使用简单的办法：直接敲入web，也就是通过web server查看可视化的图  
网页中显示效果如下：
![](http://wx4.sinaimg.cn/mw690/71c65545ly1g1aez6u4b0j20r90ntmyd.jpg)

**注意：这里需要安装graphviz工具，可以到官网下载安装，Windows安装之后记得把软件的bin目录添加到环境变量中去，然后重新打开命令窗口即可**
具体步骤可参考：
![](https://img-blog.csdn.net/20160628204148230)


### GO程序性能优化
![](http://wx3.sinaimg.cn/mw690/71c65545ly1g1afusfykej20kh0gkq3l.jpg)
通过上面的步骤，可以看到上面给出解法，主要耗时在map的访问  
那我们可以想办法把这块时间优化掉，通常这采用的办法是：**用空间换时间**
我们可以选择不用map，那么可以将程序改成：
```GO
package main

import (
	"fmt"
)

func longestNotRpeatedSubstring(s string) int {
	lastOccurred := make([]int, 0xffff)

	for i := range lastOccurred {
		lastOccurred[i] = -1
	}
	start := 0
	maxLength := 0
	for i, ch := range []rune(s) {
		lastI := lastOccurred[ch]
		if lastI >= start {
			start = lastI + 1
		}
		if i-start+1 > maxLength {
			maxLength = i - start + 1
		}
		lastOccurred[ch] = i
	}

	return maxLength
}

func main() {
	fmt.Println(longestNotRpeatedSubstring("abcadefg"))
	fmt.Println(longestNotRpeatedSubstring("abcabcbb"))
	fmt.Println(longestNotRpeatedSubstring("黑化肥发灰挥发会花飞灰化肥挥发发黑会飞花"))
}
```
再次执行：
```shell
go test -bench . -cpuprofile cpu.out
go tool pprof cpu.out
```
![](http://wx2.sinaimg.cn/mw690/71c65545ly1g1ag9vepk5j20ib0903yp.jpg)
可以看到优化后的代码每次操作，也就是每次调用我们的函数耗时已经降为4.92ms，原来的时间是9.19ms（可以看上面的截图）可以得知此次优化还是非常好的，我们还可以使用web通过web server查看耗时情况：
![](http://wx3.sinaimg.cn/mw690/71c65545ly1g1agfzg1dtj20mq0n7t99.jpg)


通过上面优化的示例，我们就可以看到优化的过程如下  
![](http://wx3.sinaimg.cn/mw690/71c65545ly1g1agtbsrkpj20h409cmyv.jpg) 


当然上面的程序还可以做一点优化，那就是把slice移到外面去，不用每次都去重新分配一个slice
```GO
package main

import (
	"fmt"
)

var lastOccurred = make([]int, 0xffff)

func longestNotRpeatedSubstring(s string) int {

	for i := range lastOccurred {
		lastOccurred[i] = -1
	}
	start := 0
	maxLength := 0
	for i, ch := range []rune(s) {
		lastI := lastOccurred[ch]
		if lastI >= start {
			start = lastI + 1
		}
		if i-start+1 > maxLength {
			maxLength = i - start + 1
		}
		lastOccurred[ch] = i
	}

	return maxLength
}

func main() {
	fmt.Println(longestNotRpeatedSubstring("abcadefg"))
	fmt.Println(longestNotRpeatedSubstring("abcabcbb"))
	fmt.Println(longestNotRpeatedSubstring("黑化肥发灰挥发会花飞灰化肥挥发发黑会飞花"))
}
```

### GO程序普通测试的方法

编写测试用例，GO语言中主要采用表格测试的方法，将测试数据，与真正的测试逻辑分开

如果需要做普通的测试也可以在上述性能测试文件longestNotRpeatedSubstring_test.go里面添加如下函数：    
```GO
func TestSubstr(t *testing.T) {
	tests := []struct {
		s   string
		ans int
	}{
		{"abcacbcc", 3},
		{"pwwkew", 3},

		{"", 0},
		{"b", 1},
		{"abcabcabcd", 4},
		{"灰化肥挥发发黑会飞花", 5},
	}

	for _, tt := range tests {
		actual := longestNotRpeatedSubstring(tt.s)
		if actual != tt.ans {
			t.Errorf("got %d for input %s; expected %d",
				actual, tt.s, tt.ans)
		}
	}
}
```

测试主要使用如下命令
```shell
go test .
```
此外使用GO自带的工具还可以查看代码覆盖率，非常方便：
```shell
go test -coverprofile=c.out
go tool cover -html=c.out
```

效果如下：
![](http://wx3.sinaimg.cn/mw690/71c65545ly1g1agzv7ii2j20tw0itaad.jpg)

### C语言实现
下面补充本题的C语言实现，仅供参考
```c
int checkExist(char *s, int start, int end, char target)
{
    for(int i = start; i < end; i++)
    {
        if(s[i] == target)
        {
            return i;
        }
    }
    
    return -1;
}

int lengthOfLongestSubstring(char* s) {
    if(!s)
    {
        return 0;
    }
    
    int start = 0;
    int maxlen = 0;
    int repeatPosition = -1;
    int cur = 0;
    char *p = s;
    while(*p != '\0')
    {
        repeatPosition = checkExist(s, start, cur, *p);
        if(repeatPosition >= start)
        {
            start = repeatPosition + 1;
        }
        
        if(cur - start + 1 > maxlen)
        {
            maxlen = cur - start + 1;
        }
        
        p++;
        cur++;
    }
    
    return maxlen;
}
```




### 参考资料：
[无重复字符的最长子串](https://leetcode-cn.com/problems/longest-substring-without-repeating-characters/)  
[无重复字符的最长子串题解](https://leetcode-cn.com/problems/longest-substring-without-repeating-characters/solution/)

