# Subarray Sum Equals K

## Problem
Given an integer array, count the number of subarrays whose sum equals K.

## Solution
```java
public int subarraySum(int[] nums, int k) {
    int count = 0;
    int sum = 0;
    Map<Integer, Integer> prefixSumCount = new HashMap<>();
    prefixSumCount.put(0, 1); // Base case: sum 0 appears once
    
    for (int num : nums) {
        sum += num;
        
        // If (sum - k) exists in map, we found subarrays
        if (prefixSumCount.containsKey(sum - k)) {
            count += prefixSumCount.get(sum - k);
        }
        
        // Update prefix sum count
        prefixSumCount.put(sum, prefixSumCount.getOrDefault(sum, 0) + 1);
    }
    
    return count;
}
```

## Time Complexity: O(n)
## Space Complexity: O(n)