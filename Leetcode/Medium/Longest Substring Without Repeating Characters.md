

# [Longest Substring Without Repeating Characters][title]

## Description

Given a string, find the length of the **longest substring** without repeating characters.

**Examples:**

Given `"abcabcbb"`, the answer is `"abc"`, which the length is 3.

Given `"bbbbb"`, the answer is `"b"`, with the length of 1.

Given `"pwwkew"`, the answer is `"wke"`, with the length of 3. Note that the answer must be a **substring**, `"pwke"` is a *subsequence* and not a substring.

**Tags:** Hash Table, Two Pointers, String


## Approach

The objective is to calculate the length of the longest substring that contains no repeating characters. We can use a hash array (or an integer array used as a map) to store the last seen position of each character. For instance, if `hash[a] = 3`, it indicates that the character 'a' was last encountered at a position that influences our starting boundary. 

As we iterate through the string, we maintain a pointer `preP` representing the start of the current valid substring. This pointer must only move forward (it cannot move backward because characters before it are already excluded). The current length is calculated as the difference between the current index and `preP`. By tracking and updating the maximum length during the iteration, we arrive at the final result.

```java
class Solution {
    public int lengthOfLongestSubstring(String s) {
        int len;
        if (s == null || (len = s.length()) == 0) return 0;
        int preP = 0, max = 0;
        int[] hash = new int[128];
        for (int i = 0; i < len; ++i) {
            char c = s.charAt(i);
            if (hash[c] > preP) {
                preP = hash[c];
            }
            int l = i - preP + 1;
            hash[c] = i + 1;
            if (l > max) max = l;
        }
        return max;
    }
}
```


## Conclusion

If you are as passionate about data structures, algorithms, and LeetCode as I am, feel free to follow my LeetCode solutions repository on GitHub: [awesome-java-leetcode][ajl]



[title]: https://leetcode.com/problems/longest-substring-without-repeating-characters
[ajl]: https://github.com/Blankj/awesome-java-leetcode