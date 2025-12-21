
# [Convert Sorted Array to Binary Search Tree][title108]

## Description

Given an array where elements are sorted in ascending order, convert it to a height-balanced BST.

For this problem, a height-balanced binary tree is defined as a binary tree in which the depth of the two subtrees of *every* node never differ by more than 1.

**Example:**

```
Given the sorted array: [-10,-3,0,5,9],

One possible answer is: [0,-3,9,-10,null,5], which represents the following height balanced BST:

      0
     / \
   -3   9
   /   /
 -10  5
```

**Tags:** Tree, Depth-first Search

## Analysis

To convert a sorted array into a height-balanced Binary Search Tree (BST), we need to maintain the BST properties:
1. The left subtree contains values smaller than the root.
2. The right subtree contains values larger than the root.
3. To ensure the tree is height-balanced, the middle element of the sorted array should be chosen as the root of the current (sub)tree.

The approach uses recursion:
- Find the middle element of the current range in the array and make it the root.
- Recursively repeat the process for the left half of the array to build the left subtree.
- Recursively repeat the process for the right half of the array to build the right subtree.

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
    public TreeNode sortedArrayToBST(int[] nums) {
        if (nums == null || nums.length == 0) return null;
        return helper(nums, 0, nums.length - 1);
    }

    private TreeNode helper(int[] nums, int left, int right) {
        if (left > right) return null;
        
        // Find the middle element to keep the tree balanced
        int mid = (left + right) >>> 1;
        TreeNode node = new TreeNode(nums[mid]);
        
        // Build left and right subtrees recursively
        node.left = helper(nums, left, mid - 1);
        node.right = helper(nums, mid + 1, right);
        
        return node;
    }
}
```

## Conclusion

If you found these explanations helpful, you can follow my full LeetCode solutions repository on GitHub: [Awesome-Java-LeetCode][ajl]

[title108]: https://leetcode.com/problems/convert-sorted-array-to-binary-search-tree
[ajl]: https://github.com/Blankj/awesome-java-leetcode