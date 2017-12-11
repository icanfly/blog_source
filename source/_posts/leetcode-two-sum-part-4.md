title: leetcode算法题653_两数之和（IV）
date: 2017-09-07
tags:
 - leetcode
 - 算法
categories:
 - leetcode
thumbnail: /images/leetcode.png
---

给定二进制搜索树和目标数字，如果BST中存在两个元素，使得它们的和等于给定的目标，则返回true。

Example 1:

Input:

          5
          / \
         3   6
        / \   \
       2   4   7


Target = 9

Output: True

Example 2:


Input:

           5
           / \
          3   6
         / \   \
        2   4   7


Target = 28

Output: False

<!-- more -->


方案：

 ```java
 /**
 * Definition for a binary tree node.
 * public class TreeNode {
 *     int val;
 *     TreeNode left;
 *     TreeNode right;
 *     TreeNode(int x) { val = x; }
 * }
 */
class Solution {
    public boolean findTarget(TreeNode root, int k) {
        return traval(root,root,k);
    }

    /**
    * 进行遍历，root用于在遍历和搜索中透传，node为要遍历的节点，k为target值
    **/
    public boolean travel(TreeNode root,TreeNode node, int k){
        if(searchVal(root,node,k-node.val)) return true;
        if(node.left != null){
            if(travel(root,node.left,k)){
                return true;
            }
        }
        if(node.right != null){
            if(travel(root,node.right,k)){
                return true;
            }
        }

        return false;
    }

    /**
    * 搜索node节点及其子节点中是否有值和val相等，并且节点不为src
    **/
    public boolean searchVal(TreeNode node,TreeNode src, int val){
        if(node == null) return false;
        if(node.val == val && node != src) return true;
        else if(val > node.val && node.right != null){
            return searchVal(node.right,src,val);
        }
        else if(val < node.val && node.left != null){
            return searchVal(node.left,src,val);
        }else{
            return false;
        }
    }
}
 ```

 原文地址：[Two Sum IV - Input is a BST](https://leetcode.com/problems/two-sum-iv-input-is-a-bst/description/)
