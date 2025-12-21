

# [Roman to Integer][title]

## Description

Roman numerals are represented by seven different symbols: `I`, `V`, `X`, `L`, `C`, `D` and `M`.

```
Symbol       Value
I             1
V             5
X             10
L             50
C             100
D             500
M             1000
```

For example, two is written as `II` in Roman numeral, just two one's added together. Twelve is written as, `XII`, which is simply `X` + `II`. The number twenty seven is written as `XXVII`, which is `XX` + `V` + `II`.

Roman numerals are usually written largest to smallest from left to right. However, the numeral for four is not `IIII`. Instead, the number four is written as `IV`. Because the one is before the five we subtract it making four. The same principle applies to the number nine, which is written as `IX`. There are six instances where subtraction is used:

- `I` can be placed before `V` (5) and `X` (10) to make 4 and 9. 
- `X` can be placed before `L` (50) and `C` (100) to make 40 and 90. 
- `C` can be placed before `D` (500) and `M` (1000) to make 400 and 900.

Given a roman numeral, convert it to an integer. Input is guaranteed to be within the range from 1 to 3999.

**Example 1:**
```
Input: "III"
Output: 3
```

**Example 2:**
```
Input: "IV"
Output: 4
```

**Example 3:**
```
Input: "IX"
Output: 9
```

**Example 4:**
```
Input: "LVIII"
Output: 58
Explanation: C = 100, L = 50, XXX = 30 and III = 3.
```

**Example 5:**
```
Input: "MCMXCIV"
Output: 1994
Explanation: M = 1000, CM = 900, XC = 90 and IV = 4.
```

**Tags:** Math, String


## Approach

The task is to convert a Roman numeral string into an integer within the range of 1 to 3999. According to the standard rules for Roman numerals:

1.  **Addition:** Identical symbols written consecutively are added together (e.g., III = 3).
2.  **Right-side Addition:** If a smaller value is placed to the right of a larger value, they are added together (e.g., VIII = 8, XII = 12).
3.  **Left-side Subtraction:** If a smaller value (limited to I, X, and C) is placed to the left of a larger value, the smaller value is subtracted from the larger value (e.g., IV = 4, IX = 9).

We can use a `Map` to store the mapping between the seven Roman symbols (I, V, X, L, C, D, M) and their corresponding integer values. Then, we iterate through the string—scanning from right to left is particularly efficient here—and apply the rules to calculate the final sum.

```java
class Solution {
    public int romanToInt(String s) {
        Map<Character, Integer> map = new HashMap<>();
        map.put('I', 1);
        map.put('V', 5);
        map.put('X', 10);
        map.put('L', 50);
        map.put('C', 100);
        map.put('D', 500);
        map.put('M', 1000);
        
        int len = s.length();
        int sum = map.get(s.charAt(len - 1));
        
        // Traverse from right to left
        for (int i = len - 2; i >= 0; --i) {
            // If current value is less than the value to its right, subtract it
            if (map.get(s.charAt(i)) < map.get(s.charAt(i + 1))) {
                sum -= map.get(s.charAt(i));
            } else {
                // Otherwise, add it
                sum += map.get(s.charAt(i));
            }
        }
        return sum;
    }
}
```


## Conclusion

If you are as passionate about data structures, algorithms, and LeetCode as I am, feel free to follow my LeetCode solutions repository on GitHub: [awesome-java-leetcode][ajl]



[title]: https://leetcode.com/problems/roman-to-integer
[ajl]: https://github.com/Blankj/awesome-java-leetcode