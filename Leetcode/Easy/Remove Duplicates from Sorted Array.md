

# [Remove Duplicates from Sorted Array][title_1]

## Description

Given a sorted array *nums*, remove the duplicates [**in-place**](https://en.wikipedia.org/wiki/In-place_algorithm) such that each element appear only *once* and return the new length.

Do not allocate extra space for another array, you must do this by **modifying the input array [in-place](https://en.wikipedia.org/wiki/In-place_algorithm)** with O(1) extra memory.

**Example 1:**
```
Given nums = [1,1,2],
Your function should return length = 2, with the first two elements of nums being 1 and 2 respectively.
```

**Example 2:**
```
Given nums = [0,0,1,1,1,2,2,3,3,4],
Your function should return length = 5, with the first five elements of nums being modified to 0, 1, 2, 3, and 4 respectively.
```

**Clarification:**
Confused why the returned value is an integer but your answer is an array?
Note that the input array is passed in by **reference**, which means modification to the input array will be known to the caller as well.

**Tags:** Array, Two Pointers


## Approach

The objective is to remove duplicate elements from a sorted array and return the new length. Since the array is already sorted, all duplicate elements will be adjacent to each other.

My approach is as follows: If the array length is 0 or 1, simply return the length as no duplicates can exist. Otherwise, traverse the array starting from the second element. Use a `tail` variable to keep track of the position of the last unique element found. Whenever the current element is different from the previous one, it means we have found a new unique value. We then assign this value to `nums[tail]` and increment `tail`. Finally, return `tail` as the length of the modified array.

```java
class Solution {
    public int removeDuplicates(int[] nums) {
        int len = nums.length;
        if (len <= 1) return len;
        int tail = 1;
        for (int i = 1; i < len; ++i) {
            if (nums[i - 1] != nums[i]) {
                nums[tail++] = nums[i];
            }
        }
        return tail;
    }
}
```


## Conclusion

If you are as passionate about data structures, algorithms, and LeetCode as I am, feel free to follow my LeetCode solutions repository on GitHub: [awesome-java-leetcode][ajl]

---



[title_1]: https://leetcode.com/problems/remove-duplicates-from-sorted-array
[ajl]: https://github.com/Blankj/awesome-java-leetcode