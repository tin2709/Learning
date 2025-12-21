

# [Pascal's Triangle][title118]

## Description

Given a non-negative integer *numRows*, generate the first *numRows* of Pascal's triangle.

![img](https://upload.wikimedia.org/wikipedia/commons/0/0d/PascalTriangleAnimated2.gif)

In Pascal's triangle, each number is the sum of the two numbers directly above it.

**Example:**

```
Input: 5
Output:
[
     [1],
    [1,1],
   [1,2,1],
  [1,3,3,1],
 [1,4,6,4,1]
]
```

**Tags:** Array

## Analysis

This problem is a straightforward simulation of Pascal's Triangle construction. The rules are as follows:
1. The first and last element of every row is always `1`.
2. Any other element at index $j$ in row $i$ is the sum of the elements at indices $j-1$ and $j$ from row $i-1$.

We can simply iterate through the number of rows required and build each row based on the values stored in the previously generated row.

```java
class Solution {
    public List<List<Integer>> generate(int numRows) {
        if (numRows == 0) return Collections.emptyList();
        List<List<Integer>> list = new ArrayList<>();
        
        for (int i = 0; i < numRows; ++i) {
            List<Integer> sub = new ArrayList<>();
            for (int j = 0; j <= i; ++j) {
                // First and last elements of each row are 1
                if (j == 0 || j == i) {
                    sub.add(1);
                } else {
                    // Current element = sum of two elements directly above it
                    List<Integer> upSub = list.get(i - 1);
                    sub.add(upSub.get(j - 1) + upSub.get(j));
                }
            }
            list.add(sub);
        }
        return list;
    }
}
```

## Conclusion

If you found these explanations helpful, you can follow my full LeetCode solutions repository on GitHub: [Awesome-Java-LeetCode][ajl]

[title118]: https://leetcode.com/problems/pascals-triangle
[ajl]: https://github.com/Blankj/awesome-java-leetcode