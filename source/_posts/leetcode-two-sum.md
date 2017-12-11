title: leetcode算法题1_两数之和
date: 2017-09-07
tags:
 - leetcode
 - 算法
categories:
 - leetcode
thumbnail: /images/leetcode.png
---

给定一个整型数组，以及一个目标数值V，要求返回两个能够相对等于V值的两个数字的索引序列。每个数字只能使用一次。

例子：
`
Given nums = [2, 7, 11, 15], target = 9,

Because nums[0] + nums[1] = 2 + 7 = 9,
return [0, 1].
`

<!-- more -->

方案：暴力运算


```java
class Solution {
    public int[] twoSum(int[] nums, int target) {
        if(nums.length < 2) return null;
        for(int i=0;i<nums.length;++i){
            for(int j=i+1;j<nums.length;++j){
                if((nums[i] + nums[j]) == target){
                    return new int[]{i,j};
                }
            }
        }
        return null;
    }
}
```

时间复杂度O(n*n)，空间复杂度O(n)

扩展方案：带索引

```java
class Solution {
    public int[] twoSum(int[] nums, int target) {
        Map<Integer,Integer> indexMap = new HashMap<Integer,Integer>();
        for(int i=0;i<nums.length;++i){
            indexMap.put(nums[i],i);
        }

        for(int i=0;i<nums.length;++i){
            Integer idx = indexMap.get(target-nums[i]);
            if(idx != null && i != idx){
                return new int[]{i,idx};
            }
        }
        return null;
    }
}

```

时间复杂度O(n)，空间复杂度O(n)

注：该种方法有一定问题，前提需要加强条件：序列中没有重复的数字。不然该indexMap会被冲突覆盖

如：
Input:  [0,4,3,0] 0
Output：null
Expected: [0,3]

原文地址：https://leetcode.com/problems/two-sum/description/
