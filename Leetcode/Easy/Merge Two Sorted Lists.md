

# [Merge Two Sorted Lists][title]

## Description

Merge two sorted linked lists and return it as a new list. The new list should be made by splicing together the nodes of the first two lists.

**Example:**

```
Input: 1->2->4, 1->3->4
Output: 1->1->2->3->4->4
```

**Tags:** Linked List


## Approach

The goal is to merge two sorted linked lists into a single new sorted linked list. To do this, we compare the nodes of the two lists starting from their heads. At each step, we point the `next` of our new list to the node with the smaller value. We then move the pointer of that list forward and repeat the process. 

Finally, when one of the lists reaches its end (`null`), we simply point the tail of the new list to the remaining portion of the list that hasn't finished yet.

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
    public ListNode mergeTwoLists(ListNode l1, ListNode l2) {
        // Use a dummy head to simplify the logic
        ListNode head = new ListNode(0);
        ListNode temp = head;
        
        while (l1 != null && l2 != null) {
            if (l1.val < l2.val) {
                temp.next = l1;
                l1 = l1.next;
            } else {
                temp.next = l2;
                l2 = l2.next;
            }
            temp = temp.next;
        }
        
        // Splicing the remaining part of the non-empty list
        temp.next = l1 != null ? l1 : l2;
        
        return head.next;
    }
}
```


## Conclusion

If you are as passionate about data structures, algorithms, and LeetCode as I am, feel free to follow my LeetCode solutions repository on GitHub: [awesome-java-leetcode][ajl]



[title]: https://leetcode.com/problems/merge-two-sorted-lists
[ajl]: https://github.com/Blankj/awesome-java-leetcode