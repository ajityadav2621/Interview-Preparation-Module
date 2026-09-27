# Python Concepts — Basic → Intermediate → Advanced

A dedicated, structured pass through Python itself (separate from the coding *problems* in `05_coding_questions.md`). Each concept includes what it is, how it works internally, and a short code example. This is the file to use if an interviewer says "let's talk about Python" rather than "solve this problem."

---

## BASIC

### 1. Data Types & Mutability
- Immutable: `int`, `float`, `str`, `tuple`, `frozenset`, `bool`.
- Mutable: `list`, `dict`, `set`.
- **Internally**: every value is an object; a variable is a name bound to an object, not a container holding it. `a = b` binds `a` to the *same object* `b` points to.
```python
a = [1, 2, 3]
b = a
b.append(4)
print(a)  # [1, 2, 3, 4]  -- same underlying object
```

### 2. Control Flow
`if/elif/else`, `for`, `while`, `break`/`continue`, and the less-common but interview-relevant `for...else` (the `else` runs only if the loop completed without `break` — useful for "search and flag if not found" patterns).
```python
for item in items:
    if item == target:
        break
else:
    print("not found")
```

### 3. Functions
- Default arguments, `*args` (extra positional args as a tuple), `**kwargs` (extra keyword args as a dict).
- **Common trap**: mutable default arguments.
```python
def add_item(item, bucket=[]):   # BUG: default list is created ONCE at function definition
    bucket.append(item)
    return bucket
```
Because default argument objects are created once when the function is defined (not on every call), repeated calls share and mutate the *same* list. Fix: use `None` as the default and create the list inside the function.

### 4. Strings
- Immutable sequences; slicing (`s[1:4]`), f-strings for formatting, `.split()`/`.join()`/`.strip()`.
- **Internally**: string concatenation in a loop (`s += x`) is O(n²) overall because each `+=` creates a new string object (strings are immutable); use `"".join(list_of_pieces)` instead, which builds the result once.

### 5. Collections: list, tuple, dict, set
- `list`: dynamic array, ordered, mutable, O(1) amortized append, O(n) insert/delete at arbitrary position.
- `tuple`: fixed, immutable, hashable if contents are hashable.
- `dict`: hash table, O(1) average lookup/insert; insertion order preserved since Python 3.7.
- `set`: hash table storing only keys — O(1) average membership check, useful for de-duplication and fast "in" checks.

### 6. Exceptions
```python
try:
    risky()
except ValueError as e:
    handle(e)
except (TypeError, KeyError):
    handle_multiple()
else:
    print("ran only if no exception")
finally:
    print("always runs, even on return/exception")
```
Catch the most specific exception you can — a bare `except:` hides bugs (including things like `KeyboardInterrupt`).

---

## INTERMEDIATE

### 7. List/Dict/Set Comprehensions & Generator Expressions
```python
squares = [x**2 for x in range(10) if x % 2 == 0]     # list, built eagerly
gen = (x**2 for x in range(10) if x % 2 == 0)          # generator, lazy
```
**Internally**: a list comprehension builds the entire list in memory immediately; a generator expression (parentheses instead of brackets) produces a lazy iterator that computes one value at a time on demand — use it when you don't need the whole sequence in memory at once, or might stop early.

### 8. Generators & `yield`
```python
def read_large_file_in_chunks(path, chunk_size=1024):
    with open(path) as f:
        while chunk := f.read(chunk_size):
            yield chunk
```
**Internally**: calling a generator function doesn't run the body — it returns a generator object. Each call to `next()` resumes execution right after the last `yield`, running until the next `yield` or the function ends (raising `StopIteration`). This is how you process files/streams larger than memory, one piece at a time.

### 9. Decorators
```python
import functools, time

def timed(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        result = func(*args, **kwargs)
        print(f"{func.__name__} took {time.perf_counter() - start:.4f}s")
        return result
    return wrapper

@timed
def slow_task():
    ...
```
**Internally**: `@timed` above `slow_task` is exactly `slow_task = timed(slow_task)` — a decorator is a function that takes a function and returns a (usually wrapped) function. `functools.wraps` copies over `__name__`/`__doc__` from the original so introspection/debugging tools still show the real function's identity instead of `wrapper`.

### 10. Context Managers (`with`)
```python
class DatabaseConnection:
    def __enter__(self):
        self.conn = connect()
        return self.conn
    def __exit__(self, exc_type, exc_val, exc_tb):
        self.conn.close()
        return False  # don't suppress exceptions

with DatabaseConnection() as conn:
    conn.execute(...)
# conn.close() is guaranteed to run even if an exception was raised inside the block
```
**Internally**: `with` calls `__enter__` before the block and `__exit__` after — guaranteed even on an exception — which is why context managers are the standard way to manage resources (files, DB connections, locks) that must always be released.

### 11. OOP: Classes, Inheritance, Composition
```python
class Animal:
    def speak(self):
        raise NotImplementedError

class Dog(Animal):
    def speak(self):
        return "Woof"
```
- **Inheritance** ("is-a"): `Dog` is an `Animal`. Overused inheritance leads to fragile hierarchies.
- **Composition** ("has-a"): prefer holding an instance of another class as an attribute when you just need to reuse behavior, not model a true is-a relationship — generally more flexible and easier to test.
- `super()` calls the parent class's method — useful in `__init__` to extend rather than fully override parent setup.

### 12. Modules & Packages
- A module is a `.py` file; a package is a directory with `__init__.py` (or, since Python 3.3+, even without one — a "namespace package").
- `if __name__ == "__main__":` guards code that should run only when the file is executed directly, not when it's imported — this is why a script can double as both a runnable tool and an importable library.

### 13. Type Hints
```python
def get_user(user_id: int) -> dict[str, str] | None:
    ...
```
Type hints don't enforce anything at runtime by default — they're for readability, IDE support, and static checkers like `mypy`. Worth mentioning if asked "does Python have static typing" — it's optional and gradual, not enforced.

---

## ADVANCED

### 14. The GIL (Global Interpreter Lock) and Concurrency Models
- CPython's GIL means only one thread executes Python bytecode at a time, even on a multi-core machine.
- **Threading** still helps for I/O-bound work (network calls, file I/O, DB queries) because the GIL is released while waiting on I/O.
- **Multiprocessing** is the way to get true parallelism for CPU-bound work — each process has its own Python interpreter and memory space (and thus its own GIL), at the cost of higher memory use and the need to serialize data between processes (`pickle`).
- **`asyncio`**: a single-threaded event loop that runs many coroutines concurrently by cooperatively switching between them at `await` points — great for high-throughput I/O-bound workloads (e.g., many concurrent API calls) without the overhead of threads/processes, but a single CPU-bound `await`-free chunk of code still blocks the whole event loop.

```python
import asyncio

async def fetch(url):
    async with session.get(url) as resp:
        return await resp.text()

async def main():
    results = await asyncio.gather(*(fetch(u) for u in urls))  # concurrent, not parallel
```

### 15. Memory Management: Reference Counting & Garbage Collection
- Every object has a reference count; when it hits zero, CPython frees it immediately.
- Reference counting alone can't collect **reference cycles** (e.g., two objects referencing each other) — a separate generational garbage collector periodically scans for and collects these.
- `__del__` (a destructor) is called when an object is garbage collected — but its timing isn't guaranteed for cyclic garbage, which is why relying on it for critical cleanup (like closing a file) is discouraged; use a context manager instead.

### 16. Descriptors & Properties
```python
class Celsius:
    def __set_name__(self, owner, name):
        self.name = "_" + name
    def __get__(self, obj, objtype=None):
        return getattr(obj, self.name, 0)
    def __set__(self, obj, value):
        if value < -273.15:
            raise ValueError("Below absolute zero")
        setattr(obj, self.name, value)

class Weather:
    temperature = Celsius()
```
**Internally**: a descriptor is any object implementing `__get__`/`__set__`; Python's attribute lookup (`obj.attr`) checks the class for a descriptor before falling back to the instance's `__dict__`. `@property` is just a built-in, simpler way to define a descriptor for a single attribute without writing a full descriptor class.

### 17. Metaclasses
- A metaclass is "the class of a class" — `type` is the default metaclass for all classes. Defining `class Meta(type): ...` and using `class Foo(metaclass=Meta):` lets you customize *how classes themselves are created* (e.g., auto-registering every subclass, validating that required methods are implemented).
- **When to actually use this**: rarely, in application code — mention you know it exists and what it's for (frameworks like Django's ORM use metaclasses to turn model class definitions into database-backed classes), but for most day-to-day work, simpler tools (decorators, `__init_subclass__`) solve the same problems with less complexity.

### 18. `__init_subclass__` (a lighter alternative to metaclasses)
```python
class Plugin:
    registry = []
    def __init_subclass__(cls, **kwargs):
        super().__init_subclass__(**kwargs)
        Plugin.registry.append(cls)

class MyPlugin(Plugin):
    pass
# MyPlugin is automatically registered — no metaclass needed
```

### 19. Iterators & the Iterator Protocol
- An **iterable** implements `__iter__` (returns an iterator). An **iterator** implements both `__iter__` (returns itself) and `__next__` (returns the next value or raises `StopIteration`).
- This is the actual mechanism behind `for` loops — `for x in thing:` is sugar for calling `iter(thing)` once, then `next()` repeatedly until `StopIteration`. Generators are the easiest way to get a correct iterator implementation without writing the protocol by hand.

### 20. `*args`/`**kwargs` Unpacking, Closures, and `functools.partial`
```python
from functools import partial

def power(base, exponent):
    return base ** exponent

square = partial(power, exponent=2)
square(5)  # 25
```
**Closures**: a nested function that captures variables from its enclosing scope — this is the mechanism decorators rely on (`wrapper` closing over `func`).

### 21. Performance Considerations Worth Mentioning
- Prefer built-in functions/comprehensions over manual Python loops where possible — they're implemented in C and avoid per-iteration bytecode dispatch overhead.
- `__slots__` on a class avoids creating a per-instance `__dict__`, reducing memory for classes with many small instances (at the cost of losing dynamic attribute assignment).
- Profile before optimizing (`cProfile`, `timeit`) — guessing at bottlenecks is a common mistake.

---

## Quick Self-Test
- Explain why mutable default arguments are a bug waiting to happen.
- Explain the difference between a generator and a list comprehension, and when each is appropriate.
- Explain why threading helps I/O-bound Python code but not CPU-bound code, and what does help CPU-bound code.
- Explain what a context manager guarantees that a plain try/finally doesn't add — and why they're related (a context manager is often *implemented* using try/finally internally).
- Explain what a decorator actually does to the function it wraps, at the assignment level (`func = decorator(func)`).
