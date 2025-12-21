
# [Symmetric Tree][title]

## Description

Given a binary tree, check whether it is a mirror of itself (ie, symmetric around its center).

For example, this binary tree `[1,2,2,3,4,4,3]` is symmetric:

```
    1
   / \
  2   2
 / \ / \
3  4 4  3
```

But the following `[1,2,2,null,3,null,3]` is not:

```
    1
   / \
  2   2
   \   \
   3    3
```

**Note:**

Bonus points if you could solve it both recursively and iteratively.

**Tags:** Tree, Depth-first Search, Breadth-first Search


## Approach 0: Recursion (DFS)

The objective is to determine if a binary tree is symmetric. The most intuitive way to solve this is using Depth-First Search (DFS) to compare the left and right subtrees of the root. 

For two nodes to be symmetric:
1. Their values must be equal.
2. The left child of the left subtree must be a mirror of the right child of the right subtree.
3. The right child of the left subtree must be a mirror of the left child of the right subtree.

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
    public boolean isSymmetric(TreeNode root) {
        return root == null || helper(root.left, root.right);
    }

    private boolean helper(TreeNode left, TreeNode right) {
        if (left == null || right == null) return left == right;
        if (left.val != right.val) return false;
        // Compare outer nodes (left.left & right.right) and inner nodes (left.right & right.left)
        return helper(left.left, right.right) && helper(left.right, right.left);
    }
}
```

## Approach 1: Iteration (BFS)

The second approach utilizes Breadth-First Search (BFS). BFS typically requires a queue; in Java, a `LinkedList` can serve as an implementation. 

We add the left and right children of the root into the queue in pairs. In each iteration, we extract two nodes and compare them. If they match, we push their children into the queue in a specific order: the left node's left child with the right node's right child (the outer pair), and the left node's right child with the right node's left child (the inner pair).

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
     public boolean isSymmetric(TreeNode root) {
        if (root == null) return true;
        LinkedList<TreeNode> q = new LinkedList<>();
        q.add(root.left);
        q.add(root.right);
        
        while (q.size() > 1) {
            TreeNode left = q.poll();
            TreeNode right = q.poll();
            
            if (left == null && right == null) continue;
            if (left == null || right == null) return false;
            if (left.val != right.val) return false;
            
            // Add outer pair
            q.add(left.left);
            q.add(right.right);
            // Add inner pair
            q.add(left.right);
            q.add(right.left);
        }
        return true;
    }
}
```


## Conclusion

If you are as passionate about data structures, algorithms, and LeetCode as I am, feel free to follow my LeetCode solutions repository on GitHub: [awesome-java-leetcode][ajl]



[title]: https://leetcode.com/problems/symmetric-tree
[ajl]: https://github.com/Blankj/awesome-java-leetcode