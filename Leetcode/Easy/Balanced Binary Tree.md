
# [Balanced Binary Tree][title110]

## Description

Given a binary tree, determine if it is height-balanced.

For this problem, a height-balanced binary tree is defined as:

> a binary tree in which the depth of the two subtrees of *every* node never differ by more than 1.

**Example 1:**

Given the following tree `[3,9,20,null,null,15,7]`:

```
    3
   / \
  9  20
    /  \
   15   7
```

Return true.

**Example 2:**

Given the following tree `[1,2,2,3,3,null,null,4,4]`:

```
       1
      / \
     2   2
    / \
   3   3
  / \
 4   4
```

Return false.

**Tags:** Tree, Depth-first Search

## Analysis

A binary tree is height-balanced if the height difference between the left and right subtrees of **every** node is no more than 1.

To solve this optimally, we can use a bottom-up approach while calculating heights:
1.  Recursively calculate the height of the left and right subtrees.
2.  If any subtree is found to be unbalanced, return `-1` to immediately signal the imbalance up the recursion stack.
3.  If a node's left and right heights differ by more than 1, return `-1`.
4.  Otherwise, return the actual height of the node: `1 + Math.max(leftHeight, rightHeight)`.

This approach ensures $O(n)$ time complexity as each node is visited only once.

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
    public boolean isBalanced(TreeNode root) {
        return helper(root) != -1;
    }

    private int helper(TreeNode node) {
        if (node == null) return 0;
        
        // Calculate left subtree height
        int l = helper(node.left);
        if (l == -1) return -1; // Already unbalanced
        
        // Calculate right subtree height
        int r = helper(node.right);
        if (r == -1) return -1; // Already unbalanced
        
        // Check current node balance
        if (Math.abs(l - r) > 1) return -1;
        
        // Return actual height
        return 1 + Math.max(l, r);
    }
}
```
If you found these explanations helpful, you can follow my full LeetCode solutions repository on GitHub: [Awesome-Java-LeetCode][ajl]


[title110]: https://leetcode.com/problems/balanced-binary-tree
[ajl]: https://github.com/Blankj/awesome-java-leetcode