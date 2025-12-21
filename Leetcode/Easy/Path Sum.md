
# [Path Sum][title112]

## Description

Given a binary tree and a sum, determine if the tree has a root-to-leaf path such that adding up all the values along the path equals the given sum.

**Note:** A leaf is a node with no children.

**Example:**

Given the below binary tree and `sum = 22`,

```
      5
     / \
    4   8
   /   / \
  11  13  4
 /  \      \
7    2      1
```

return true, as there exist a root-to-leaf path `5->4->11->2` which sum is 22.

**Tags:** Tree, Depth-first Search

## Analysis

The problem asks us to find whether there is a path from the root node to any leaf node where the total sum of node values equals a specific target.

We can solve this efficiently using Depth-First Search (DFS):
1.  **Base Case:** If the current node is `null`, it's not a path, so return `false`.
2.  **Leaf Condition:** If the current node is a leaf (both `left` and `right` are `null`), check if its value equals the remaining `sum`.
3.  **Recursive Step:** Subtract the current node's value from the target `sum` and recursively call the function for both the left and right children. If either branch returns `true`, the target sum exists in a path.

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
    public boolean hasPathSum(TreeNode root, int sum) {
        if (root == null) return false;
        // Check if it's a leaf node
        if (root.left == null && root.right == null) return sum == root.val;
        // Subtract current value and recurse through children
        return hasPathSum(root.left, sum - root.val) || hasPathSum(root.right, sum - root.val);
    }
}
```

---

## Conclusion

If you found these explanations helpful, you can follow my full LeetCode solutions repository on GitHub: [Awesome-Java-LeetCode][ajl]

[title112]: https://leetcode.com/problems/path-sum
[ajl]: https://github.com/Blankj/awesome-java-leetcode