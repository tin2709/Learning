
# [ Add Binary][title67]

## Description

Given two binary strings, return their sum (also a binary string).

The input strings are both **non-empty** and contain only characters `1` or `0`.

**Example 1:**

```
Input: a = "11", b = "1"
Output: "100"
```

**Example 2:**

```
Input: a = "1010", b = "1011"
Output: "10101"
```

**Tags:** Math, String

## Analysis

To add two binary strings, we simulate the standard column-addition method used in elementary arithmetic, moving from the rightmost character (least significant bit) to the left.

1.  Use a variable `carry` to store the overflow from the previous position.
2.  Iterate through both strings simultaneously using two pointers (`p1` and `p2`).
3.  Calculate the sum of bits at the current position along with the carry.
4.  The new bit for the result is `sum % 2`, and the new carry is `sum / 2`.
5.  Continue until all bits and the final carry are processed.

*Note: In the implementation below, `sb.insert(0, ...)` is used for clarity, though `sb.append()` followed by `sb.reverse()` is generally more performant in Java.*

```java
class Solution {
    public String addBinary(String a, String b) {
        StringBuilder sb = new StringBuilder();
        int carry = 0, p1 = a.length() - 1, p2 = b.length() - 1;
        
        while (p1 >= 0 || p2 >= 0) {
            if (p1 >= 0) carry += a.charAt(p1--) - '0';
            if (p2 >= 0) carry += b.charAt(p2--) - '0';
            
            sb.insert(0, (char) (carry % 2 + '0'));
            carry >>= 1; // equivalent to carry / 2
        }
        
        if (carry == 1) {
            sb.insert(0, '1');
        }
        
        return sb.toString();
    }
}
```

## Conclusion

If you found these explanations helpful, you can follow my full LeetCode solutions repository on GitHub: [Awesome-Java-LeetCode][ajl]

[title67]: https://leetcode.com/problems/add-binary
[ajl]: https://github.com/Blankj/awesome-java-leetcode