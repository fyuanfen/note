合并 k 个排序链表，返回合并后的排序链表。请分析和描述算法的复杂度。

示例:

输入:

```
[
  1->4->5,
  1->3->4,
  2->6
]
```

输出: `1->1->2->3->4->4->5->6`

```js
/**
 * Definition for singly-linked list.
 * function ListNode(val) {
 *     this.val = val;
 *     this.next = null;
 * }
 */
/**
 * @param {ListNode[]} lists
 * @return {ListNode}
 */

var partition = function(lists, from, to) {
  if (from === to) {
    return lists[from];
  }

  if (from < to) {
    var middle = parseInt((from + to) / 2);
    var l1 = partition(lists, from, middle);
    var l2 = partition(lists, middle + 1, to);
    return merge(l1, l2);
  } else {
    return null;
  }
};

var mergeKLists = function(lists) {
  return partition(lists, 0, lists.length - 1);
};

var merge = function(l1, l2) {
  if (!l1) {
    return l2;
  }
  if (!l2) {
    return l1;
  }
  if (l1.val < l2.val) {
    l1.next = merge(l1.next, l2);
    return l1;
  } else {
    l2.next = merge(l2.next, l1);
    return l2;
  }
};
```
