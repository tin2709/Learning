
# [Implement strStr()][title_1]

## Description

Implement [strStr()](http://www.cplusplus.com/reference/cstring/strstr/).

Return the index of the first occurrence of needle in haystack, or **-1** if needle is not part of haystack.

**Example 1:**
```
Input: haystack = "hello", needle = "ll"
Output: 2
```

**Example 2:**
```
Input: haystack = "aaaaa", needle = "bba"
Output: -1
```

**Clarification:**
What should we return when `needle` is an empty string? This is a great question to ask during an interview. For the purpose of this problem, we will return 0 when `needle` is an empty string.

**Tags:** Two Pointers, String


## Approach

The task is to find the starting index of a substring (`needle`) within a main string (`haystack`). If the substring is not found, return -1. If the length of the `needle` is greater than the `haystack`, it's impossible for it to exist as a substring, so we immediately return -1. Otherwise, we iterate through the `haystack` and compare characters one by one.

```java
class Solution {
    public int strStr(String haystack, String needle) {
        int l1 = haystack.length(), l2 = needle.length();
        if (l1 < l2) return -1;
        for (int i = 0; ; i++) {
            // If the remaining length of haystack is less than needle, no match possible
            if (i + l2 > l1) return -1;
            for (int j = 0; ; j++) {
                // Successfully matched all characters of needle
                if (j == l2) return i;
                // Character mismatch, break inner loop to try next starting position
                if (haystack.charAt(i + j) != needle.charAt(j)) break;
            }
        }
    }
}
```


## Conclusion

If you are as passionate about data structures, algorithms, and LeetCode as I am, feel free to follow my LeetCode solutions repository on GitHub: [awesome-java-leetcode][ajl]

[title_1]: https://leetcode.com/problems/implement-strstr
[ajl]: https://github.com/Blankj/awesome-java-leetcode