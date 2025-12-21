
# [Search Insert Position][title_2]

## Description

Given a sorted array and a target value, return the index if the target is found. If not, return the index where it would be if it were inserted in order.

You may assume no duplicates in the array.

**Example 1:**
```
Input: [1,3,5,6], 5
Output: 2
```

**Example 2:**
```
Input: [1,3,5,6], 2
Output: 1
```

**Example 3:**
```
Input: [1,3,5,6], 7
Output: 4
```

**Example 4:**
```
Input: [1,3,5,6], 0
Output: 0
```

**Tags:** Array, Binary Search


## Approach

The objective is to find the index where a `target` value should be inserted into a sorted array without duplicates. Since the array is already sorted, **Binary Search** is the most efficient method. 

The condition for finding the insertion point is to identify the index of the first element that is greater than or equal to the `target`. We can adapt the standard binary search algorithm to continuously narrow down the range until the `left` pointer points to the correct insertion index.

```java
class Solution {
    public int searchInsert(int[] nums, int target) {
        int left = 0, right = nums.length - 1;
        int mid = (right + left) >> 1;
        while (left <= right) {
            if (target <= nums[mid]) {
                right = mid - 1;
            } else {
                left = mid + 1;
            }
            mid = (right + left) >> 1;
        }
        // After the loop, 'left' is the first index where nums[index] >= target
        return left;
    }
}
```


## Conclusion

If you are as passionate about data structures, algorithms, and LeetCode as I am, feel free to follow my LeetCode solutions repository on GitHub: [awesome-java-leetcode][ajl]

[ajl]: https://github.com/Blankj/awesome-java-leetcode
[title_2]: https://leetcode.com/problems/search-insert-position
