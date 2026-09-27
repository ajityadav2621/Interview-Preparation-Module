# Coding Questions — Python (Fundamentals → Advanced)

Each question has a full solution, an explanation of *why* it works, and time/space complexity. Type these out yourself rather than reading passively — muscle memory matters more than recognition in a live interview.

---

## Fundamentals

### 1. FizzBuzz
**Question**: Print numbers 1 to n; multiples of 3 print "Fizz", multiples of 5 print "Buzz", multiples of both print "FizzBuzz".

```python
def fizzbuzz(n: int) -> list[str]:
    result = []
    for i in range(1, n + 1):
        if i % 15 == 0:
            result.append("FizzBuzz")
        elif i % 3 == 0:
            result.append("Fizz")
        elif i % 5 == 0:
            result.append("Buzz")
        else:
            result.append(str(i))
    return result
```
**Why**: check the combined case (`% 15`) first, otherwise "Fizz" or "Buzz" would fire before you get a chance to catch "FizzBuzz". **Complexity**: O(n) time, O(n) space for the result list.

---

### 2. Check if a string is a palindrome
```python
def is_palindrome(s: str) -> bool:
    cleaned = [c.lower() for c in s if c.isalnum()]
    return cleaned == cleaned[::-1]
```
**Why**: strip non-alphanumeric characters and normalize case first (so "A man, a plan, a canal: Panama" works), then compare the sequence to its reverse. **Complexity**: O(n) time, O(n) space.

**Follow-up they may ask**: do it with two pointers instead of slicing, to avoid the extra reversed copy:
```python
def is_palindrome_two_pointer(s: str) -> bool:
    cleaned = [c.lower() for c in s if c.isalnum()]
    left, right = 0, len(cleaned) - 1
    while left < right:
        if cleaned[left] != cleaned[right]:
            return False
        left += 1
        right -= 1
    return True
```

---

### 3. Count word frequency in a text
```python
from collections import Counter

def word_frequency(text: str) -> dict[str, int]:
    words = text.lower().split()
    return dict(Counter(words))
```
**Why**: `Counter` is a dict subclass built exactly for counting hashable items — internally it's a hash table just like `dict`, so counting is O(1) per insert. **Complexity**: O(n) time where n is word count.

---

### 4. Check for balanced parentheses
```python
def is_balanced(s: str) -> bool:
    pairs = {')': '(', ']': '[', '}': '{'}
    stack = []
    for char in s:
        if char in '([{':
            stack.append(char)
        elif char in ')]}':
            if not stack or stack.pop() != pairs[char]:
                return False
    return not stack
```
**Why**: a stack naturally models "most recently opened bracket must close first" (LIFO). If a closing bracket doesn't match the top of the stack, or the stack is empty when we hit a closer, it's unbalanced. At the end, the stack must be empty (every opener was closed). **Complexity**: O(n) time, O(n) space worst case.

---

### 5. Find the first non-repeating character in a string
```python
from collections import Counter

def first_unique_char(s: str) -> str | None:
    counts = Counter(s)
    for char in s:
        if counts[char] == 1:
            return char
    return None
```
**Why**: one pass to count, one pass in original order to find the first with count 1 — using a hash map avoids the O(n²) of checking `s.count(char)` for every character. **Complexity**: O(n) time, O(1) space (bounded alphabet) or O(n) generally.

---

## Intermediate

### 6. Flatten a nested list of arbitrary depth
```python
def flatten(nested: list) -> list:
    result = []
    for item in nested:
        if isinstance(item, list):
            result.extend(flatten(item))
        else:
            result.append(item)
    return result

# flatten([1, [2, [3, 4], 5], 6]) -> [1, 2, 3, 4, 5, 6]
```
**Why**: recursion mirrors the recursive structure of the data itself — each nested list is solved the same way as the outer one. **Complexity**: O(n) time where n is total number of elements, O(d) extra space for recursion depth d.

---

### 7. Implement a retry decorator with exponential backoff
```python
import time
import functools

def retry_with_backoff(max_attempts=3, base_delay=1):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            attempt = 0
            while True:
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    attempt += 1
                    if attempt >= max_attempts:
                        raise
                    delay = base_delay * (2 ** (attempt - 1))
                    time.sleep(delay)
        return wrapper
    return decorator

@retry_with_backoff(max_attempts=4, base_delay=1)
def call_flaky_api():
    ...
```
**Why**: a decorator wraps a function to add behavior without changing its code — here, catching failures and retrying with a growing delay (1s, 2s, 4s...) instead of hammering a struggling API. `functools.wraps` preserves the original function's name/docstring for debugging. **Complexity**: not applicable in the usual sense — bounded by `max_attempts` calls.

---

### 8. Implement an LRU (Least Recently Used) cache
```python
from collections import OrderedDict

class LRUCache:
    def __init__(self, capacity: int):
        self.capacity = capacity
        self.cache = OrderedDict()

    def get(self, key):
        if key not in self.cache:
            return -1
        self.cache.move_to_end(key)   # mark as recently used
        return self.cache[key]

    def put(self, key, value):
        if key in self.cache:
            self.cache.move_to_end(key)
        self.cache[key] = value
        if len(self.cache) > self.capacity:
            self.cache.popitem(last=False)   # evict least recently used
```
**Why**: `OrderedDict` maintains insertion order and supports O(1) `move_to_end`, so the "least recently used" item is always the one at the front — evicting it on overflow is O(1). This is a very common interview question because it tests hash map + ordering reasoning together. **Complexity**: O(1) for both `get` and `put`.

---

### 9. Paginate a list
```python
def paginate(items: list, page: int, page_size: int) -> list:
    start = (page - 1) * page_size
    end = start + page_size
    return items[start:end]
```
**Why**: this is the core logic behind API pagination (`?page=2&page_size=20`) — computing an offset from page number, then slicing. **Complexity**: O(page_size) since slicing copies that many elements.

---

### 10. Merge two sorted lists
```python
def merge_sorted(a: list[int], b: list[int]) -> list[int]:
    result = []
    i = j = 0
    while i < len(a) and j < len(b):
        if a[i] <= b[j]:
            result.append(a[i]); i += 1
        else:
            result.append(b[j]); j += 1
    result.extend(a[i:])
    result.extend(b[j:])
    return result
```
**Why**: classic two-pointer merge — the same core step used inside merge sort. **Complexity**: O(n + m) time, O(n + m) space.

---

## Advanced (AI / API-flavored)

### 11. Chunk text into overlapping chunks (for RAG embedding)
```python
def chunk_text(text: str, chunk_size: int = 500, overlap: int = 50) -> list[str]:
    words = text.split()
    chunks = []
    start = 0
    while start < len(words):
        end = start + chunk_size
        chunk = " ".join(words[start:end])
        chunks.append(chunk)
        start += chunk_size - overlap   # step forward, but re-include the overlap
    return chunks
```
**Why**: overlap ensures a sentence/idea that falls right on a chunk boundary isn't split with no shared context between the two chunks — this materially affects retrieval quality. **Complexity**: O(n) time/space where n is word count. (See `07_rag_pdf_chatbot_design.md` for how this fits the full pipeline.)

---

### 12. Cosine similarity search over a small set of embeddings
```python
import math

def cosine_similarity(a: list[float], b: list[float]) -> float:
    dot = sum(x * y for x, y in zip(a, b))
    norm_a = math.sqrt(sum(x * x for x in a))
    norm_b = math.sqrt(sum(y * y for y in b))
    return dot / (norm_a * norm_b) if norm_a and norm_b else 0.0

def top_k_similar(query_vec: list[float], candidates: list[tuple[str, list[float]]], k: int = 3):
    scored = [(text, cosine_similarity(query_vec, vec)) for text, vec in candidates]
    scored.sort(key=lambda pair: pair[1], reverse=True)
    return scored[:k]
```
**Why**: this is what a vector database does at scale (with an approximate-nearest-neighbor index instead of brute force). Cosine similarity measures the angle between two vectors, not their magnitude — appropriate for embeddings, where direction encodes meaning. **Complexity**: brute-force top-k is O(n · d) for n candidates of dimension d, then O(n log n) to sort — a real vector DB avoids the full scan with an index (e.g., HNSW).

---

### 13. Simple token-bucket rate limiter
```python
import time

class TokenBucket:
    def __init__(self, capacity: int, refill_rate: float):
        self.capacity = capacity
        self.tokens = capacity
        self.refill_rate = refill_rate  # tokens per second
        self.last_refill = time.monotonic()

    def allow_request(self) -> bool:
        now = time.monotonic()
        elapsed = now - self.last_refill
        self.tokens = min(self.capacity, self.tokens + elapsed * self.refill_rate)
        self.last_refill = now
        if self.tokens >= 1:
            self.tokens -= 1
            return True
        return False
```
**Why**: tokens refill continuously based on elapsed time (not in discrete ticks), which smooths out bursts while still allowing short spikes up to `capacity` — this is how most real API rate limiters work internally. **Complexity**: O(1) per check.

---

### 14. Idempotency-key decorator (avoid duplicate side effects on retry)
```python
def idempotent(store: dict):
    def decorator(func):
        def wrapper(idempotency_key, *args, **kwargs):
            if idempotency_key in store:
                return store[idempotency_key]
            result = func(*args, **kwargs)
            store[idempotency_key] = result
            return result
        return wrapper
    return decorator

seen_requests = {}

@idempotent(seen_requests)
def create_ticket(payload):
    ...  # actually create the ticket
```
**Why**: if a client retries a request after a timeout (not knowing if it succeeded), the server can look up the same idempotency key and return the cached result instead of creating a duplicate — directly relevant to integrating with ITSM/CMDB systems as named in the JD. **Complexity**: O(1) per call given a dict-backed store.

---

### 15. Simple LLM-response JSON validator with one retry
```python
import json

def get_structured_output(llm_call, prompt: str, max_retries: int = 1):
    for attempt in range(max_retries + 1):
        raw = llm_call(prompt)
        try:
            return json.loads(raw)
        except json.JSONDecodeError:
            if attempt == max_retries:
                raise ValueError("Model did not return valid JSON after retries")
            prompt = prompt + "\n\nReturn ONLY valid JSON, no other text."
```
**Why**: LLMs predicting "JSON-shaped" text isn't a hard guarantee (see `02_intermediate.md` internals on prompting) — production code has to validate and retry with a stricter instruction rather than trusting the output blindly.

---

### How to use this section in an interview
- If asked to solve live, **talk while you code**: state the approach before typing, mention the complexity unprompted, and name the edge cases you're handling (empty input, single element, duplicates).
- If you finish early, proactively mention the follow-up/optimized version (like the two-pointer palindrome check) — it signals depth without being asked.
