title: leetcode算法题167_两数之和（II）
date: 2017-09-07
tags:
 - leetcode
 - 算法
categories:
 - leetcode

---

给定一个已经按升序排序的整数数组，找到两个数字，使它们相加到一个特定的目标数。

函数twoSum应该返回两个数字的索引，使它们相加到目标，其中index1必须小于index2。 请注意，您返回的答案（index1和index2）都不是基于零的。

您可以假设每个输入都将具有一个解决方案，您可能不会使用相同的元素两次。

输入：numbers = {2，7，11，15}，target = 9
输出：index1 = 1，index2 = 2

<!-- more -->

方案：

```java
class Solution {
    public int[] twoSum(int[] numbers, int target) {
        Map<Integer,Integer> indexMap = new HashMap<Integer,Integer>();
        for(int i=0;i<numbers.length;++i){
            indexMap.put(numbers[i],i);
        }
        
        for(int i=0;i<numbers.length;++i){
            Integer idx = indexMap.get(target-numbers[i]);
            if(idx != null && i != idx){
                return new int[]{i+1,idx+1};
            }
        }
        return null;
    }
}
```

此题同时可见：[leetcode算法题1_两数之和](/leetcode-two-sum/)

原文地址：https://leetcode.com/problems/two-sum-ii-input-array-is-sorted/description/