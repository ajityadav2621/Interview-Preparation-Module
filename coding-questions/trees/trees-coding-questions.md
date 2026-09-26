# Trees Coding Questions

## 1. Binary Tree Traversals

```java
// Inorder: Left → Root → Right
public void inorder(TreeNode root, List<Integer> result) {
    if (root == null) return;
    inorder(root.left, result);
    result.add(root.val);
    inorder(root.right, result);
}

// Preorder: Root → Left → Right
public void preorder(TreeNode root, List<Integer> result) {
    if (root == null) return;
    result.add(root.val);
    preorder(root.left, result);
    preorder(root.right, result);
}

// Postorder: Left → Right → Root
public void postorder(TreeNode root, List<Integer> result) {
    if (root == null) return;
    postorder(root.left, result);
    postorder(root.right, result);
    result.add(root.val);
}

// Iterative inorder
public List<Integer> inorderIterative(TreeNode root) {
    List<Integer> result = new ArrayList<>();
    Stack<TreeNode> stack = new Stack<>();
    TreeNode curr = root;
    
    while (curr != null || !stack.isEmpty()) {
        while (curr != null) {
            stack.push(curr);
            curr = curr.left;
        }
        curr = stack.pop();
        result.add(curr.val);
        curr = curr.right;
    }
    return result;
}
```

---

## 2. Validate Binary Search Tree

```java
public boolean isValidBST(TreeNode root) {
    return validate(root, Long.MIN_VALUE, Long.MAX_VALUE);
}

private boolean validate(TreeNode node, long min, long max) {
    if (node == null) return true;
    if (node.val <= min || node.val >= max) return false;
    return validate(node.left, min, node.val) && 
           validate(node.right, node.val, max);
}
```

**Time**: O(n) | **Space**: O(h)

---

## 3. Lowest Common Ancestor

```java
public TreeNode lowestCommonAncestor(TreeNode root, TreeNode p, TreeNode q) {
    if (root == null || root == p || root == q) return root;
    
    TreeNode left = lowestCommonAncestor(root.left, p, q);
    TreeNode right = lowestCommonAncestor(root.right, p, q);
    
    if (left != null && right != null) return root;
    return left != null ? left : right;
}
```

**Time**: O(n) | **Space**: O(h)

---

## 4. Binary Tree Level Order Traversal

```java
public List<List<Integer>> levelOrder(TreeNode root) {
    List<List<Integer>> result = new ArrayList<>();
    if (root == null) return result;
    
    Queue<TreeNode> queue = new LinkedList<>();
    queue.offer(root);
    
    while (!queue.isEmpty()) {
        int levelSize = queue.size();
        List<Integer> level = new ArrayList<>();
        
        for (int i = 0; i < levelSize; i++) {
            TreeNode node = queue.poll();
            level.add(node.val);
            if (node.left != null) queue.offer(node.left);
            if (node.right != null) queue.offer(node.right);
        }
        result.add(level);
    }
    return result;
}
```

**Time**: O(n) | **Space**: O(n)

---

## 5. Construct Binary Tree from Preorder and Inorder

```java
public TreeNode buildTree(int[] preorder, int[] inorder) {
    Map<Integer, Integer> inorderMap = new HashMap<>();
    for (int i = 0; i < inorder.length; i++) {
        inorderMap.put(inorder[i], i);
    }
    return build(preorder, 0, preorder.length - 1,
                 inorder, 0, inorder.length - 1, inorderMap);
}

private TreeNode build(int[] preorder, int preStart, int preEnd,
                       int[] inorder, int inStart, int inEnd,
                       Map<Integer, Integer> map) {
    if (preStart > preEnd || inStart > inEnd) return null;
    
    TreeNode root = new TreeNode(preorder[preStart]);
    int inRoot = map.get(root.val);
    int leftSize = inRoot - inStart;
    
    root.left = build(preorder, preStart + 1, preStart + leftSize,
                      inorder, inStart, inRoot - 1, map);
    root.right = build(preorder, preStart + leftSize + 1, preEnd,
                       inorder, inRoot + 1, inEnd, map);
    return root;
}
```

**Time**: O(n) | **Space**: O(n)

---

## 6. Maximum Path Sum in Binary Tree

```java
private int maxSum = Integer.MIN_VALUE;

public int maxPathSum(TreeNode root) {
    maxPathSumHelper(root);
    return maxSum;
}

private int maxPathSumHelper(TreeNode node) {
    if (node == null) return 0;
    
    int left = Math.max(0, maxPathSumHelper(node.left));
    int right = Math.max(0, maxPathSumHelper(node.right));
    
    maxSum = Math.max(maxSum, node.val + left + right);
    
    return node.val + Math.max(left, right);
}
```

**Time**: O(n) | **Space**: O(h)

---

## 7. Serialize and Deserialize Binary Tree

```java
public class Codec {
    
    // Encode a tree to a single string
    public String serialize(TreeNode root) {
        if (root == null) return "null,";
        StringBuilder sb = new StringBuilder();
        serializeHelper(root, sb);
        return sb.toString();
    }
    
    private void serializeHelper(TreeNode node, StringBuilder sb) {
        if (node == null) {
            sb.append("null,");
            return;
        }
        sb.append(node.val).append(",");
        serializeHelper(node.left, sb);
        serializeHelper(node.right, sb);
    }
    
    // Decode a single string to tree
    public TreeNode deserialize(String data) {
        Queue<String> queue = new LinkedList<>(Arrays.asList(data.split(",")));
        return deserializeHelper(queue);
    }
    
    private TreeNode deserializeHelper(Queue<String> queue) {
        String val = queue.poll();
        if (val.equals("null")) return null;
        
        TreeNode node = new TreeNode(Integer.parseInt(val));
        node.left = deserializeHelper(queue);
        node.right = deserializeHelper(queue);
        return node;
    }
}
```

**Time**: O(n) | **Space**: O(n)

---

## 8. Trie (Prefix Tree)

```java
class Trie {
    private TrieNode root;
    
    class TrieNode {
        Map<Character, TrieNode> children;
        boolean isEndOfWord;
        
        TrieNode() {
            children = new HashMap<>();
            isEndOfWord = false;
        }
    }
    
    public Trie() {
        root = new TrieNode();
    }
    
    public void insert(String word) {
        TrieNode curr = root;
        for (char c : word.toCharArray()) {
            curr.children.putIfAbsent(c, new TrieNode());
            curr = curr.children.get(c);
        }
        curr.isEndOfWord = true;
    }
    
    public boolean search(String word) {
        TrieNode node = searchPrefix(word);
        return node != null && node.isEndOfWord;
    }
    
    public boolean startsWith(String prefix) {
        return searchPrefix(prefix) != null;
    }
    
    private TrieNode searchPrefix(String prefix) {
        TrieNode curr = root;
        for (char c : prefix.toCharArray()) {
            if (!curr.children.containsKey(c)) return null;
            curr = curr.children.get(c);
        }
        return curr;
    }
}
```

---

## Key Patterns

| Pattern | Use Case |
|---------|----------|
| DFS (recursive) | Tree traversal, path finding |
| DFS (iterative) | Avoid stack overflow |
| BFS | Level-order traversal, shortest path |
| Backtracking | Tree construction, path enumeration |
| Memoization | Overlapping subproblems |
| Divide and conquer | Tree construction, merge sort tree |

---

## 9. Flatten Binary Tree to Linked List (LeetCode 114)

Given a binary tree, flatten it into a linked list **in-place** following the **same order as a pre-order traversal**.

**Example:**
```
Input:
    1
   / \
  2   5
 / \   \
3   4   6

Output (right pointers only):
1 → 2 → 3 → 4 → 5 → 6
```

### Approach 1: Recursive (Modify in place — preferred for interview)
```java
public void flatten(TreeNode root) {
    if (root == null) return;
    
    // Flatten left and right subtrees first
    flatten(root.left);
    flatten(root.right);
    
    // Save the right subtree
    TreeNode rightSubtree = root.right;
    
    // Move flattened left subtree to the right
    root.right = root.left;
    root.left = null;
    
    // Find the new rightmost node and attach the saved right subtree
    TreeNode curr = root;
    while (curr.right != null) {
        curr = curr.right;
    }
    curr.right = rightSubtree;
}
```

**Time**: O(n) amortized — O(n²) worst case (skewed tree, due to the `while` walk in each call) | **Space**: O(h) recursion stack

### Approach 2: Iterative Using Stack (Pre-order Simulation)
```java
public void flatten(TreeNode root) {
    if (root == null) return;
    
    Deque<TreeNode> stack = new ArrayDeque<>();
    stack.push(root);
    
    while (!stack.isEmpty()) {
        TreeNode curr = stack.pop();
        
        // Push right first so left is processed first (stack LIFO)
        if (curr.right != null) stack.push(curr.right);
        if (curr.left != null) stack.push(curr.left);
        
        // Link to next node in pre-order
        if (!stack.isEmpty()) {
            curr.right = stack.peek();
        }
        curr.left = null;
    }
}
```

**Time**: O(n) | **Space**: O(h) stack

### Approach 3: Morris Traversal (O(1) Extra Space) — Most Elegant
```java
public void flatten(TreeNode root) {
    TreeNode curr = root;
    
    while (curr != null) {
        if (curr.left == null) {
            // No left subtree → just move right
            curr = curr.right;
        } else {
            // Find rightmost node in left subtree
            TreeNode rightmost = curr.left;
            while (rightmost.right != null) {
                rightmost = rightmost.right;
            }
            
            // Rewire: left subtree becomes right subtree
            rightmost.right = curr.right;
            curr.right = curr.left;
            curr.left = null;
            
            // Move to next node
            curr = curr.right;
        }
    }
}
```

**Time**: O(n) | **Space**: O(1) — no stack, no recursion

### Approach 4: Reverse Post-Order (Reverse Engineering)
Insight: if we traverse in **reverse pre-order** (right → left → root), the previous node visited is exactly the "next" node in the flattened list.

```java
private TreeNode prev = null;

public void flatten(TreeNode root) {
    if (root == null) return;
    
    flatten(root.right);   // process right first
    flatten(root.left);    // then left
    
    root.right = prev;     // point to previously visited node
    root.left = null;
    prev = root;           // update previous
}
```

**Time**: O(n) | **Space**: O(h) recursion

### Comparison Table

| Approach | Time | Space | Notes |
|----------|------|-------|-------|
| Recursive (modify) | O(n²) worst | O(h) | Easy to understand |
| Stack-based | O(n) | O(h) | Standard iterative |
| Morris Traversal | O(n) | **O(1)** | Optimal, advanced |
| Reverse Post-Order | O(n) | O(h) | Elegant, uses `prev` |

### Key Insights
- Pre-order: **root → left → right**
- The "next" node after root's left subtree is root's right subtree
- For O(1) space, Morris Traversal uses the rightmost node of left subtree as a temporary bridge

---
