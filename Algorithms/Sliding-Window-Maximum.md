# Sliding Window Maximum

## Problem
Given an integer array and a window size K, return the maximum element in every sliding window.

## Solution
```java
public int[] slidingWindowMaximum(int[] nums, int k) {
    if (nums == null || nums.length == 0 || k <= 0) {
        return new int[0];
    }
    
    int n = nums.length;
    int[] result = new int[n - k + 1];
    Deque<Integer> deque = new ArrayDeque<>(); // Store indices
    
    for (int i = 0; i < n; i++) {
        // Remove indices outside current window
        while (!deque.isEmpty() && deque.peekFirst() < i - k + 1) {
            deque.pollFirst();
        }
        
        // Remove smaller elements from back
        while (!deque.isEmpty() && nums[deque.peekLast()] < nums[i]) {
            deque.pollLast();
        }
        
        // Add current index
        deque.offerLast(i);
        
        // Add maximum to result when window is full
        if (i >= k - 1) {
            result[i - k + 1] = nums[deque.peekFirst()];
        }
    }
    
    return result;
}
```

## Time Complexity: O(n)
## Space Complexity: O(k)