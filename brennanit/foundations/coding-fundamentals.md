# 6. Coding Fundamentals

## 6.1 The Right Mindset for Coding Discussions

Coding interviews for this role test problem-solving and clear communication, not language trivia. The goal is to show how you think under constraints.

Core habits:

- Always clarify the problem before writing anything.
- Think out loud so the interviewer can follow your reasoning.
- Plan an approach before reaching for the keyboard.
- State constraints, complexity, and edge cases explicitly.
- Write clean, readable code with meaningful names.
- Test your solution against examples and edge cases by hand.
- Discuss trade-offs rather than presenting one fixed answer.

## 6.2 A Repeatable Problem-Solving Framework

Use this sequence for every coding problem:

1. **Clarify** — inputs, outputs, duplicates, ordering, size limits, special characters.
2. **Example** — confirm understanding with a concrete case.
3. **Brute force** — describe the simple correct solution, even if slow.
4. **Optimize** — identify the bottleneck and apply a pattern or data structure.
5. **Design** — state the algorithm, complexity, and memory trade-off.
6. **Implement** — write clean code, explaining decisions as you go.
7. **Verify** — trace through examples and edge cases.
8. **Reflect** — discuss alternatives and when they would be better.

## 6.3 Choosing the Right Language and Environment

Pick a language you know well enough to write without looking up syntax. The problem-solving logic matters more than the language choice.

- Python is acceptable for its readability and standard-library richness.
- Use a plain editor-style approach unless told otherwise.
- Avoid over-engineering or importing heavy libraries.
- If you use a helper, explain it briefly.

## 6.4 Essential Data Structures

Know the behavior, complexity, and typical use of each:

| Structure | Strength | Common operations |
|---|---|---|
| Array / List | Indexed access, cache friendly | Read by index, append, slice |
| Hash map / Set | O(1) lookup on average | Insert, lookup, delete |
| String | Text processing | Compare, split, search |
| Stack | Last-in, first-out | Push, pop |
| Queue / Deque | First-in, first-out | Enqueue, dequeue |
| Linked list | Efficient inserts/removal mid-list | Traverse, insert, delete |
| Tree / BST | Hierarchical, ordered | Traverse, search, insert |
| Heap | Fast min or max | Push, pop, peek |
| Graph | Relationships and paths | Traverse, shortest path |

## 6.5 Essential Algorithms and Patterns

### Sorting and searching
- Sorting establishes order in O(n log n).
- Binary search finds an element in a sorted structure in O(log n).

### Two pointers
Use when a problem involves pairs, palindromes, or partitions in a sorted or sequential structure. One pointer advances the other.

### Sliding window
Use for subarrays or substrings under constraints. Expand the right boundary and shrink the left when the window becomes invalid.

### Depth-first and breadth-first search
- DFS explores as deep as possible, useful for paths, cycles, and recursion.
- BFS explores level by level, useful for shortest paths in unweighted graphs.

### Recursion and backtracking
Break a problem into smaller identical subproblems. Useful for trees, permutations, and combinations.

### Dynamic programming basics
Cache subproblem results to avoid recomputation. Key steps: define the state, find the recurrence, choose between memoization and tabulation, and consider space optimization.

### Greedy and union-find
Greedy makes the best local choice and is easy to justify incorrectly; verify correctness carefully. Union-find efficiently answers connectivity questions and supports Kruskal-style algorithms.

## 6.6 Complexity Analysis

### Time and space
- Express complexity in terms of input size n.
- State both time and space, and explain the trade-off.
- Note the difference between worst case and average case where relevant.

### Amortized analysis
Some operations are occasionally expensive but cheap on average, such as dynamic array resizing.

### Trade-offs to discuss
- Precompute for faster queries at higher memory cost.
- Sort once to enable fast lookups later.
- Use a hash map for speed, or a balanced tree to keep order.

## 6.7 Testing, Debugging, and Edge Cases

Good engineers do not treat the first solution as final.

Edge cases to consider:

- Empty input or single element
- Duplicates or repeated patterns
- Maximum and minimum values
- Null or invalid references
- Already sorted or reverse-sorted input

Debugging advice:

- Trace through examples by hand, aloud.
- Use clear variable names so logic reads left to right.
- Isolate the failing case and re-derive expected output.
- Add invariants in comments only when helpful to reasoning.

## 6.8 Writing Clean, Maintainable Code

Readability matters as much as correctness.

- Name variables to express intent, not just type.
- Keep functions short and single-purpose.
- Avoid clever tricks that obscure intent.
- Prefer explicit handling of edge cases over silent defaults.
- When you choose a trade-off, state it.

## 6.9 Common Pitfalls in Coding Interviews

- Rushing to code without clarifying requirements.
- Not considering edge cases and complexity.
- Freezing on a hard problem instead of narrating a working brute force.
- Over-explaining trivial steps or under-explaining key decisions.
- Ignoring the interviewer's hints.
- Leaving a partial solution when a complete simpler one is better.

## 6.10 Interview Answer: A Complete Walkthrough

> I start by confirming inputs, outputs, and constraints, then I agree on an example. I describe a brute-force solution and its complexity before optimizing. As I refine the approach, I state the pattern I am using, the data structure choice, and the resulting time and space complexity. I implement deliberately, narrating decisions, then trace the example and edge cases by hand. Finally, I discuss alternatives and when each would be preferable.

### Sample problem walk-through (conceptual)

Problem: find whether an array contains two distinct indices whose values sum to a target.

- Clarify: integers, duplicates allowed, one or many solutions, return indices.
- Brute force: nested loop, O(n^2), O(1) space.
- Optimize: use a hash map to store seen values; for each value, check whether the complement exists. O(n) time, O(n) space.
- Complexity trade-off: hash map for speed versus sorted two pointers for O(1) extra space at the cost of O(n log n) sort.
- Edge cases: duplicates, negative numbers, no valid pair, target reached by one value needing itself.

This pattern generalizes to many "pair" and "subarray" problems.

## 6.11 Python Web Development and Relational Databases

This role builds web applications using a framework such as Django or Flask, backed by a relational database. Focus on concepts and flows, not framework trivia.

### Choosing a framework
- Flask is lightweight and explicit, good for small services and learning core concepts.
- Django is batteries-included with an ORM, admin, and authentication, good for larger applications.
- Both support REST APIs, routing, request validation, and middleware for cross-cutting concerns such as logging and authentication.

### REST API design
- Define resources and map HTTP methods to actions, preferring idempotent methods.
- Return consistent status codes (200, 201, 400, 401, 403, 404, 409, 429, 5xx) and a standard error format.
- Validate input at the edge and validate output before responding.
- Document the API with OpenAPI so clients and reviewers share one source of truth.

### Relational databases
- Model data with tables, primary keys, foreign keys, and relationships.
- Use an ORM for most access, but understand the SQL it generates.
- Apply normalization to reduce duplication, and add indexes for access patterns.
- Rely on transactions (ACID) to keep related writes consistent, and choose isolation to balance correctness and concurrency.

### Key trade-offs
- ORM convenience versus query control; prefer the ORM for maintainability and raw SQL only for verified hot paths.
- Normalization versus read performance; add materialized views or caches only when a clear access pattern justifies it.
- Strong consistency versus availability; use synchronous commits for correctness, async replication for scale.

## 6.12 Testing and Debugging in Practice

Reliable delivery depends on layered testing and methodical debugging.

### Test types
- Unit tests cover individual functions and edge cases.
- Integration tests verify interactions with databases and external APIs.
- Contract tests verify API behavior against the OpenAPI spec.
- Evaluation tests measure AI output quality and catch regressions.
- End-to-end or user acceptance tests confirm the feature meets the specification.

### Debugging production issues
- Start from telemetry: correlation IDs, logs, metrics, and traces for one slow or failing request.
- Reproduce locally when possible using the same inputs and version.
- Narrow the cause by checking each stage of the flow: validation, retrieval, model call, output validation, action.
- Verify fixes against the failing case and confirm no regression in the broader test set.

### Linux and Bash scripting
- Navigate the filesystem, inspect logs, and chain commands with pipes and conditionals.
- Use variables, loops, and simple scripts to automate repetitive operational tasks.
- Understand exit codes and how to fail fast when a step errors.

## 6.13 Interview Questions and Answers

### Q: When would you choose Django over Flask, or vice versa?

> I choose Flask for small services or when I want explicit control over every piece. I choose Django for larger applications where the built-in ORM, admin, authentication, and conventions reduce duplicated effort. The decision rests on team familiarity and the size of the common scaffolding needed.

### Q: What is the difference between an ORM and raw SQL, and when would you bypass the ORM?

> An ORM maps database rows to objects and keeps code readable and database-agnostic for most queries. I bypass it for verified hot paths, complex aggregations, or operations the ORM generates poorly. I always check the generated SQL and add an integration test so the bypass stays correct.

### Q: What is a database transaction, and why does it matter?

> A transaction groups operations so they succeed or fail together, preserving ACID properties. It matters because an AI workflow may read, retrieve context, call a model, and then update several records; without a transaction, a partial failure leaves data inconsistent.

### Q: How do you test an AI application's quality?

> Beyond unit and integration tests, I maintain an evaluation dataset of representative and edge cases with golden answers. I run it against the candidate and baseline versions, comparing metrics like faithfulness, correctness, and retrieval quality. A release is blocked if quality regresses or safety failures rise.

### Q: How do you debug a production issue quickly?

> I start at the telemetry for the specific request using a correlation ID, follow the trace to the slow or failing service, reproduce locally with the same inputs, narrow each stage of the flow, fix, and verify against the failing case. I also check recent releases and config changes as likely causes.

### Q: How do you approach an unfamiliar coding problem?

> I first restate the problem in my own words and confirm the inputs, outputs, and constraints. I produce a simple, correct brute-force solution and then look for the repeated structure or bottleneck. I pick a data structure that improves on the brute force, state the complexity, and implement it cleanly while narrating my reasoning.

### Q: When would you use a hash map versus a balanced tree?

> I use a hash map when I need fast average O(1) lookup and insertion and do not need ordering. I use a balanced tree when I need sorted iteration or range queries. A tree costs O(log n) per operation but gives predictable ordering that a hash map cannot provide.

### Q: How do you decide between depth-first and breadth-first search?

> I use DFS when I need to explore a path to completion, detect cycles, or traverse a tree recursively. I use BFS when I need the shortest path in an unweighted graph or a level-by-level traversal. DFS uses less memory in the worst case, while BFS can require more memory for the frontier.

### Q: What is the difference between a list and a deque?

> A list allows efficient operations at one end but costly removal from the other. A deque supports efficient addition and removal at both ends. For problems that process elements from both sides, such as sliding windows, a deque is the more natural choice.

### Q: How do you handle timeouts in an algorithm?

> I analyze the complexity and estimate the worst case against the stated input limits. If a brute force is too slow, I look for a data structure or pattern that reduces the bottleneck, such as hashing for lookups or prefix sums for range queries. I always state the trade-off between speed and memory.

### Q: How do you debug an incorrect solution?

> I trace the algorithm against a small example by hand, including edge cases. I check each assumption, such as loop bounds and index updates. When I find the breaking case, I re-derive the expected result and fix the logic, then re-run the example to confirm.

### Q: What is dynamic programming, and when should you use it?

> Dynamic programming applies when a problem has overlapping subproblems and optimal substructure. I define a state that captures what I need to solve a subproblem, find a recurrence to build the answer, and cache results. I choose memoization for natural recursion or tabulation for iterative control.
