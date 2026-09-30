---
title: "Python"
org_id: "705DF36B-D6E2-4662-8EA2-F0AAE6ABB5BC"
---

# Python

## Collections

### The Two-Argument iter(callable, sentinel) in Python

The built-in iter() function, typically used to convert iterables (like lists) into iterators, features a powerful secondary mode when provided with exactly two arguments. It transforms a repetitive while loop into an elegant, iterable for loop.

``` python
iterator = iter(callable, sentinel) 
```

⚙️ How it Works

1.  The callable : A function (often a lambda ) that takes no arguments. It is executed repeatedly each time the iterator

advances.

1.  The sentinel : A designated "stop value."
2.  Execution Flow: When iterated over, Python continuously executes the callable . It yields the return value to the loop until

the function returns a value that exactly matches the sentinel . Once the sentinel is returned, a StopIteration exception is internally raised, and the loop terminates cleanly.
