# Graphs Coding Questions

## 1. BFS (Breadth-First Search)

```java
public List<Integer> bfs(int[][] graph, int start) {
    List<Integer> result = new ArrayList<>();
    boolean[] visited = new boolean[graph.length];
    Queue<Integer> queue = new LinkedList<>();
    
    queue.offer(start);
    visited[start] = true;
    
    while (!queue.isEmpty()) {
        int node = queue.poll();
        result.add(node);
        
        for (int neighbor : graph[node]) {
            if (!visited[neighbor]) {
                visited[neighbor] = true;
                queue.offer(neighbor);
            }
        }
    }
    return result;
}
```

**Time**: O(V + E) | **Space**: O(V)

---

## 2. DFS (Depth-First Search)

```java
// Recursive
public void dfsRecursive(int[][] graph, int node, boolean[] visited, List<Integer> result) {
    visited[node] = true;
    result.add(node);
    
    for (int neighbor : graph[node]) {
        if (!visited[neighbor]) {
            dfsRecursive(graph, neighbor, visited, result);
        }
    }
}

// Iterative
public List<Integer> dfsIterative(int[][] graph, int start) {
    List<Integer> result = new ArrayList<>();
    boolean[] visited = new boolean[graph.length];
    Stack<Integer> stack = new Stack<>();
    
    stack.push(start);
    
    while (!stack.isEmpty()) {
        int node = stack.pop();
        if (visited[node]) continue;
        
        visited[node] = true;
        result.add(node);
        
        for (int neighbor : graph[node]) {
            if (!visited[neighbor]) {
                stack.push(neighbor);
            }
        }
    }
    return result;
}
```

**Time**: O(V + E) | **Space**: O(V)

---

## 3. Number of Islands

```java
public int numIslands(char[][] grid) {
    if (grid.length == 0) return 0;
    
    int rows = grid.length;
    int cols = grid[0].length;
    int islands = 0;
    
    for (int i = 0; i < rows; i++) {
        for (int j = 0; j < cols; j++) {
            if (grid[i][j] == '1') {
                islands++;
                dfs(grid, i, j, rows, cols);
            }
        }
    }
    return islands;
}

private void dfs(char[][] grid, int i, int j, int rows, int cols) {
    if (i < 0 || j < 0 || i >= rows || j >= cols || grid[i][j] == '0') return;
    
    grid[i][j] = '0';  // Mark as visited
    
    dfs(grid, i + 1, j, rows, cols);
    dfs(grid, i - 1, j, rows, cols);
    dfs(grid, i, j + 1, rows, cols);
    dfs(grid, i, j - 1, rows, cols);
}
```

**Time**: O(m × n) | **Space**: O(m × n)

---

## 4. Course Schedule (Topological Sort)

```java
public boolean canFinish(int numCourses, int[][] prerequisites) {
    // Build adjacency list and in-degree array
    List<Integer>[] graph = new ArrayList[numCourses];
    int[] inDegree = new int[numCourses];
    
    for (int i = 0; i < numCourses; i++) {
        graph[i] = new ArrayList<>();
    }
    
    for (int[] prereq : prerequisites) {
        graph[prereq[1]].add(prereq[0]);
        inDegree[prereq[0]]++;
    }
    
    // Kahn's algorithm (BFS-based topological sort)
    Queue<Integer> queue = new LinkedList<>();
    for (int i = 0; i < numCourses; i++) {
        if (inDegree[i] == 0) {
            queue.offer(i);
        }
    }
    
    int count = 0;
    while (!queue.isEmpty()) {
        int course = queue.poll();
        count++;
        
        for (int neighbor : graph[course]) {
            inDegree[neighbor]--;
            if (inDegree[neighbor] == 0) {
                queue.offer(neighbor);
            }
        }
    }
    
    return count == numCourses;
}
```

**Time**: O(V + E) | **Space**: O(V + E)

---

## 5. Clone Graph

```java
public Node cloneGraph(Node node) {
    if (node == null) return null;
    
    Map<Node, Node> cloned = new HashMap<>();
    Queue<Node> queue = new LinkedList<>();
    
    // Clone the first node
    cloned.put(node, new Node(node.val));
    queue.offer(node);
    
    while (!queue.isEmpty()) {
        Node curr = queue.poll();
        Node clone = cloned.get(curr);
        
        for (Node neighbor : curr.neighbors) {
            if (!cloned.containsKey(neighbor)) {
                cloned.put(neighbor, new Node(neighbor.val));
                queue.offer(neighbor);
            }
            clone.neighbors.add(cloned.get(neighbor));
        }
    }
    
    return cloned.get(node);
}
```

**Time**: O(V + E) | **Space**: O(V)

---

## 6. Shortest Path in Binary Matrix

```java
public int shortestPathBinaryMatrix(int[][] grid) {
    if (grid[0][0] == 1 || grid[grid.length - 1][grid[0].length - 1] == 1) {
        return -1;
    }
    
    int n = grid.length;
    int[][] directions = {{-1, -1}, {-1, 0}, {-1, 1}, {0, -1}, 
                          {0, 1}, {1, -1}, {1, 0}, {1, 1}};
    
    Queue<int[]> queue = new LinkedList<>();
    queue.offer(new int[]{0, 0});
    grid[0][0] = 1;  // Mark as visited
    
    int pathLength = 1;
    
    while (!queue.isEmpty()) {
        int size = queue.size();
        
        for (int i = 0; i < size; i++) {
            int[] curr = queue.poll();
            
            if (curr[0] == n - 1 && curr[1] == n - 1) {
                return pathLength;
            }
            
            for (int[] dir : directions) {
                int newRow = curr[0] + dir[0];
                int newCol = curr[1] + dir[1];
                
                if (newRow >= 0 && newRow < n && newCol >= 0 && newCol < n 
                    && grid[newRow][newCol] == 0) {
                    grid[newRow][newCol] = 1;  // Mark as visited
                    queue.offer(new int[]{newRow, newCol});
                }
            }
        }
        pathLength++;
    }
    
    return -1;
}
```

**Time**: O(n²) | **Space**: O(n²)

---

## 7. Word Ladder (BFS)

```java
public int ladderLength(String beginWord, String endWord, List<String> wordList) {
    Set<String> wordSet = new HashSet<>(wordList);
    if (!wordSet.contains(endWord)) return 0;
    
    Queue<String> queue = new LinkedList<>();
    Set<String> visited = new HashSet<>();
    
    queue.offer(beginWord);
    visited.add(beginWord);
    
    int level = 1;
    
    while (!queue.isEmpty()) {
        int size = queue.size();
        
        for (int i = 0; i < size; i++) {
            String word = queue.poll();
            
            if (word.equals(endWord)) return level;
            
            for (String neighbor : getNeighbors(word, wordSet)) {
                if (!visited.contains(neighbor)) {
                    visited.add(neighbor);
                    queue.offer(neighbor);
                }
            }
        }
        level++;
    }
    
    return 0;
}

private List<String> getNeighbors(String word, Set<String> wordSet) {
    List<String> neighbors = new ArrayList<>();
    char[] chars = word.toCharArray();
    
    for (int i = 0; i < chars.length; i++) {
        char original = chars[i];
        for (char c = 'a'; c <= 'z'; c++) {
            if (c != original) {
                chars[i] = c;
                String newWord = new String(chars);
                if (wordSet.contains(newWord)) {
                    neighbors.add(newWord);
                }
            }
        }
        chars[i] = original;
    }
    return neighbors;
}
```

**Time**: O(M² × N) where M is word length, N is number of words

---

## Key Patterns

| Pattern | Use Case |
|---------|----------|
| BFS | Shortest path, level-order traversal |
| DFS | Connected components, cycle detection |
| Topological sort | Dependency resolution, scheduling |
| Union-Find | Connected components, MST |
| Dijkstra | Shortest path with weights |
| Bellman-Ford | Shortest path with negative weights |
| Floyd-Warshall | All-pairs shortest path |
