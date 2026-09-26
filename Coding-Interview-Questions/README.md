# Coding Interview Questions - Complete Guide

## Table of Contents

### Arrays and Strings
1. Two Sum
2. Valid Parentheses
3. Reverse String
4. Palindrome Check
5. Find Duplicate Characters
6. Maximum Subarray Sum (Kadane's Algorithm)
7. Longest Substring Without Repeating Characters
8. String to Integer (atoi)

### Linked Lists
1. Merge Two Sorted Lists
2. Reverse Linked List
3. Detect Cycle in Linked List
4. Remove Nth Node From End
5. Add Two Numbers

### Trees
1. Binary Tree Inorder Traversal
2. Binary Tree Maximum Path Sum
3. Validate Binary Search Tree
4. Symmetric Tree
5. Level Order Traversal

### Graphs
1. Word Ladder
2. Number of Islands
3. Clone Graph
4. Course Schedule

### Dynamic Programming
1. Fibonacci Series
2. Longest Common Subsequence
3. Longest Increasing Subsequence
4. Coin Change
5. House Robber

### Sorting and Searching
1. Binary Search
2. Merge Sort
3. Quick Sort
4. Search in Rotated Sorted Array
5. Find First and Last Position of Element

### Concurrency
1. Producer-Consumer Pattern
2. Print Alternating Threads
3. Thread-Safe Counter
4. Deadlock Detection
5. ThreadPool Implementation

### System Design
1. Design URL Shortener
2. Design Twitter Feed
3. Design Chat Application
4. Design File System
5. Design Parking Lot System

## Problem Templates

### Two Pointers Pattern
```java
// Template for two pointers
int left = 0, right = arr.length - 1;
while (left < right) {
    // Process
    left++;
    right--;
}
```

### Sliding Window Pattern
```java
// Template for sliding window
int left = 0, maxLen = 0;
Map<Character, Integer> window = new HashMap<>();
for (int right = 0; right < s.length(); right++) {
    // Expand window
    // Shrink if needed
    // Update maxLen
}
```

### Fast and Slow Pointers
```java
// Template for cycle detection
ListNode slow = head, fast = head;
while (fast != null && fast.next != null) {
    slow = slow.next;
    fast = fast.next.next;
    if (slow == fast) {
        // Cycle detected
    }
}
```

## Interview Tips

1. **Clarify Requirements**: Ask about input/output constraints
2. **Discuss Approach**: Explain your thought process
3. **Analyze Complexity**: Time and space complexity
4. **Write Clean Code**: Proper naming, modular functions
5. **Test Edge Cases**: Empty input, single element, large input
6. **Optimize**: Start with brute force, then optimize

## Common Patterns

- Hash Table for O(1) lookups
- Two Pointers for sorted arrays
- Sliding Window for subarrays/substrings
- DFS/BFS for tree/graph traversal
- Dynamic Programming for optimization problems
- Binary Search for sorted arrays

Good luck with your interview preparation!