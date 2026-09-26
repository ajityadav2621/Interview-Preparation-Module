# Next Greater Element

## Problem
Given an array of values, for each element find the next greater element on the right.

## Solution
```java
public int[] nextGreaterElement(int[] nums) {
    int n = nums.length;
    int[] result = new int[n];
    Arrays.fill(result, -1);
    Deque<Integer> stack = new ArrayDeque<>(); // Store indices
    
    for (int i = 0; i < n; i++) {
        // While stack is not empty and current element is greater
        while (!stack.isEmpty() && nums[stack.peek()] < nums[i]) {
            int idx = stack.pollFirst();
            result[idx] = nums[i];
        }
        stack.offerFirst(i);
    }
    
    return result;
}
```

## Time Complexity: O(n)
## Space Complexity: O(n)