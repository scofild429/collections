---
title: "Python"
org_id: "705DF36B-D6E2-4662-8EA2-F0AAE6ABB5BC"
---

# Python

## Iterators

### Two-argument `iter(callable, sentinel)`

`iter(iterable)` obtains an iterator from an iterable. The two-argument form instead repeatedly calls a callable **with no arguments** until its result compares equal to a stop value, the **sentinel**.

Each request for the next item makes one call. A result equal to the sentinel ends iteration and is **not yielded**. The comparison uses equality (`==`), not identity (`is`); for example, `0.0` compares equal to `0`.

### Example: read fixed-size blocks

```python
from io import BytesIO

stream = BytesIO(b"abcdefg")
blocks = list(iter(lambda: stream.read(3), b""))
assert blocks == [b"abc", b"def", b"g"]
```

The final `read(3)` returns `b""`, ending iteration. An equivalent loop is:

```python
from io import BytesIO

stream = BytesIO(b"abcdefg")
blocks = []
while True:
    block = stream.read(3)
    if block == b"":
        break
    blocks.append(block)
assert blocks == [b"abc", b"def", b"g"]
```

### Details to remember

- Iteration is lazy: creating the iterator does not call the callable.
- Choose a sentinel that cannot be a valid item you want to retain.
- Iterator exhaustion raises `StopIteration`; a `for` loop handles it automatically.
- `StopIteration` from the callable also ends iteration. Other exceptions propagate.

Reference: [Python built-in `iter`](https://docs.python.org/3/library/functions.html#iter).
