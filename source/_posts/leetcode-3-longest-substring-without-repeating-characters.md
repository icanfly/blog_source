title: leetcode算法3_最长无重复子串长度
date: 2017-09-08
tags:
 - leetcode
 - 算法
categories:
 - leetcode

---

# 题目

>给定一个字符串，找到最长子串的长度，而不重复字符。

# 例子

给定“abcabcbb”，答案是“abc”，长度为3。

给定“bbbbb”，答案是“b”，长度为1。

给定“pwwkew”，答案是“wke”，长度为3.请注意，答案必须是子字符串，“pwke”是子序列而不是子字符串。

<!-- more -->

# 思路

利用滑动窗口思想，滑动窗口内的字符将不会重复，滑动窗口利用两个索引i,j分别指向窗口的前后界限，通过分别移动i，j指针来寻求最大子串长度。


# 方案

## 方法＃1 暴力方式[超时]
- 思路

检查所有子字符串逐个查看是否没有重复的字符。

- 算法

假设我们有一个函数boolean allUnique（String substring），如果子字符串中的字符都是唯一的，否则返回true，否则为false。我们可以遍历给定字符串s的所有可能的子字符串，并调用allUnique函数。如果事实证明是正确的，那么我们更新我们的子字符串的最大长度的答案，而不会有重复的字符。

现在我们填写缺失的部分：

要枚举给定字符串的所有子字符串，我们枚举它们的开始和结束索引。假设开始和结束指数分别为i和j。那么我们有0 <= i < j <=n（这里的结束索引j按照惯例排除）。因此，使用从0到n-1的i的两个嵌套循环和从i + 1到n的j，我们可以枚举s的所有子串。

要检查一个字符串是否有重复的字符，我们可以使用一个字符串。我们遍历字符串中的所有字符，并将它们逐个放入。在放置一个字符之前，我们检查该集合是否已经包含它。如果是这样，我们返回false。循环后，我们返回true。

- Javaw代码

```java
public class Solution{
    public int lengthOfLongestSubstring（String s）{
        int n = s.length（）;
        int ans = 0;
        for（int i = 0; i <n; i ++）
            for（int j = i + 1; j <= n; j ++）
                if（allUnique（s，i，j））ans = Math.max（ans，j-i）;
        return ans;
    }

    public boolean allUnique（String s，int start，int end）{
        set <Character> set = new HashSet <>（）;
        for（int i = start; i <end; i ++）{
            Character ch = s.charAt（i）;
            if（set.contains（ch））return false;
            set.add（ch）;
        }
        return true;
    }
}
```

- 复杂性分析

时间复杂度：O（n ^ 3）

空间复杂度：O（min（n，m））O（min（n，m））。我们需要O（k）空格用于检查子串没有重复字符，其中k是Set的大小。集合的大小由字符串n的大小和字符集/字母表m的大小限定。

## 方法＃2滑动窗口[已接受]

- 算法

上面算法一方法非常简单。但是太慢了那么我们如何才能优化呢？

在上面的方法中，我们反复检查一个子字符串，看看它是否具有重复的字符。但这是没有必要的。如果一个子字符串s[i,j）是从索引i到j-1已经被检查为没有重复的字符。我们只需要检查s[j]是否已经在子串s[i,j)中。要检查字符是否已经在子字符串中，我们可以扫描子字符串，导致O（n ^ 2）算法。但我们可以做得更好。

通过使用HashSet作为滑动窗口，检查当前的字符是否可以在O（1）中完成。

滑动窗口是数组/字符串问题中常用的抽象概念。窗口是数组/字符串中通常由开始和结束索引定义的元素范围，即[i，j)（左闭合，右开）。滑动窗口是一个窗口，将其两个边界滑动到某个方向。例如，如果我们通过一个元素将[i，j）向右滑动，则它变为[i + 1，j + 1）（左闭右开）。

回到我们的问题。我们使用HashSet将字符存储在当前窗口[i，j）（j = i）。然后我们将索引j向右滑动。如果不在HashSet中，我们会进一步滑动j。这样做直到s[j]已经在HashSet中。在这一点上，我们发现没有重复字符的子字符串的最大大小从索引i开始。重复上面的步骤，我们就能得到我们的答案。

- Java代码
```java
public class Solution {
    public int lengthOfLongestSubstring(String s) {
        int n = s.length();
        Set<Character> set = new HashSet<>();
        int ans = 0, i = 0, j = 0;
        while (i < n && j < n) {
            // try to extend the range [i, j]
            if (!set.contains(s.charAt(j))){
                set.add(s.charAt(j++));
                ans = Math.max(ans, j - i);
            }
            else {
                set.remove(s.charAt(i++));
            }
        }
        return ans;
    }
}
```

- 复杂性分析

时间复杂度： O(2n) = O(n)。最糟糕的情况是每个字符都需要被i,j指针访问两次。
空间复杂度： O(min(m,n))。 和上面的方案一样，我们同样需要一个O(k)的空间用于滑动窗口，k表示滑动窗口大小。这个大小取决于字符串n的大小以及字符集的大小m

## 方法＃3滑动窗优化[已接受]

上述方法2中解决方案最多需要2n步。 事实上，它可以被优化，只需要n个步骤。 我们可以定义一个字符与其索引的映射，而不是使用一个字符来判断一个字符是否存在。 然后，当我们发现重复的字符时，我们可以立即跳过这些字符。

原因是如果s[j]在索引j的范围[i，j）中具有重复，重复的这个索引为j'，我们不需要一点一点地增加i。 我们可以跳过[i，j']范围内的所有元素,直接令i=j'+ 1。

- Java代码

```java
public class Solution {
    public int lengthOfLongestSubstring(String s) {
        int n = s.length(), ans = 0;
        Map<Character, Integer> map = new HashMap<>(); // current index of character
        // try to extend the range [i, j]
        for (int j = 0, i = 0; j < n; j++) {
            if (map.containsKey(s.charAt(j))) {
                i = Math.max(map.get(s.charAt(j)), i);
            }
            ans = Math.max(ans, j - i + 1);
            map.put(s.charAt(j), j + 1);
        }
        return ans;
    }
}
```

- 更优代码

假设字符集为ASCII 128

以前的实现都没有对字符串的字符集的假设。

如果我们知道字符集相当小，我们可以将整数数组替换为直接访问表。

常用的表格有：

- int[26] 用于'a'-'z'以及'A'-'Z'
- int[128] 用于ASCII集
- int[256] 用于ASCII扩展集

```java
public class Solution {
    public int lengthOfLongestSubstring(String s) {
        int n = s.length(), ans = 0;
        int[] index = new int[128]; // current index of character
        // try to extend the range [i, j]
        for (int j = 0, i = 0; j < n; j++) {
            i = Math.max(index[s.charAt(j)], i);
            ans = Math.max(ans, j - i + 1);
            index[s.charAt(j)] = j + 1;
        }
        return ans;
    }
}
```

- 复杂度分析

时间复杂度：O(n)
Hashmap空间复杂度：O(min(m,n))
Table方式空间复杂度：O(m) m表示字符集的大小

原文地址：[Longest Substring Without Repeating Characters](https://leetcode.com/problems/longest-substring-without-repeating-characters/description/)
