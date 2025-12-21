

# [Reverse Integer][title_1]

## Description

Given a 32-bit signed integer, reverse digits of an integer.

**Example 1:**
```
Input: 123
Output:  321
```

**Example 2:**
```
Input: -123
Output: -321
```

**Example 3:**
```
Input: 120
Output: 21
```

**Note:**
Assume we are dealing with an environment which could only hold integers within the 32-bit signed integer range. For the purpose of this problem, assume that your function returns 0 when the reversed integer overflows.

**Tags:** Math


## Approach

The goal is to reverse the digits of a given integer. A key detail to handle is potential overflow: if the reversed integer exceeds the 32-bit signed integer range, the function must return 0. To implement this, we can store the reversed result in a `long` variable. Finally, we compare the result with the boundaries of a standard integer (`Integer.MAX_VALUE` and `Integer.MIN_VALUE`) before returning it.

```java
class Solution {
    public int reverse(int x) {
        long res = 0;
        for (; x != 0; x /= 10)
            res = res * 10 + x % 10;
        return res > Integer.MAX_VALUE || res < Integer.MIN_VALUE ? 0 : (int) res;
    }
}
```


## Conclusion

If you are as passionate about data structures, algorithms, and LeetCode as I am, feel free to follow my LeetCode solutions repository on GitHub: [awesome-java-leetcode][ajl]

---


## Conclusion

If you are as passionate about data structures, algorithms, and LeetCode as I am, feel free to follow my LeetCode solutions repository on GitHub: [awesome-java-leetcode][ajl]


[title_1]: https://leetcode.com/problems/reverse-integer
[ajl]: https://github.com/Blankj/awesome-java-leetcode