



# [Merge Sorted Array][title88]

## Description

Given two sorted integer arrays *nums1* and *nums2*, merge *nums2* into *nums1* as one sorted array.

**Note:**

- The number of elements initialized in *nums1* and *nums2* are *m* and *n* respectively.
- You may assume that *nums1* has enough space (size that is greater or equal to *m* + *n*) to hold additional elements from *nums2*.

**Example:**

```
Input:
nums1 = [1,2,3,0,0,0], m = 3
nums2 = [2,5,6],       n = 3

Output: [1,2,2,3,5,6]
```

**Tags:** Array, Two Pointers

## Analysis

The task is to merge `nums2` into `nums1` in-place. Since both arrays are already sorted, we can use a **Two Pointers** approach.

- If we start merging from the beginning (index 0), we would need an auxiliary array to store the elements, otherwise, we might overwrite elements in `nums1` that haven't been compared yet.
- A better strategy is to merge from the **end** (right to left). Since `nums1` has extra space at the back, merging backwards allows us to overwrite the "0" placeholders without needing extra space.
- We compare the largest elements from both arrays (at indices `m-1` and `n-1`) and place the larger one at the end of `nums1` (index `m+n-1`).
- Finally, if there are remaining elements in `nums2`, we copy them into the front of `nums1`. (If elements remain in `nums1`, they are already in their correct positions).

```java
class Solution {
    public void merge(int[] nums1, int m, int[] nums2, int n) {
        int p = m + n - 1; // Tail of nums1
        m--; // Last initialized element in nums1
        n--; // Last initialized element in nums2
        
        while (m >= 0 && n >= 0) {
            // Place the larger element at the end
            nums1[p--] = nums1[m] > nums2[n] ? nums1[m--] : nums2[n--];
        }
        
        // If elements remain in nums2, copy them
        while (n >= 0) {
            nums1[p--] = nums2[n--];
        }
    }
}
```

## Conclusion

If you found these explanations helpful, you can follow my full LeetCode solutions repository on GitHub: [Awesome-Java-LeetCode][ajl]

[title88]: https://leetcode.com/problems/merge-sorted-array
[ajl]: https://github.com/Blankj/awesome-java-leetcode