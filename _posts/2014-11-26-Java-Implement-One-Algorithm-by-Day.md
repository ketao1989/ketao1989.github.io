---
layout: post
category: Java
title:  "一天一算法之java实现"
date: 2014-11-26 22:21:35 +0800
comments: true
---

说明：算法背景和解法来自<https://github.com/ketao1989/The-Art-Of-Programming-By-July.git>,如有版权问题，请留言告知！

## <a id="Intro">前言</a>

最近看博客，发现一些有趣的算法，很久没接触，都不清楚了。这里，把上述的github中一些算法用java实现。

## <a id="RotateString">旋转字符串</a>

### 背景

给定一个字符串，要求把字符串前面的若干个字符移动到字符串的尾部，如把字符串“abcddg”前面的2个字符'a'和'b'移动到字符串的尾部，使得原字符串变成字符串“cdcgab”。请写一个函数完成此功能，要求对长度为n的字符串操作的时间复杂度为 O(n)，空间复杂度为 O(1)。

### Java 代码

``` java

/*************************************************************************
	> File Name: RotateString.java
	> Author: ketao
	> Mail: ketao1989@gmail.com
	> Created Time: 2014年11月26日 星期三 16时33分28秒
 ************************************************************************/
public class RotateString{
    
    /**
     * 把一个字符数组，按照指定位置移动
     */

    public static void rotateString(char[] str,int n,int len){
        reverseString(str,0,n-1);
        reverseString(str,n,len-1);
        reverseString(str,0,len-1);
    }
    /**
     *反转指定区间的字符数组
     *
     */
    private static void reverseString(char[] str,int from,int to){
        
        while(from < to){
            char ch = str[from];
            str[from++] = str[to];
            str[to--] = ch;
        }
    }

    /**
     *测试代码
     */
    public static void main(String[] args){
        char[] str = {'a','b','c','d','c','g'};
        rotateString(str,2,6);
        System.out.println(str);
    }
}

```

<!-- more -->

## <a id="StringContain">字符包含</a>

### 背景

给定两个分别由字母组成的字符串A和字符串B，字符串B的长度比字符串A短。请问，如何最快地判断字符串B中所有字母是否都在字符串A里？简单起见，暂不考虑非字符符号。

### Java 代码

``` java

/*************************************************************************
	> File Name: StringContain.java
	> Author: ketao
	> Mail: ketao1989@gmail.com
	> Created Time: 2014年11月26日 星期三 21时16分42秒
 ************************************************************************/

public class StringContain{

    public static boolean isContainString(String longger,String shortter) throws Exception{
        
        //26*2个字符，使用long就可以了
        long hash = 0L;

        for(int i =0;i<longger.length();i++){
            hash |= (1L << computeLetterBit(longger.charAt(i)));
        }

        for(int i =0; i<shortter.length();i++){

            //如果&操作为0，表示该位对应的字符，longger没有包含
            if((hash & (1L << computeLetterBit(shortter.charAt(i)))) == 0){
                return false ;
            }
        }

        return true;

    }

    private static int computeLetterBit(char ch){
        if(ch >= 'A' && ch <= 'Z'){
            return (ch-'A');
        }

        if(ch >='a' && ch <= 'z'){
            return (ch-'a'+26);
        }

        throw new IllegalArgumentException("参数错误！非法字符！");
    }

    public static void main(String[] args) throws Exception{
        String a = "AcBbDg";
        String b = "AcD";
        String c = "DdD";
        String d ="*AcN";

        if(isContainString(a,b)){
            System.out.println("a 包含b所有的字符！！");
        }else{
            System.out.println("a 不包含b所有的字符！！！");
        }

        if(isContainString(a,c)){
            System.out.println("a 包含c所有的字符！！");
        }else{
            System.out.println("a 不包含c所有的字符！！");
        }

        if(isContainString(a,d)){
            System.out.println("a 包含d所有的字符！");
        }

    }
}


```

## <a id="PalindromeString">回文判断</a>

### 背景

回文，英文palindrome，指一个顺着读和反过来读都一样的字符串，比如madam、我爱我，这样的短句在智力性、趣味性和艺术性上都颇有特色，中国历史上还有很多有趣的回文诗。

那么，我们的第一个问题就是：判断一个字串是否是回文？


### Java代码

``` java


/*************************************************************************
    > File Name: PalindromeString.java
    > Author: ketao
    > Mail: ketao1989@gmail.com
    > Created Time: 2014年11月27日 星期四 17时03分52秒
 ************************************************************************/

public class PalindromeString{

    public static boolean isPalindrome(final char[] palindromeString,final int len){

        if( null  == palindromeString || len <1){
            return false;
        }

        int middle = middlePos(len);

        int before = middle -1;

        int behind = (len % 2 == 0) ? middle : (middle + 1);

        while(before >= 0){
            System.out.println(palindromeString[before]+"----"+palindromeString[behind]);
            if(palindromeString[before--] != palindromeString[behind++]){
                return false;
            }
        }
        return true;


    }

    private static int middlePos(final int num ){
        
        int middle = (num >> 1);

        return middle > 0 ? middle : 0;
    }

    public static void main(String[] args){P
        char[] palindromeString1 = {'a','b','c','c','b','a'};
        char[] palindromeString2 = {'a','b','c','b','d'};

        if(isPalindrome(palindromeString1,6)){
            System.out.println("palindromeString1 is PalindromeString");
        }else{
            System.out.println("palindromeString1 is not PalindromeString");
        }

        if(isPalindrome(palindromeString2,5)){
            System.out.println("palindromeString2 is PalindromeString");
        }else{
            System.out.println("palindromeString2 is not PalindromeString");
        }

    }
}
```


## <a id="LongestPalindromeSubString">最长回文子串</a>

### 背景

上一小节已经介绍了回文字符串，不再做介绍。最长回文子串，就是：给定一个字符串，求它的最长回文串的长度，并且把这个回文子串打印出来。

### 算法分析

由于查找最长回文子串问题的解决方法有很多种，而最快捷高效的算法，理解起来比较复杂，这里介绍一下，所谓的时间空间复杂度都是O(N)的`Manacher`算法.

* `Manacher`算法，首先会初始化处理字符串。在上一节的时候我们会需要针对奇数偶数不同情况来不同处理，但是如果按照`Manacher`算法的初始化规则，就可以统一为奇数一种情况来处理，方便快捷。

初始化规则就是：在字符串中每个字符前后都是用`#`来分隔，包括开头和结尾；然后再新字符串的前后分别添加`$`和`^`符号标识字符串开始和结束。例如给定一个字符串`abcdcda`，初始化处理之后为`$#a#b#c#d#c#d#a#^`

* 接下来，就需要对每一个字符分别计算它对应的最大回文子串长度(包括字符本身)，例如，
<table>
    <tr>
        <td>S[i]</td>
        <td>`$#a#b#c#d#c#d#a#^`</td>
    </tr>
    <tr>
        <td>P[i]</td>
        <td>`01212121414121210`</td>
    </tr>
</table>

这样子计算的P[i]对应的值和初始化前字符的回文长度关系是：`old[i] = P[i]-1`.

*重点来了，如果分别计算每一个长度，那么其实效率并不高，那么有没有什么特征能快速计算每个字符对应的最长回文长度呢？`Manacher`算法告诉我们是有的。

假设：id是当前求得的最长回文子串中心的位置，mx为当前最长回文子串的右边界（回文子串不包括该右边界），即mx = id + P[id]。记j = id - (i-id) = 2*id – i ，即 j 是 i 关于 id 的对称点。

1. 当i `<` mx 时；存在一个非常神奇的结论P[i] >= min(P[2*id - i], mx - i)，证明略。
2. 当i >= mx, 无法对P[i]做更多的假设，只能P[i] = 1,然后按照原始方法再去匹配计算长度。


