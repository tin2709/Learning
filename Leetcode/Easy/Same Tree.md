# [100. Same Tree][title100]

## Description

Given two binary trees, write a function to check if they are the same or not.

Two binary trees are considered the same if they are structurally identical and the nodes have the same value.

**Example 1:**

```
Input:     1         1
          / \       / \
         2   3     2   3

        [1,2,3],   [1,2,3]

Output: true
```

**Example 2:**

```
Input:     1         1
          /           \
         2             2

        [1,2],     [1,null,2]

Output: false
```

**Tags:** Tree, Depth-first Search

## Analysis

To determine if two binary trees are identical, we need to verify two conditions:
1. They must be structurally identical.
2. Corresponding nodes must have the same values.

We can solve this efficiently using a recursive Depth-First Search (DFS) approach:
- **Base Case:** If both nodes are `null`, the subtrees are the same.
- **Structural Mismatch:** If one node is `null` and the other is not, they are different.
- **Value Mismatch:** If the values of the current nodes differ, they are different.
- **Recursion:** If the values match, we recursively check if the left subtrees are the same AND the right subtrees are the same.

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
    public boolean isSameTree(TreeNode p, TreeNode q) {
        // Both are null
        if (p == null && q == null) return true;
        // One is null, the other is not
        if (p == null || q == null) return false;
        // Values match, check subtrees
        if (p.val == q.val) {
            return isSameTree(p.left, q.left) && isSameTree(p.right, q.right);
        }
        return false;
    }
}
```

If you found these explanations helpful, you can follow my full LeetCode solutions repository on GitHub: [Awesome-Java-LeetCode][ajl]

[ajl]: https://github.com/Blankj/awesome-java-leetcode
[title100]: https://leetcode.com/problems/same-tree
