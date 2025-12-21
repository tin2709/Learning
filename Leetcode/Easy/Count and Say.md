

# [ Count and Say][title38]

## Description

The count-and-say sequence is the sequence of integers with the first five terms as following:

```
1.     1
2.     11
3.     21
4.     1211
5.     111221
```

- `1` is read off as `"one 1"` or `11`.
- `11` is read off as `"two 1s"` or `21`.
- `21` is read off as `"one 2`, then `one 1"` or `1211`.

Given an integer *n*, generate the *n*<sup>th</sup> term of the count-and-say sequence.

**Note:** Each term of the sequence of integers will be represented as a string.

**Example 1:**
```
Input: 1
Output: "1"
```

**Example 2:**
```
Input: 4
Output: "1211"
```

**Tags:** String

## Analysis

The core logic of this problem is simulation. To generate the next term, we "count" and "say" the previous term. For example, `21` consists of "one 2" and "one 1", which results in `1211`. 

The implementation follows these steps:
1. Start with the string `"1"`.
2. Iterate $n-1$ times to reach the $n^{th}$ term.
3. In each iteration, use a pointer or a loop to count consecutive identical characters.
4. Use a `StringBuilder` to append the count (how many) followed by the digit (the character itself).

```java
class Solution {
    public String countAndSay(int n) {
        String str = "1";
        while (--n > 0) {
            int times = 1;
            StringBuilder sb = new StringBuilder();
            char[] chars = str.toCharArray();
            int len = chars.length;
            for (int j = 1; j < len; j++) {
                if (chars[j - 1] == chars[j]) {
                    times++;
                } else {
                    sb.append(times).append(chars[j - 1]);
                    times = 1;
                }
            }
            str = sb.append(times).append(chars[len - 1]).toString();
        }
        return str;
    }
}
```



If you found these solutions helpful, feel free to check out my full LeetCode repository on GitHub: [Awesome-Java-LeetCode][ajl]

[title38]: https://leetcode.com/problems/count-and-say

[ajl]: https://github.com/Blankj/awesome-java-leetcode