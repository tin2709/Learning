
# [Remove Element][title_2]

## Description

Given an array *nums* and a value *val*, remove all instances of that value [**in-place**](https://en.wikipedia.org/wiki/In-place_algorithm) and return the new length.

Do not allocate extra space for another array, you must do this by **modifying the input array [in-place](https://en.wikipedia.org/wiki/In-place_algorithm)** with O(1) extra memory.

The order of elements can be changed. It doesn't matter what you leave beyond the new length.

**Example 1:**
```
Given nums = [3,2,2,3], val = 3,
Your function should return length = 2, with the first two elements of nums being 2.
```

**Example 2:**
```
Given nums = [0,1,2,2,3,0,4,2], val = 2,
Your function should return length = 5, with the first five elements of nums containing 0, 1, 3, 0, and 4.
```

**Tags:** Array, Two Pointers


## Approach

The goal is to remove all occurrences of a specific value `val` from an array in-place and return the resulting length. The problem specifies a space complexity of O(1).

My approach uses a `tail` pointer to mark the position where the next "valid" element (one that is not equal to `val`) should be placed. As we iterate through the array, whenever we encounter an element that is not equal to `val`, we copy it to the position indicated by `tail` and then increment `tail`. After the loop finishes, all elements from index `0` to `tail - 1` are the elements not equal to `val`, and `tail` itself represents the new length.

```java
class Solution {
    public int removeElement(int[] nums, int val) {
        int tail = 0;
        for (int i = 0, len = nums.length; i < len; ++i) {
            if (nums[i] != val) {
                nums[tail++] = nums[i];
            }
        }
        return tail;
    }
}
```


## Conclusion

If you are as passionate about data structures, algorithms, and LeetCode as I am, feel free to follow my LeetCode solutions repository on GitHub: [awesome-java-leetcode][ajl]

[ajl]: https://github.com/Blankj/awesome-java-leetcode
[title_2]: https://leetcode.com/problems/remove-element
