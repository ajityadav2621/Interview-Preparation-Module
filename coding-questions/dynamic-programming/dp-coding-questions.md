# Dynamic Programming Coding Questions

## 1. Fibonacci Numbers

```java
// Naive recursive — O(2^n)
public int fib(int n) {
    if (n <= 1) return n;
    return fib(n - 1) + fib(n - 2);
}

// Memoized — O(n)
public int fibMemo(int n) {
    int[] memo = new int[n + 1];
    return fibMemo(n, memo);
}

private int fibMemo(int n, int[] memo) {
    if (n <= 1) return n;
    if (memo[n] != 0) return memo[n];
    memo[n] = fibMemo(n - 1, memo) + fibMemo(n - 2, memo);
    return memo[n];
}

// Bottom-up — O(n)
public int fibDP(int n) {
    if (n <= 1) return n;
    int[] dp = new int[n + 1];
    dp[1] = 1;
    for (int i = 2; i <= n; i++) {
        dp[i] = dp[i - 1] + dp[i - 2];
    }
    return dp[n];
}

// Space optimized — O(n) time, O(1) space
public int fibOptimized(int n) {
    if (n <= 1) return n;
    int a = 0, b = 1;
    for (int i = 2; i <= n; i++) {
        int c = a + b;
        a = b;
        b = c;
    }
    return b;
}
```

---

## 2. Climbing Stairs

```java
// You can climb 1 or 2 steps at a time. How many ways to reach n steps?
public int climbStairs(int n) {
    if (n <= 2) return n;
    int a = 1, b = 2;
    for (int i = 3; i <= n; i++) {
        int c = a + b;
        a = b;
        b = c;
    }
    return b;
}
```

**Time**: O(n) | **Space**: O(1)

---

## 3. Coin Change

```java
// Find minimum coins needed to make amount
public int coinChange(int[] coins, int amount) {
    int[] dp = new int[amount + 1];
    Arrays.fill(dp, amount + 1);  // Initialize with impossible value
    dp[0] = 0;
    
    for (int i = 1; i <= amount; i++) {
        for (int coin : coins) {
            if (coin <= i) {
                dp[i] = Math.min(dp[i], dp[i - coin] + 1);
            }
        }
    }
    
    return dp[amount] > amount ? -1 : dp[amount];
}
```

**Time**: O(amount × coins) | **Space**: O(amount)

---

## 4. Longest Common Subsequence

```java
public int longestCommonSubsequence(String text1, String text2) {
    int m = text1.length();
    int n = text2.length();
    int[][] dp = new int[m + 1][n + 1];
    
    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            if (text1.charAt(i - 1) == text2.charAt(j - 1)) {
                dp[i][j] = dp[i - 1][j - 1] + 1;
            } else {
                dp[i][j] = Math.max(dp[i - 1][j], dp[i][j - 1]);
            }
        }
    }
    
    return dp[m][n];
}
```

**Time**: O(m × n) | **Space**: O(m × n)

---

## 5. Longest Increasing Subsequence

```java
// O(n²) DP approach
public int lengthOfLIS(int[] nums) {
    if (nums.length == 0) return 0;
    
    int[] dp = new int[nums.length];
    Arrays.fill(dp, 1);
    
    for (int i = 1; i < nums.length; i++) {
        for (int j = 0; j < i; j++) {
            if (nums[i] > nums[j]) {
                dp[i] = Math.max(dp[i], dp[j] + 1);
            }
        }
    }
    
    return Arrays.stream(dp).max().getAsInt();
}

// O(n log n) binary search approach
public int lengthOfLISBinarySearch(int[] nums) {
    int[] tails = new int[nums.length];
    int size = 0;
    
    for (int num : nums) {
        int left = 0, right = size;
        while (left < right) {
            int mid = left + (right - left) / 2;
            if (tails[mid] < num) {
                left = mid + 1;
            } else {
                right = mid;
            }
        }
        tails[left] = num;
        if (left == size) size++;
    }
    
    return size;
}
```

**Time**: O(n²) or O(n log n) | **Space**: O(n)

---

## 6. Edit Distance

```java
public int minDistance(String word1, String word2) {
    int m = word1.length();
    int n = word2.length();
    int[][] dp = new int[m + 1][n + 1];
    
    // Initialize base cases
    for (int i = 0; i <= m; i++) dp[i][0] = i;  // Delete all
    for (int j = 0; j <= n; j++) dp[0][j] = j;  // Insert all
    
    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            if (word1.charAt(i - 1) == word2.charAt(j - 1)) {
                dp[i][j] = dp[i - 1][j - 1];  // No operation needed
            } else {
                dp[i][j] = 1 + Math.min(
                    dp[i - 1][j],      // Delete
                    Math.min(
                        dp[i][j - 1],  // Insert
                        dp[i - 1][j - 1]  // Replace
                    )
                );
            }
        }
    }
    
    return dp[m][n];
}
```

**Time**: O(m × n) | **Space**: O(m × n)

---

## 7. House Robber

```java
public int rob(int[] nums) {
    if (nums.length == 0) return 0;
    if (nums.length == 1) return nums[0];
    
    int[] dp = new int[nums.length];
    dp[0] = nums[0];
    dp[1] = Math.max(nums[0], nums[1]);
    
    for (int i = 2; i < nums.length; i++) {
        dp[i] = Math.max(dp[i - 1], dp[i - 2] + nums[i]);
    }
    
    return dp[nums.length - 1];
}

// Space optimized
public int robOptimized(int[] nums) {
    if (nums.length == 0) return 0;
    if (nums.length == 1) return nums[0];
    
    int prev2 = 0;
    int prev1 = nums[0];
    
    for (int i = 1; i < nums.length; i++) {
        int curr = Math.max(prev1, prev2 + nums[i]);
        prev2 = prev1;
        prev1 = curr;
    }
    
    return prev1;
}
```

**Time**: O(n) | **Space**: O(1)

---

## 8. Unique Paths

```java
public int uniquePaths(int m, int n) {
    int[][] dp = new int[m][n];
    
    // Initialize first row and column
    for (int i = 0; i < m; i++) dp[i][0] = 1;
    for (int j = 0; j < n; j++) dp[0][j] = 1;
    
    for (int i = 1; i < m; i++) {
        for (int j = 1; j < n; j++) {
            dp[i][j] = dp[i - 1][j] + dp[i][j - 1];
        }
    }
    
    return dp[m - 1][n - 1];
}

// Space optimized
public int uniquePathsOptimized(int m, int n) {
    int[] dp = new int[n];
    Arrays.fill(dp, 1);
    
    for (int i = 1; i < m; i++) {
        for (int j = 1; j < n; j++) {
            dp[j] += dp[j - 1];
        }
    }
    
    return dp[n - 1];
}
```

**Time**: O(m × n) | **Space**: O(n)

---

## 9. Word Break

```java
public boolean wordBreak(String s, List<String> wordDict) {
    Set<String> wordSet = new HashSet<>(wordDict);
    boolean[] dp = new boolean[s.length() + 1];
    dp[0] = true;  // Empty string is valid
    
    for (int i = 1; i <= s.length(); i++) {
        for (int j = 0; j < i; j++) {
            if (dp[j] && wordSet.contains(s.substring(j, i))) {
                dp[i] = true;
                break;
            }
        }
    }
    
    return dp[s.length()];
}
```

**Time**: O(n³) | **Space**: O(n)

---

## 10. Decode Ways

```java
public int numDecodings(String s) {
    if (s.charAt(0) == '0') return 0;
    
    int n = s.length();
    int[] dp = new int[n + 1];
    dp[0] = 1;
    dp[1] = 1;
    
    for (int i = 2; i <= n; i++) {
        // Check single digit
        if (s.charAt(i - 1) != '0') {
            dp[i] += dp[i - 1];
        }
        
        // Check two digits
        int twoDigit = Integer.parseInt(s.substring(i - 2, i));
        if (twoDigit >= 10 && twoDigit <= 26) {
            dp[i] += dp[i - 2];
        }
    }
    
    return dp[n];
}
```

**Time**: O(n) | **Space**: O(n)

---

## Key Patterns

| Pattern | Use Case |
|---------|----------|
| Memoization | Top-down with caching |
| Tabulation | Bottom-up DP table |
| State reduction | Reduce 2D to 1D |
| Prefix sums | Range sum queries |
| Sliding window | Variable-size window DP |
| Bitmask DP | Subset problems |
| Tree DP | Tree-based problems |
