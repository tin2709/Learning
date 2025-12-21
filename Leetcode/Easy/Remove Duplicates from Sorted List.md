

# [83. Remove Duplicates from Sorted List][title83]

## Description

Given a sorted linked list, delete all duplicates such that each element appears only *once*.

**Example 1:**

```
Input: 1->1->2
Output: 1->2
```

**Example 2:**

```
Input: 1->1->2->3->3
Output: 1->2->3
```

**Tags:** Linked List

## Analysis

The problem asks us to remove duplicate values from a sorted linked list. Since the list is already sorted, all duplicate values are guaranteed to be adjacent.

We can solve this by traversing the list once:
1. Compare the value of the current node with the value of the next node.
2. If they are the same, skip the next node by setting `current.next = current.next.next`.
3. If they are different, move the current pointer forward to the next node.
4. Continue until the end of the list is reached.

```java
/**
 * Definition for singly-linked list.
 * public class ListNode {
 *     int val;
 *     ListNode next;
 *     ListNode(int x) { val = x; }
 * }
 */
class Solution {
    public ListNode deleteDuplicates(ListNode head) {
        if (head == null || head.next == null) return head;
        
        ListNode curr = head;
        while (curr.next != null) {
            if (curr.next.val == curr.val) {
                // Duplicate found, skip the next node
                curr.next = curr.next.next;
            } else {
                // No duplicate, move pointer forward
                curr = curr.next;
            }
        }
        return head;
    }
}
```

## Conclusion

If you found these explanations helpful, you can follow my full LeetCode solutions repository on GitHub: [Awesome-Java-LeetCode][ajl]

[title83]: https://leetcode.com/problems/remove-duplicates-from-sorted-list
[ajl]: https://github.com/Blankj/awesome-java-leetcode