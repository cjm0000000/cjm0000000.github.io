---
layout: post
title: "Java字符串搜索学习（String）"
description: "学习String中的几种字符串搜索算法"
category: Java
tags: [Algorithm]
---
{% include JB/setup %}


今天抽空研究了String的源码。
<?prettify linenums=1?>
    int indexOf(String str);
	
核心查找代码如下：
<?prettify linenums=1?>
    for (int i = sourceOffset + fromIndex; i <= max; i++) {  
		/* Look for first character. */  
		if (source[i] != first) {  
			while (++i <= max && source[i] != first);  
		}  
	  
		/* Found first character, now look at the rest of v2 */  
		if (i <= max) {  
			int j = i + 1;  
			int end = j + targetCount - 1;  
			for (int k = targetOffset + 1; j < end && source[j]  
					== target[k]; j++, k++);  
	  
			if (j == end) {  
				/* Found whole string. */  
				return i - sourceOffset;  
			}  
		}  
	}
	
这段代码的简单思想是：从源字符串的索引`0`开始，按字符查找，直到最大索引`max`（max=源字符串长度 - 子字符串长度），找到和子串第一个字符（`first`）匹配的字符，记下索引位置并且存放到变量`i`中；
 
接下去判断i是否超出`max`，如果超出，则直接返回`-1`，查找结束；
 
如果`i`小于max，则源字符串索引`i`加1，子字符串索引`k`也加1，然后比对这两个索引对应的字符：
如果相等，两个字符串的索引再各自加1继续循环；
如果不相等，跳出循环；
 
继续执行下面的代码： 如果`j==end`意味着子串遍历完了（也就是在源字符串查找到了子串），返回查找到的子串索引，程序结束；
 
如果不满足`j==end`，则源字符串的索引`i++`，然后重新执行上面过程，直到源字符串遍历到最大索引`max`（这中间可能找到子串，提前返回）；
 
这种字符串查找的算法谈不上高效，但是对于小字符串来说应该够用了；
 
对于字符串查找场景较多，并且字符串很长的情况（可能需要查找的子串靠近源字符串的后半部分），这个方法可能效率不够高；

接着写 `int lastIndexOf(String str)`，先上源码：
<?prettify linenums=1?>
	int strLastIndex = targetOffset + targetCount - 1;  
	char strLastChar = target[strLastIndex];  
	int min = sourceOffset + targetCount - 1;  
	int i = min + fromIndex;  
	  
	startSearchForLastChar:  
	while (true) {  
		while (i >= min && source[i] != strLastChar) {  
			i--;  
		}  
		if (i < min) {  
			return -1;  
		}  
		int j = i - 1;  
		int start = j - (targetCount - 1);  
		int k = strLastIndex - 1;  
	  
		while (j > start) {  
			if (source[j--] != target[k--]) {  
				i--;  
				continue startSearchForLastChar;  
			}  
		}  
		return start - sourceOffset + 1;  
	}
	
查找的原理大致和`indexOf`方法相同，只不过`lastIndexOf`是从尾部开始查找的，并且中间的一些细节还是和`indexOf`方法不同的。
 
上面这段代码的主要意思是：
初始计算部分：
 计算子串的最后一个字符，存入变量`strLastChar`中；
计算源字符串查找的终点索引，存入变量`min`；
计算源字符串的查找起点索引，存入变量`i`；

进入第一个while循环：
<?prettify linenums=1?>
	while (i >= min && source[i] != strLastChar) {  
		 i--;  
	 } 
	 
这段代码是在源字符串找到和子串最后一个字符相等的字符，把他的索引存入变量`i`（从尾部开始查找）
 
如果索引`i`小余`min`，意味着已经不在上面规定的查找范围之内了，直接返回-1；
如果索引`i`合法，那么确定源字符串的查找范围（起点`j=i-1`，终点`start=起点-子串长度+1`）*源码中终点的变量是start，作者觉得这是查找的终点，请注意区分*
确定子串的查找起点`k=子串最后一个字符索引-1`
用源字符串的索引j处的字符和子串索引`k`处的字符比较，并且`j`和`k`各自减1：
如果字符相等，则继续执行上面的比较，直到索引`j`递减到`j>start`（*索引j超出了查找的终点start*），则跳出循环继续往下执行；
如果字符不相等，则程序跳转到锚点`startSearchForLastChar`处，按照上面的逻辑继续执行；
 
如果`if (source[j--] != target[k--])`的比对一直相等，并且把`while (j > start)`循环执行完了，意味着在源字符串找到了子串（只需要找到第一个就可以了，当然是从末尾开始找的），程序返回子串的索引。

对于很长的字符串查找的优化，推荐一篇文章： [KMP子字符串查找算法分析与实现](http://my.oschina.net/BreathL/blog/137916)。
 
全文完-