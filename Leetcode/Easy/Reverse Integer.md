

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

# [Palindrome Number][title_2]

## Description

Determine whether an integer is a palindrome. An integer is a palindrome when it reads the same backward as forward.

**Example 1:**
```
Input: 121
Output: true
```

**Example 2:**
```
Input: -121
Output: false
Explanation: From left to right, it reads -121. From right to left, it becomes 121-. Therefore it is not a palindrome.
```

**Example 3:**
```
Input: 10
Output: false
Explanation: Reads 01 from right to left. Therefore it is not a palindrome.
```

**Follow up:**
Could you solve it without converting the integer to a string?

**Tags:** Math


## Approach 0: Full Reverse

The problem asks to determine if a signed integer is a palindrome, meaning the integer remains identical when its digits are reversed. First, any negative number is automatically not a palindrome. A straightforward solution is to calculate the full reversed version of the number and compare it with the original input.

```java
class Solution {
    public boolean isPalindrome(int x) {
        if (x < 0) return false;
        int copyX = x, reverse = 0;
        while (copyX > 0) {
            reverse = reverse * 10 + copyX % 10;
            copyX /= 10;
        }
        return x == reverse;
    }
}
```

## Approach 1: Half Reverse

We can optimize this by considering whether we actually need to reverse the entire number. For a number like `1234321`, we only need to reverse the second half of the digits. We can stop the process once the reversed part becomes greater than or equal to the remaining part of the original number. 

However, this logic introduces a specific edge case for multiples of 10 (e.g., `10010`), which might incorrectly return `true`. To fix this, we exclude any non-zero number ending in 0. The optimized code is shown below.

```java
class Solution {
    public boolean isPalindrome(int x) {
        if (x < 0 || (x != 0 && x % 10 == 0)) return false;
        int halfReverseX = 0;
        while (x > halfReverseX) {
            halfReverseX = halfReverseX * 10 + x % 10;
            x /= 10;
        }
        return halfReverseX == x || halfReverseX / 10 == x;
    }
}
```


## Conclusion

If you are as passionate about data structures, algorithms, and LeetCode as I am, feel free to follow my LeetCode solutions repository on GitHub: [awesome-java-leetcode][ajl]


[title_1]: https://leetcode.com/problems/reverse-integer
[title_2]: https://leetcode.com/problems/palindrome-number
[ajl]: https://github.com/Blankj/awesome-java-leetcode