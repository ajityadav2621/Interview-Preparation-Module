# LinkedList Coding Questions

## 1. Reverse a Linked List

```java
// Iterative approach
public ListNode reverseList(ListNode head) {
    ListNode prev = null;
    ListNode curr = head;
    
    while (curr != null) {
        ListNode next = curr.next;
        curr.next = prev;
        prev = curr;
        curr = next;
    }
    return prev;
}

// Recursive approach
public ListNode reverseListRecursive(ListNode head) {
    if (head == null || head.next == null) return head;
    
    ListNode reversed = reverseListRecursive(head.next);
    head.next.next = head;
    head.next = null;
    return reversed;
}
```

**Time**: O(n) | **Space**: O(1) iterative, O(n) recursive

---

## 2. Detect Cycle in a Linked List

```java
// Floyd's Cycle Detection Algorithm (Tortoise and Hare)
public boolean hasCycle(ListNode head) {
    if (head == null || head.next == null) return false;
    
    ListNode slow = head;
    ListNode fast = head;
    
    while (fast != null && fast.next != null) {
        slow = slow.next;
        fast = fast.next.next;
        
        if (slow == fast) return true;  // Cycle detected
    }
    return false;
}

// Find the start of the cycle
public ListNode detectCycle(ListNode head) {
    if (head == null || head.next == null) return null;
    
    ListNode slow = head;
    ListNode fast = head;
    
    // Detect cycle
    while (fast != null && fast.next != null) {
        slow = slow.next;
        fast = fast.next.next;
        if (slow == fast) break;
    }
    
    if (fast == null || fast.next == null) return null;  // No cycle
    
    // Find cycle start
    slow = head;
    while (slow != fast) {
        slow = slow.next;
        fast = fast.next;
    }
    return slow;
}
```

**Time**: O(n) | **Space**: O(1)

---

## 3. Merge Two Sorted Lists

```java
public ListNode mergeTwoLists(ListNode l1, ListNode l2) {
    ListNode dummy = new ListNode(0);
    ListNode curr = dummy;
    
    while (l1 != null && l2 != null) {
        if (l1.val <= l2.val) {
            curr.next = l1;
            l1 = l1.next;
        } else {
            curr.next = l2;
            l2 = l2.next;
        }
        curr = curr.next;
    }
    
    curr.next = l1 != null ? l1 : l2;
    return dummy.next;
}
```

**Time**: O(n + m) | **Space**: O(1)

---

## 4. Remove Nth Node From End of List

```java
public ListNode removeNthFromEnd(ListNode head, int n) {
    ListNode dummy = new ListNode(0);
    dummy.next = head;
    
    ListNode first = dummy;
    ListNode second = dummy;
    
    // Move first n+1 steps ahead
    for (int i = 0; i <= n; i++) {
        first = first.next;
    }
    
    // Move both until first reaches the end
    while (first != null) {
        first = first.next;
        second = second.next;
    }
    
    // Remove the nth node
    second.next = second.next.next;
    
    return dummy.next;
}
```

**Time**: O(n) | **Space**: O(1)

---

## 5. Add Two Numbers (Linked List Representation)

```java
public ListNode addTwoNumbers(ListNode l1, ListNode l2) {
    ListNode dummy = new ListNode(0);
    ListNode curr = dummy;
    int carry = 0;
    
    while (l1 != null || l2 != null || carry != 0) {
        int sum = carry;
        if (l1 != null) {
            sum += l1.val;
            l1 = l1.next;
        }
        if (l2 != null) {
            sum += l2.val;
            l2 = l2.next;
        }
        
        carry = sum / 10;
        curr.next = new ListNode(sum % 10);
        curr = curr.next;
    }
    
    return dummy.next;
}
```

**Time**: O(max(n, m)) | **Space**: O(max(n, m))

---

## 6. Intersection of Two Linked Lists

```java
public ListNode getIntersectionNode(ListNode headA, ListNode headB) {
    if (headA == null || headB == null) return null;
    
    ListNode a = headA;
    ListNode b = headB;
    
    // After at most 2 passes, both pointers will be at the intersection
    // or both will be null
    while (a != b) {
        a = a == null ? headB : a.next;
        b = b == null ? headA : b.next;
    }
    
    return a;
}
```

**Time**: O(n + m) | **Space**: O(1)

---

## 7. Copy List with Random Pointer

```java
public Node copyRandomList(Node head) {
    if (head == null) return null;
    
    // Step 1: Create copy nodes and interleave them
    Node curr = head;
    while (curr != null) {
        Node copy = new Node(curr.val);
        copy.next = curr.next;
        curr.next = copy;
        curr = copy.next;
    }
    
    // Step 2: Assign random pointers
    curr = head;
    while (curr != null) {
        if (curr.random != null) {
            curr.next.random = curr.random.next;
        }
        curr = curr.next.next;
    }
    
    // Step 3: Separate the two lists
    curr = head;
    Node dummy = new Node(0);
    Node copyCurr = dummy;
    
    while (curr != null) {
        copyCurr.next = curr.next;
        copyCurr = copyCurr.next;
        curr.next = curr.next.next;
        curr = curr.next;
    }
    
    return dummy.next;
}
```

**Time**: O(n) | **Space**: O(1)

---

## 8. Reverse Nodes in k-Group

```java
public ListNode reverseKGroup(ListNode head, int k) {
    if (head == null || k == 1) return head;
    
    ListNode dummy = new ListNode(0);
    dummy.next = head;
    ListNode curr = head;
    int length = 0;
    
    while (curr != null) {
        length++;
        curr = curr.next;
    }
    
    curr = head;
    ListNode prev = dummy;
    
    while (length >= k) {
        ListNode temp = curr;
        for (int i = 1; i < k; i++) {
            curr = curr.next;
        }
        
        ListNode next = curr.next;
        reverse(temp, curr);
        prev.next = curr;
        temp.next = next;
        
        prev = temp;
        curr = next;
        length -= k;
    }
    
    return dummy.next;
}

private void reverse(ListNode start, ListNode end) {
    ListNode prev = null;
    ListNode curr = start;
    while (prev != end) {
        ListNode next = curr.next;
        curr.next = prev;
        prev = curr;
        curr = next;
    }
}
```

**Time**: O(n) | **Space**: O(1)

---

## Key Patterns

| Pattern | Use Case |
|---------|----------|
| Two pointers | Cycle detection, intersection |
| Dummy node | Simplify edge cases |
| Fast/slow pointers | Find middle, detect cycle |
| In-place reversal | Reverse list, reverse k-group |
| Interleaving | Copy with random pointer |
| Length calculation | k-group reversal |
