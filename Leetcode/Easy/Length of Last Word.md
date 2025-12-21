

# [58. Length of Last Word][title58]

## Description

Given a string *s* consists of upper/lower-case alphabets and empty space characters `' '`, return the length of the last word in the string.

If the last word does not exist, return 0.

**Note:** A word is defined as a character sequence consists of non-space characters only.

**Example:**

```
Input: "Hello World"
Output: 5
```

**Tags:** String

## Analysis

The goal is to find the length of the last word in a string. A word is defined as a sequence of non-space characters. 

The approach is straightforward: traverse the string backward.
1.  First, ignore any trailing spaces to find the end index of the last word.
2.  Then, continue traversing backward to find the index of the space preceding that word (or until the beginning of the string).
3.  The difference between these two indices gives us the length.

While this can be solved in a single line using Java APIs (e.g., `return s.trim().length() - s.trim().lastIndexOf(" ") - 1;`), implementing the logic manually is better for understanding basic string manipulation and pointer traversal.

```java
class Solution {
    public int lengthOfLastWord(String s) {
        int p = s.length() - 1;
        // Skip trailing spaces
        while (p >= 0 && s.charAt(p) == ' ') p--;
        int end = p;
        // Count characters of the last word
        while (p >= 0 && s.charAt(p) != ' ') p--;
        return end - p;
    }
}
```


## Conclusion

If you found these explanations helpful, you can follow my full LeetCode solutions repository on GitHub: [Awesome-Java-LeetCode][ajl]

[title58]: https://leetcode.com/problems/length-of-last-word
[ajl]: https://github.com/Blankj/awesome-java-leetcode