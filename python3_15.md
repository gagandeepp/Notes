**<u>PEP 814</u>** : Add frozendict built-in type 

A new immutable type, <u>frozendict, is added to the builtins module. It does not allow</u> modification after creation. A frozendict is not a subclass of dict; it inherits directly from object. A frozendict is hashable as long as all of its keys and values are hashable. A frozendict preserves insertion order, but comparison does not take order into account. 

For example: 

Copy 

a = frozendict(x=1, y=2) 

a 

frozendict({'x': 1, 'y': 2}) a['z'] = 3 

Traceback (most recent call last): 

File "<python-input-2>", line 1, in <module> 

a['z'] = 3 ~^^^^^ 

TypeError: 'frozendict' object does not support item assignment b = frozendict(y=2, x=1) 

hash(a) == hash(b) 

# True 

a == b 

True 

**<u>PEP 810</u>** : Explicit lazy imports for faster startup times 

The lazy keyword works with both import and from ... import statements. When you write lazy import heavy_module, Python does not immediately load the module. Instead, it creates a lightweight proxy object. The actual module loading happens transparently when you first access the name: 

# Copy 

# **lazy import json** 

**lazy from pathlib import** Path 

print("Starting up...") _# json and pathlib are not fully imported yet_ 

data = json.loads('{"key": "value"}') _# json gets fully imported here_ p = Path(".") _# pathlib gets fully imported here_ 

**<u>PEP 686</u>** : Python now uses UTF-8 as the default encoding 

Python now uses UTF-8 as the default encoding, independent of the system’s environment. This means that I/O operations without an explicit encoding, for example, open('flyingcircus.txt'), will use UTF-8. UTF-8 is a widely-supported Unicode character encoding that has become a _de facto_ standard for representing text, including nearly every webpage on the internet, many common file formats, programming languages, and more. 

**<u>PEP 798</u>** : Unpacking in comprehensions 

List, set, and dictionary comprehensions, as well as generator expressions, now support unpacking with * and **. This extends the unpacking syntax from **<u>PEP 448</u>** to comprehensions, providing a new syntax for combining an arbitrary number of iterables or dictionaries into a single flat structure. This new syntax is a direct alternative to nested comprehensions, itertools.chain(), and itertools.chain.from_iterable(). For example: 

Copy 

lists = [[1, 2], [3, 4], [5]] 

[*L **for** L **in** lists] _# equivalent to [x for L in lists for x in L]_ 

[1, 2, 3, 4, 5] 

sets = [{1, 2}, {2, 3}, {3, 4}] 

- {*s **for** s **in** sets} _# equivalent to {x for s in sets for x in s}_ 

{1, 2, 3, 4} 

dicts = [{'a': 1}, {'b': 2}, {'a': 3}] 

{**d **for** d **in** dicts} _# equivalent to {k: v for d in dicts for k,v in d.items()}_ 

{'a': 3, 'b': 2} 

**<u>PEP 829</u>** : Package startup configuration flesi 

**<u>PEP 661</u>** : Add sentinel built-in type 

A new sentinel type is added to the builtins module for creating unique sentinel values with a concise representation. Sentinel objects preserve identity when copied, support use in type expressions with the | operator, and can be pickled when they are importable by module and name. 

MISSING = sentinel("MISSING") 

MISSING 

MISSING 

**<u>PEP 799</u>** : A dedicated profiling package 

A new proflingi module has been added to organize Python’s built-in profiling tools under a single, coherent namespace. This module contains: 

- <u>profling.tracingi</u> : deterministic function-call tracing (relocated from cProfile). 

- <u>profling.samplingi</u> : a new statistical sampling profiler (named Tachyon). 

# **A. Reference Counting (The Primary Mechanism)** 

Every object in Python has a hidden counter (ob_refcnt) inside its C structure. This tracks how many variables are currently pointing to that object. [1, <u>2]</u> 

- When you create a variable or add it to a list, the count goes **up** . 

- When a variable falls out of scope or is reassigned, the count goes **down** . 

- **The moment an object's reference count hits zero, Python instantly destroys it** and frees up that memory block. [1, <u>2]</u> 

# **B. Generational Garbage Collection (The Backup System)** 

Reference counting alone has a major flaw: **Cyclic References** . If Object A points to Object B, and Object B points back to Object A, their reference counts will never hit zero—even if the rest of your program completely forgets about them. [1] 

To fix this, Python runs an automated **Generational Garbage Collector (GC)** in the background: [1, <u>2]</u> 

1. Objects are classified into three generations (0, 1, and 2) based on how long they have survived. [1] 

2. New objects start in **Generation 0** . This generation is scanned frequently because new objects tend to die quickly. [1] 

3. If an object survives a GC scan, it gets promoted to **Generation 1** , and eventually to **Generation 2** . [1] 

4. The GC specifically searches for isolated "islands" of objects that reference each other but are otherwise unreachable by your code, safely purging them from the heap. [1] 

A **memory leak in Python** occurs when a program allocates memory for objects but fails to release that memory back to the system after those objects are no longer needed. Over time, these unused objects pile up in your computer's RAM, causing the application to gradually consume more resources until it slows down, degrades performance, or completely crashes with an "Out of Memory" (OOM) error. [1, <u>2, 3]</u> 

- Are you facing a **memory leak** or running out of RAM? 

- Are you dealing with **large data structures** (like massive pandas DataFrames or numpy arrays)? 

 Do you want to know how to **manually inspect or profile** memory usage in Python? 

