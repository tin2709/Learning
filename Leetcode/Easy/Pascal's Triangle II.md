
# [Pascal's Triangle II][title119]

## Description

Given a non-negative index *k* where *k* ≤ 33, return the *k*th index row of the Pascal's triangle.

Note that the row index starts from 0.

![img](https://upload.wikimedia.org/wikipedia/commons/0/0d/PascalTriangleAnimated2.gif)

In Pascal's triangle, each number is the sum of the two numbers directly above it.

**Example:**

```
Input: 3
Output: [1,3,3,1]
```

**Follow up:**

Could you optimize your algorithm to use only *O*(*k*) extra space?

**Tags:** Array

## Analysis

While we could generate the entire triangle as in the previous problem, the challenge here is to do it using only $O(k)$ extra space. 

To achieve this, we maintain a single list and update it in place. However, if we update the list from left to right, we would overwrite values needed for the next calculation in the same row. By updating the list from **right to left** (backward traversal), we ensure that the value at `res.get(j-1)` still represents the value from the previous row, allowing us to calculate the new value for `res.get(j)` correctly.

1. Start with a list for the $i^{th}$ row.
2. For each new row, add `1` to the end.
3. Iterate backward from `i-1` to `1` and update: `row[j] = row[j] + row[j-1]`.

```java
class Solution {
    public List<Integer> getRow(int rowIndex) {
        List<Integer> res = new ArrayList<>();
        for (int i = 0; i <= rowIndex; ++i) {
            // Add the last '1' for the current row
            res.add(1);
            // Update the internal elements from right to left to save space
            for (int j = i - 1; j > 0; --j) {
                res.set(j, res.get(j - 1) + res.get(j));
            }
        }
        return res;
    }
}
```
If you found these explanations helpful, you can follow my full LeetCode solutions repository on GitHub: [Awesome-Java-LeetCode][ajl]

[ajl]: https://github.com/Blankj/awesome-java-leetcode

[title119]: https://leetcode.com/problems/pascals-triangle-ii
