# [Maximum Depth of Binary Tree][title104]

## Description

Given a binary tree, find its maximum depth.

The maximum depth is the number of nodes along the longest path from the root node down to the farthest leaf node.

**Note:** A leaf is a node with no children.

**Example:**

Given binary tree `[3,9,20,null,null,15,7]`,

```
    3
   / \
  9  20
    /  \
   15   7
```

return its depth = 3.

**Tags:** Tree, Depth-first Search

## Analysis

The problem asks for the maximum depth of a binary tree, which can be easily solved using a recursive Depth-First Search (DFS) approach. 

The maximum depth of a node is defined as:
1. If the node is `null`, its depth is 0.
2. Otherwise, its depth is $1$ (counting the current node) plus the maximum of the depths of its left and right subtrees.

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
    public int maxDepth(TreeNode root) {
        if (root == null) return 0;
        // Recursive step: 1 + max height of children
        return 1 + Math.max(maxDepth(root.left), maxDepth(root.right));
    }
}
```
If you found these explanations helpful, you can follow my full LeetCode solutions repository on GitHub: [Awesome-Java-LeetCode][ajl]


[ajl]: https://github.com/Blankj/awesome-java-leetcode
[title104]: https://leetcode.com/problems/maximum-depth-of-binary-tree