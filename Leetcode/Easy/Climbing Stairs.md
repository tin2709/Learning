
# [Climbing Stairs][title70]

## Description

You are climbing a staircase. It takes *n* steps to reach the top.

Each time you can either climb 1 or 2 steps. In how many distinct ways can you climb to the top?

**Note:** Given *n* will be a positive integer.

**Example 1:**

```
Input: 2
Output: 2
Explanation: There are two ways to climb to the top.
1. 1 step + 1 step
2. 2 steps
```

**Example 2:**

```
Input: 3
Output: 3
Explanation: There are three ways to climb to the top.
1. 1 step + 1 step + 1 step
2. 1 step + 2 steps
3. 2 steps + 1 step
```

**Tags:** Dynamic Programming

## Analysis

This is a classic Dynamic Programming problem. To reach the $n^{th}$ step, your very last move must have been either a 1-step jump from the $(n-1)^{th}$ step or a 2-step jump from the $(n-2)^{th}$ step. 

Therefore, the total number of distinct ways to reach the top, $f(n)$, is the sum of the ways to reach the two preceding steps: 
$f(n) = f(n-1) + f(n-2)$.

This logic follows the Fibonacci sequence. In the implementation below, the space complexity is optimized to $O(1)$ by using only two variables to iterate through the sequence, rather than storing an entire DP array.

```java
class Solution {
    public int climbStairs(int n) {
        // Base case: n=1 -> 1 way
        int a = 1, b = 1;
        while (--n > 0) {
            b += a;
            a = b - a;
        }
        return b;
    }
}
```
If you found these explanations helpful, you can follow my full LeetCode solutions repository on GitHub: [Awesome-Java-LeetCode][ajl]


[title70]: https://leetcode.com/problems/climbing-stairs
[ajl]: https://github.com/Blankj/awesome-java-leetcode