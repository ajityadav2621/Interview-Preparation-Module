# Flatten Binary Tree to Linked List

## Problem
Given a binary tree, flatten it into a linked list in-place following the same order as a pre-order traversal.

## Solution
```java
public void flatten(TreeNode root) {
    if (root == null) return;
    
    TreeNode current = root;
    
    while (current != null) {
        if (current.left != null) {
            // Find rightmost node in left subtree
            TreeNode rightmost = current.left;
            while (rightmost.right != null) {
                rightmost = rightmost.right;
            }
            
            // Rearrange pointers
            rightmost.right = current.right;
            current.right = current.left;
            current.left = null;
        }
        
        current = current.right;
    }
}
```

## Time Complexity: O(n)
## Space Complexity: O(1)