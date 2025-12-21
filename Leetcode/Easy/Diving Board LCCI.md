

# [Diving Board LCCI][title]

## Description

You are building a diving board using a stack of boards. There are two types of boards: a shorter type with length `shorter` and a longer type with length `longer`. You must use **exactly** `k` boards. Write a method to generate all possible lengths of the diving board.

The returned lengths should be sorted from smallest to largest.

**Example:**

```
Input:
shorter = 1
longer = 2
k = 3
Output: {3, 4, 5, 6}
```

**Note:**

*   0 < shorter <= longer
*   0 <= k <= 100000

**Tags:** Recursion, Memoization (Mathematical induction is also used in this solution)


## Approach

At first glance, this problem might look like it requires recursion or dynamic programming. However, upon closer inspection, it is actually a problem involving an **arithmetic progression (sequence)**.

1.  **Case `k == 0`**: If you use zero boards, the only possible length is zero (or technically none), so return an empty array `[]`.
2.  **Case `shorter == longer`**: If the two types of boards have the same length, no matter how you combine them, the total length will always be `k * shorter`. Return `[k * shorter]`.
3.  **Case `shorter != longer`**: This scenario forms an arithmetic progression where:
    *   The **first term** is `k * shorter` (using all shorter boards).
    *   The **last term** is `k * longer` (using all longer boards).
    *   The **common difference** is `longer - shorter` (each time you replace one shorter board with one longer board, the total length increases by this difference).
    *   There are exactly `k + 1` possible lengths.

Based on this logic, we can implement the solution as follows:

```java
public class Solution {
    public int[] divingBoard(int shorter, int longer, int k) {
        if (k == 0) {
            return new int[0];
        }
        if (shorter == longer) {
            return new int[]{shorter * k};
        }
        int[] ans = new int[k + 1];
        int startTerm = k * shorter;      // First term of the arithmetic progression
        int commonDiff = longer - shorter; // Common difference
        for (int i = 0; i <= k; i++) {
            ans[i] = startTerm + i * commonDiff;
        }
        return ans;
    }
}
```


## Conclusion

If you are as passionate about data structures, algorithms, and LeetCode as I am, feel free to follow my LeetCode solutions repository on GitHub: [awesome-java-leetcode][ajl]



[title]: https://leetcode.com/problems/diving-board-lcci/
[ajl]: https://github.com/Blankj/awesome-java-leetcode