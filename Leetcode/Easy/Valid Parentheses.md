

# [Valid Parentheses][title]

## Description

Given a string containing just the characters `'('`, `')'`, `'{'`, `'}'`, `'['` and `']'`, determine if the input string is valid.

An input string is valid if:

1. Open brackets must be closed by the same type of brackets.
2. Open brackets must be closed in the correct order.

Note that an empty string is also considered valid.

**Example 1:**
```
Input: "()"
Output: true
```

**Example 2:**
```
Input: "()[]{}"
Output: true
```

**Example 3:**
```
Input: "(]"
Output: false
```

**Example 4:**
```
Input: "([)]"
Output: false
```

**Example 5:**
```
Input: "{[]}"
Output: true
```

**Tags:** Stack, String


## Approach

The objective is to determine whether the parentheses in a string match correctly. This is a classic problem that can be solved using a **Stack**. 

The logic is straightforward: when we encounter an opening bracket, we push it onto the stack. When we encounter a closing bracket, we check if the top element of the stack is the corresponding opening bracket. If they don't match, or if the stack is empty when a closing bracket appears, the string is invalid and we return `false`. After iterating through the entire string, the string is valid only if the stack is empty.

In the implementation below, we simulate the stack using a `char[]` array for better performance. A clever detail is initializing `top = 1` and using an array size of `s.length() + 1`. This ensures that when the first character is a closing bracket, `stack[--top]` accesses `stack[0]` (which is empty/null) rather than causing an array index out of bounds error, allowing the comparison to return `false` naturally.

```java
class Solution {
    public boolean isValid(String s) {
        // Use an array to simulate a stack for faster performance
        char[] stack = new char[s.length() + 1];
        int top = 1;
        for (char c : s.toCharArray()) {
            if (c == '(' || c == '[' || c == '{') {
                stack[top++] = c; 
            } else if (c == ')' && stack[--top] != '(') {
                return false;
            } else if (c == ']' && stack[--top] != '[') {
                return false;
            } else if (c == '}' && stack[--top] != '{') {
                return false;
            }
        }
        // If top is back to 1, all brackets were matched and popped
        return top == 1;
    }
}
```


## Conclusion

If you are as passionate about data structures, algorithms, and LeetCode as I am, feel free to follow my LeetCode solutions repository on GitHub: [awesome-java-leetcode][ajl]



[title]: https://leetcode.com/problems/valid-parentheses
[ajl]: https://github.com/Blankj/awesome-java-leetcode