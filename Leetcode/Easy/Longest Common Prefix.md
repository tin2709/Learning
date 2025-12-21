

# [Longest Common Prefix][title]

## Description

Write a function to find the longest common prefix string amongst an array of strings.

If there is no common prefix, return an empty string `""`.

**Example 1:**

```
Input: ["flower","flow","flight"]
Output: "fl"
```

**Example 2:**

```
Input: ["dog","racecar","car"]
Output: ""
Explanation: There is no common prefix among the input strings.
```

**Note:**

All given inputs are in lowercase letters `a-z`.

**Tags:** String


## Approach

The goal is to find the longest common prefix among an array of strings. My approach is to first determine the length of the shortest string in the array, let's call it `minLen`. Then, we iterate through the character positions from `0` up to `minLen`. 

For each position `j`, we compare the character at `strs[0].charAt(j)` with the character at the same position in all other strings. If we encounter a character that differs at any point, we immediately return the substring from index `0` to `j`. If the loops complete without finding any mismatches, it means the common prefix is the entirety of the shortest string, which we then return.

```java
class Solution {
    public String longestCommonPrefix(String[] strs) {
        int len = strs.length;
        if (len == 0) return "";
        int minLen = 0x7fffffff;
        // Find the length of the shortest string
        for (String str : strs) minLen = Math.min(minLen, str.length());
        
        // Compare characters at each index across all strings
        for (int j = 0; j < minLen; ++j)
            for (int i = 1; i < len; ++i)
                if (strs[0].charAt(j) != strs[i].charAt(j))
                    return strs[0].substring(0, j);
                    
        return strs[0].substring(0, minLen);
    }
}
```


## Conclusion

If you are as passionate about data structures, algorithms, and LeetCode as I am, feel free to follow my LeetCode solutions repository on GitHub: [awesome-java-leetcode][ajl]



[title]: https://leetcode.com/problems/longest-common-prefix
[ajl]: https://github.com/Blankj/awesome-java-leetcode