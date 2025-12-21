
# [Two Sum][title]

## Description

Given an array of integers, return **indices** of the two numbers such that they add up to a specific target.

You may assume that each input would have ***exactly*** one solution, and you may not use the same element twice.

**Example:**

```
Given nums = [2, 7, 11, 15], target = 9,

Because nums[0] + nums[1] = 2 + 7 = 9,
return [0, 1].
```

**Tags:** Array, Hash Table


## Approach 0: Brute Force

The goal is to find the indices of two elements in a given array that sum up to a specific value. The most straightforward method is using a nested loop to check every pair, which has a time complexity of `O(n^2)`. Interestingly, the first submission of this code took only 2ms, beating 100% of submissions—a mysterious result, as subsequent submissions never reached that speed again.

```java
class Solution {
    public int[] twoSum(int[] nums, int target) {
        for (int i = 0; i < nums.length; ++i) {
            for (int j = i + 1; j < nums.length; ++j) {
                if (nums[i] + nums[j] == target) {
                    return new int[]{i, j};
                }
            }
        }
        return null;
    }
}
```

## Approach 1: Hash Table (One-pass)

We can use a `HashMap` for storage, where the **key** represents the "complement" (target minus the current element) and the **value** represents the index. 

For example, when `i = 0` and `nums[0] = 2`, we first check if `2` exists in the map. If it does not, we insert the key-value pair `key = 9 - 2 = 7, value = 0`. Later, when `i = 1` and `nums[1] = 7`, we check the map and find that `7` already exists. We then retrieve the stored `value = 0` as the first index and use the current `i = 1` as the second index. This approach has a time complexity of `O(n)`.

```java
class Solution {
    public int[] twoSum(int[] nums, int target) {
        int len = nums.length;
        HashMap<Integer, Integer> map = new HashMap<>();
        for (int i = 0; i < len; ++i) {
            final Integer value = map.get(nums[i]);
            if (value != null) {
                return new int[] { value, i };
            }
            map.put(target - nums[i], i);
        }
        return null;
    }
}
```


## Conclusion

If you are as passionate about data structures, algorithms, and LeetCode as I am, feel free to follow my LeetCode solutions repository on GitHub: [awesome-java-leetcode][ajl]



[title]: https://leetcode.com/problems/two-sum
[ajl]: https://github.com/Blankj/awesome-java-leetcode