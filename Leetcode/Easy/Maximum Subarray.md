# [Maximum Subarray][title53]

## Description

Given an integer array `nums`, find the contiguous subarray (containing at least one number) which has the largest sum and return its sum.

**Example:**
```
Input: [-2,1,-3,4,-1,2,1,-5,4],
Output: 6
Explanation: [4,-1,2,1] has the largest sum = 6.
```

**Follow up:**
If you have figured out the $O(n)$ solution, try coding another solution using the divide and conquer approach, which is more subtle.

**Tags:** Array, Divide and Conquer, Dynamic Programming

## Analysis 0: Dynamic Programming (Optimal)

This is a classic optimization problem. We can solve it using Dynamic Programming (Kadane's Algorithm). 
If the sum of the previous subarray is positive, it will contribute to increasing the current sum. If it is negative, we should discard the previous subarray and start a new one from the current element.

The transition equation is: `dp[i] = nums[i] + (dp[i - 1] > 0 ? dp[i - 1] : 0)`.
Since we only need the previous state, we can optimize the space complexity to $O(1)$.

```java
class Solution {
    public int maxSubArray(int[] nums) {
        int len = nums.length, dp = nums[0], max = dp;
        for (int i = 1; i < len; ++i) {
            dp = nums[i] + (dp > 0 ? dp : 0);
            if (dp > max) max = dp;
        }
        return max;
    }
}
```

## Analysis 1: Divide and Conquer

The divide and conquer approach splits the array into two halves. The maximum subarray sum must be in one of three places:
1. Entirely in the left half.
2. Entirely in the right half.
3. Crossing the midpoint (includes elements from both left and right sides).

We recursively calculate the max sum for the left and right halves, and calculate the crossing sum by expanding from the midpoint to both ends.

```java
class Solution {
    public int maxSubArray(int[] nums) {
        return helper(nums, 0, nums.length - 1);
    }

    private int helper(int[] nums, int left, int right) {
        if (left >= right) return nums[left];
        int mid = (left + right) >> 1;
        
        // Max sum in left and right halves
        int leftAns = helper(nums, left, mid);
        int rightAns = helper(nums, mid + 1, right);
        
        // Max sum crossing the midpoint
        int leftMax = nums[mid], rightMax = nums[mid + 1];
        int temp = 0;
        for (int i = mid; i >= left; --i) {
            temp += nums[i];
            if (temp > leftMax) leftMax = temp;
        }
        temp = 0;
        for (int i = mid + 1; i <= right; ++i) {
            temp += nums[i];
            if (temp > rightMax) rightMax = temp;
        }
        
        return Math.max(Math.max(leftAns, rightAns), leftMax + rightMax);
    }
}
```

## Conclusion

If you found these solutions helpful, feel free to check out my full LeetCode repository on GitHub: [Awesome-Java-LeetCode][ajl]

[ajl]: https://github.com/Blankj/awesome-java-leetcode
[title53]: https://leetcode.com/problems/maximum-subarray
