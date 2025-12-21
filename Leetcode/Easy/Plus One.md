

# [Plus One][title66]

## Description

Given a **non-empty** array of digits representing a non-negative integer, plus one to the integer.

The digits are stored such that the most significant digit is at the head of the list, and each element in the array contains a single digit.

You may assume the integer does not contain any leading zero, except the number 0 itself.

**Example 1:**

```
Input: [1,2,3]
Output: [1,2,4]
Explanation: The array represents the integer 123.
```

**Example 2:**

```
Input: [4,3,2,1]
Output: [4,3,2,2]
Explanation: The array represents the integer 4321.
```

**Tags:** Array, Math

## Analysis

This problem asks us to simulate basic addition. We start adding 1 to the last element of the array (the least significant digit) and handle the carry-over logic:
1.  Iterate from the end of the array to the beginning.
2.  If the current digit is less than 9, simply increment it and return the array immediately.
3.  If the digit is 9, it becomes 0, and we move to the next digit to add the carry.
4.  If the loop completes and we still have a carry (meaning the original number was all 9s, like `999`), we create a new array with a length of `n + 1`, set the first element to `1`, and return it.

```java
class Solution {
    public int[] plusOne(int[] digits) {
        int p = digits.length - 1;
        // Traverse backward to handle carry-over
        for (int i = p; i >= 0; i--) {
            if (digits[i] < 9) {
                digits[i]++;
                return digits;
            }
            digits[i] = 0;
        }
        
        // If all digits were 9, create a new array (e.g., 99 -> 100)
        int[] res = new int[digits.length + 1];
        res[0] = 1;
        return res;
    }
}
```

## Conclusion

If you found these explanations helpful, you can follow my full LeetCode solutions repository on GitHub: [Awesome-Java-LeetCode][ajl]

[title66]: https://leetcode.com/problems/plus-one
