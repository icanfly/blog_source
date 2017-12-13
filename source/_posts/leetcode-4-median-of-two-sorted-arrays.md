title: leetcode算法4_两排序数组求中位平均数
date: 2017-09-08
tags:
 - leetcode
 - 算法
categories:
 - leetcode
---

# 题目

有两个排序的数组nums1和nums2分别为m和n大小。

找到两个排序数组的中位数。 整体运行时间复杂度应为O（log（m + n））。

示例1：
nums1 = [1,3]
nums2 = [2]

中位数为2.0
示例2：
nums1 = [1,2]
nums2 = [3,4]

中位数为（2 + 3）/ 2 = 2.5

<!-- more -->

# 分析

本题中的中位数是指对于一个长度为n的数组，如果n为偶数，则中位数为下标为n/2和n/2+1的两数相加取平均值；如果n为奇数，则中位数为下标为n/2+1的数。

对于两个数组，我们知道合并成一个数组后，同样适用上面的方式。所以我们可以根据上面的描述确定对于两个数组遍历得到的中位数下标，同时我们定义一个计数器，以确定在按序遍历时的计数，当计数与我们确定的中位数下标达到一致状态时，即可确定我们的中位数。

本题中主要关注已排序这个前提条件，我们可以使用两个指针分别指向两个数组的头部（数组下标为0的位置），两个指针对应的数进行比较得出较小值，将计数器与预计算的中位数的下标进行对比，判断是否达到中位数下标，最后累加计数器。

# 算法代码

```java
class Solution {
    public double findMedianSortedArrays(int[] nums1, int[] nums2) {
        int idx1 = 0,idx2 = 0;
        int idx1Max = nums1.length;
        int idx2Max = nums2.length;
        int total = idx1Max + idx2Max;

        int[] middles = (total % 2) == 0 ? (new int[]{total / 2,total / 2 + 1} ) : (new int[]{total / 2 + 1});
        int counter = 1;
        int num = 0;
        int median = 0;

        while(idx1 < idx1Max || idx2 < idx2Max){
            if(idx1 < idx1Max && idx2 < idx2Max){
                if(nums1[idx1] > nums2[idx2]){
                    num = nums2[idx2];
                    idx2++;
                }else{
                    num = nums1[idx1];
                    idx1++;
                }
            }else if(idx1 >= idx1Max && idx2 < idx2Max){
                num = nums2[idx2];
                idx2++;
            }else if(idx1 < idx1Max && idx2 >= idx2Max){
                num = nums1[idx1];
                idx1++;
            }else{
                throw new RuntimeException("can't reach here");
            }


            if(middles.length == 1 && counter == middles[0]){
                return (double) num;
            }else if(middles.length == 2){
                if(counter == middles[0]){
                    median += num;
                }else if(counter == middles[1]){
                    median += num;
                    return (double) median / 2;
                }
            }

            counter++;


        }

        return 0;

    }
}
```

# 复杂度分析

空间复杂度：O（1），没有使用额外的空间用于计算，只有一些变量值，空间忽略。
时间复杂度：O(log(m+n))，因在一般情况下对于两个数组基本确定在遍历到一半的情况下都能找到结果，故在m+n两数组总长度与计算耗时上存在2的倍数关系，故为O(log(m+n))。

原文地址：[4. Median of Two Sorted Arrays](https://leetcode.com/problems/median-of-two-sorted-arrays/description/)
