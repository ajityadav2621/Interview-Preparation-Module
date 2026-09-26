# Arrays Coding Questions

## 1. Two Sum

```java
// Given an array of integers, return indices of the two numbers that add up to target.
public int[] twoSum(int[] nums, int target) {
    Map<Integer, Integer> map = new HashMap<>();
    for (int i = 0; i < nums.length; i++) {
        int complement = target - nums[i];
        if (map.containsKey(complement)) {
            return new int[]{map.get(complement), i};
        }
        map.put(nums[i], i);
    }
    return new int[0];
}
```

**Time**: O(n) | **Space**: O(n)

---

## 2. Three Sum

```java
public List<List<Integer>> threeSum(int[] nums) {
    List<List<Integer>> result = new ArrayList<>();
    Arrays.sort(nums);
    
    for (int i = 0; i < nums.length - 2; i++) {
        if (i > 0 && nums[i] == nums[i - 1]) continue;  // Skip duplicates
        
        int left = i + 1, right = nums.length - 1;
        while (left < right) {
            int sum = nums[i] + nums[left] + nums[right];
            if (sum == 0) {
                result.add(Arrays.asList(nums[i], nums[left], nums[right]));
                left++;
                right--;
                while (left < right && nums[left] == nums[left - 1]) left++;
                while (left < right && nums[right] == nums[right + 1]) right--;
            } else if (sum < 0) {
                left++;
            } else {
                right--;
            }
        }
    }
    return result;
}
```

**Time**: O(n²) | **Space**: O(1)

---

## 3. Container With Most Water

```java
public int maxArea(int[] height) {
    int left = 0, right = height.length - 1;
    int maxArea = 0;
    
    while (left < right) {
        int width = right - left;
        int area = width * Math.min(height[left], height[right]);
        maxArea = Math.max(maxArea, area);
        
        if (height[left] < height[right]) {
            left++;
        } else {
            right--;
        }
    }
    return maxArea;
}
```

**Time**: O(n) | **Space**: O(1)

---

## 4. Search in Rotated Sorted Array

```java
public int search(int[] nums, int target) {
    int left = 0, right = nums.length - 1;
    
    while (left <= right) {
        int mid = left + (right - left) / 2;
        
        if (nums[mid] == target) return mid;
        
        // Left half is sorted
        if (nums[left] <= nums[mid]) {
            if (nums[left] <= target && target < nums[mid]) {
                right = mid - 1;
            } else {
                left = mid + 1;
            }
        } else {  // Right half is sorted
            if (nums[mid] < target && target <= nums[right]) {
                left = mid + 1;
            } else {
                right = mid - 1;
            }
        }
    }
    return -1;
}
```

**Time**: O(log n) | **Space**: O(1)

---

## 5. Find First and Last Position in Sorted Array

```java
public int[] searchRange(int[] nums, int target) {
    int[] result = {-1, -1};
    result[0] = findBound(nums, target, true);   // First position
    result[1] = findBound(nums, target, false);  // Last position
    return result;
}

private int findBound(int[] nums, int target, boolean isFirst) {
    int left = 0, right = nums.length - 1;
    int bound = -1;
    
    while (left <= right) {
        int mid = left + (right - left) / 2;
        
        if (nums[mid] == target) {
            bound = mid;
            if (isFirst) {
                right = mid - 1;  // Search left
            } else {
                left = mid + 1;   // Search right
            }
        } else if (nums[mid] < target) {
            left = mid + 1;
        } else {
            right = mid - 1;
        }
    }
    return bound;
}
```

**Time**: O(log n) | **Space**: O(1)

---

## 6. Merge Intervals

```java
public int[][] merge(int[][] intervals) {
    if (intervals.length <= 1) return intervals;
    
    Arrays.sort(intervals, (a, b) -> Integer.compare(a[0], b[0]));
    
    List<int[]> merged = new ArrayList<>();
    int[] current = intervals[0];
    
    for (int i = 1; i < intervals.length; i++) {
        if (current[1] >= intervals[i][0]) {
            current[1] = Math.max(current[1], intervals[i][1]);
        } else {
            merged.add(current);
            current = intervals[i];
        }
    }
    merged.add(current);
    
    return merged.toArray(new int[0][]);
}
```

**Time**: O(n log n) | **Space**: O(n)

---

## 7. Insert Interval

```java
public int[][] insert(int[][] intervals, int[] newInterval) {
    List<int[]> result = new ArrayList<>();
    int i = 0;
    
    // Add all intervals before newInterval
    while (i < intervals.length && intervals[i][1] < newInterval[0]) {
        result.add(intervals[i++]);
    }
    
    // Merge overlapping intervals
    while (i < intervals.length && intervals[i][0] <= newInterval[1]) {
        newInterval[0] = Math.min(newInterval[0], intervals[i][0]);
        newInterval[1] = Math.max(newInterval[1], intervals[i][1]);
        i++;
    }
    result.add(newInterval);
    
    // Add remaining intervals
    while (i < intervals.length) {
        result.add(intervals[i++]);
    }
    
    return result.toArray(new int[0][]);
}
```

**Time**: O(n) | **Space**: O(n)

---

## 8. Rotate Image (Matrix)

```java
public void rotate(int[][] matrix) {
    int n = matrix.length;
    
    // Transpose
    for (int i = 0; i < n; i++) {
        for (int j = i; j < n; j++) {
            int temp = matrix[i][j];
            matrix[i][j] = matrix[j][i];
            matrix[j][i] = temp;
        }
    }
    
    // Reverse each row
    for (int i = 0; i < n; i++) {
        for (int j = 0; j < n / 2; j++) {
            int temp = matrix[i][j];
            matrix[i][j] = matrix[i][n - 1 - j];
            matrix[i][n - 1 - j] = temp;
        }
    }
}
```

**Time**: O(n²) | **Space**: O(1)

---

## 9. Spiral Matrix

```java
public List<Integer> spiralOrder(int[][] matrix) {
    List<Integer> result = new ArrayList<>();
    if (matrix.length == 0) return result;
    
    int top = 0, bottom = matrix.length - 1;
    int left = 0, right = matrix[0].length - 1;
    
    while (top <= bottom && left <= right) {
        // Traverse right
        for (int i = left; i <= right; i++) result.add(matrix[top][i]);
        top++;
        
        // Traverse down
        for (int i = top; i <= bottom; i++) result.add(matrix[i][right]);
        right--;
        
        // Traverse left
        if (top <= bottom) {
            for (int i = right; i >= left; i--) result.add(matrix[bottom][i]);
            bottom--;
        }
        
        // Traverse up
        if (left <= right) {
            for (int i = bottom; i >= top; i--) result.add(matrix[i][left]);
            left++;
        }
    }
    return result;
}
```

**Time**: O(m × n) | **Space**: O(1)

---

## 10. Product of Array Except Self

```java
public int[] productExceptSelf(int[] nums) {
    int n = nums.length;
    int[] result = new int[n];
    
    // Left products
    int leftProduct = 1;
    for (int i = 0; i < n; i++) {
        result[i] = leftProduct;
        leftProduct *= nums[i];
    }
    
    // Right products
    int rightProduct = 1;
    for (int i = n - 1; i >= 0; i--) {
        result[i] *= rightProduct;
        rightProduct *= nums[i];
    }
    
    return result;
}
```

**Time**: O(n) | **Space**: O(1) (excluding output)

---

## Key Patterns

| Pattern | Use Case |
|---------|----------|
| Two pointers | Sorted array problems, sliding window |
| Binary search | Search in sorted/rotated arrays |
| Prefix sums | Range sum queries |
| Sliding window | Subarray problems |
| Matrix traversal | Spiral, rotation, transpose |
| Sorting + greedy | Interval problems |

---

## 11. Sliding Window Maximum (LeetCode 239)

Given an integer array `nums` and a window size `K`, return the maximum element in every sliding window.

**Example:** `nums = [1,3,-1,-3,5,3,6,7]`, `K = 3` → `[3,3,5,5,6,7]`

```java
public int[] maxSlidingWindow(int[] nums, int k) {
    if (nums == null || nums.length == 0 || k == 0) return new int[0];
    
    int n = nums.length;
    int[] result = new int[n - k + 1];
    Deque<Integer> deque = new ArrayDeque<>();   // stores indices
    
    for (int i = 0; i < n; i++) {
        // 1. Remove indices outside current window
        while (!deque.isEmpty() && deque.peekFirst() <= i - k) {
            deque.pollFirst();
        }
        
        // 2. Remove smaller elements from the back
        while (!deque.isEmpty() && nums[deque.peekLast()] < nums[i]) {
            deque.pollLast();
        }
        
        deque.offerLast(i);
        
        // 3. Add to result once window is fully formed
        if (i >= k - 1) {
            result[i - k + 1] = nums[deque.peekFirst()];
        }
    }
    return result;
}
```

**Time**: O(n) — each element added/removed at most once | **Space**: O(k)

**Key Insight:** Monotonic deque — always keeps indices of potential maximums in decreasing order.

---

## 12. Next Greater Element (LeetCode 496)

Given an array, for each element find the next greater element on the right. If none exists, return -1.

**Example:** `[4, 5, 2, 10, 8]` → `[5, 10, 10, -1, -1]`

```java
public int[] nextGreaterElement(int[] arr) {
    int n = arr.length;
    int[] result = new int[n];
    Arrays.fill(result, -1);
    Deque<Integer> stack = new ArrayDeque<>();   // indices of elements waiting for next greater
    
    for (int i = 0; i < n; i++) {
        // Pop elements smaller than current — current is their "next greater"
        while (!stack.isEmpty() && arr[stack.peek()] < arr[i]) {
            int idx = stack.pop();
            result[idx] = arr[i];
        }
        stack.push(i);
    }
    // Remaining elements have no greater → already -1
    return result;
}
```

**Time**: O(n) | **Space**: O(n)

**Variant — Circular Array (LeetCode 503):**
```java
public int[] nextGreaterElementsCircular(int[] nums) {
    int n = nums.length;
    int[] result = new int[n];
    Arrays.fill(result, -1);
    Deque<Integer> stack = new ArrayDeque<>();
    
    // Iterate twice to simulate circular
    for (int i = 0; i < 2 * n; i++) {
        int idx = i % n;
        while (!stack.isEmpty() && nums[stack.peek()] < nums[idx]) {
            result[stack.pop()] = nums[idx];
        }
        if (i < n) stack.push(idx);
    }
    return result;
}
```

---

## 13. Subarray Sum Equals K (LeetCode 560)

Given an integer array `nums` and integer `K`, count the number of continuous subarrays whose sum equals `K`.

**Example:** `nums = [1,1,1]`, `K = 2` → `2` (subarrays: `[1,1]` at positions 0-1 and 1-2)

```java
public int subarraySum(int[] nums, int k) {
    Map<Integer, Integer> prefixSumCount = new HashMap<>();
    prefixSumCount.put(0, 1);   // empty prefix
    
    int count = 0;
    int prefixSum = 0;
    
    for (int num : nums) {
        prefixSum += num;
        // If (prefixSum - k) seen before, those subarrays end here with sum k
        count += prefixSumCount.getOrDefault(prefixSum - k, 0);
        prefixSumCount.merge(prefixSum, 1, Integer::sum);
    }
    return count;
}
```

**Time**: O(n) | **Space**: O(n)

**Key Insight:** If `prefix[j] - prefix[i] = k`, then subarray `nums[i+1..j]` has sum `k`. Count occurrences of each prefix sum.

**Variants:**
- Subarray sum divisible by K → `prefix[j] % k == prefix[i] % k`
- Longest subarray with sum K → track first occurrence index

---
