

# [69. Sqrt(x)][title69]

## Description

Implement `int sqrt(int x)`.

Compute and return the square root of *x*, where *x* is guaranteed to be a non-negative integer.

Since the return type is an integer, the decimal digits are truncated and only the integer part of the result is returned.

**Example 1:**

```
Input: 4
Output: 2
```

**Example 2:**

```
Input: 8
Output: 2
Explanation: The square root of 8 is 2.82842..., and since the decimal part is truncated, 2 is returned.
```

**Tags:** Binary Search, Math

## Analysis

The goal is to find the integer square root of a non-negative integer. While this can be solved using binary search, a more mathematically elegant and efficient approach is **Newton's Method** (specifically the [Integer Square Root](https://en.wikipedia.org/wiki/Integer_square_root#Using_only_integer_division) algorithm).

The logic follows the iterative formula: $n_{next} = \lfloor \frac{n + \frac{x}{n}}{2} \rfloor$. We start with $n = x$ and continue as long as $n^2 > x$. To avoid potential integer overflow during the calculation of $n \times n$, we use the `long` data type.

```java
class Solution {
    public int mySqrt(int x) {
        if (x == 0) return 0;
        long n = x;
        while (n * n > x) {
            n = (n + x / n) >> 1; // Newton's method: (n + x/n) / 2
        }
        return (int) n;
    }
}
```
If you found these explanations helpful, you can follow my full LeetCode solutions repository on GitHub: [Awesome-Java-LeetCode][ajl]

[ajl]: https://github.com/Blankj/awesome-java-leetcode
[title69]: https://leetcode.com/problems/sqrtx
