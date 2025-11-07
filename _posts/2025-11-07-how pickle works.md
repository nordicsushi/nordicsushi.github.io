---
title: "Understanding Pickle: What It Really Saves (and Why It Sometimes Betrays You)"
date: 2025-10-13 10:00:00 +0000
categories: [Python]
tags: [Python, Serialization]
pin: false
author: Jiaming HUANG
image:
  path: /assets/img/python-coding-mistakes.jpg
  alt: Python
---

# **Understanding Pickle: What It Really Saves (and Why It Sometimes Betrays You)**

Python’s `pickle` is everywhere: saving models, caching objects, passing data between processes. It feels magical—until a simple `pickle.load` explodes with `ModuleNotFoundError` or weird import paths.

Let’s strip away the magic and explain:

1. What `pickle.dump` and `pickle.load` actually do
2. A simple nested example showing the full round trip
3. Common pitfalls (versioning, paths, security) and how to avoid them

------

## 1. What pickle really does

At its core, `pickle` is a **Python object graph → byte stream → object graph** transformer.

When you call:

```
import pickle

pickle.dump(obj, file)
```

pickle does roughly:

1. Look at the object’s **type**:
   - Records its module path, e.g. `myapp.models.User`
2. Determine how to reconstruct it:
   - Uses `obj.__reduce__()` if defined
   - Otherwise, falls back to:
     - The object’s class
     - And its internal state (often `obj.__dict__`)
3. Encodes:
   - The constructor or reconstruction instructions
   - The state (data) of the object
   - References between objects (so shared objects / cycles are preserved)
4. Writes this as a sequence of **opcodes + data** to the file.

When you call:

```
obj = pickle.load(file)
```

pickle:

1. Reads opcodes from the byte stream.
2. For each object:
   - Imports the recorded module (`import myapp.models`)
   - Gets the class (`User`)
   - Reconstructs an instance (often *without* calling your `__init__` normally)
   - Restores its state via `__setstate__` or by assigning to `__dict__`.
3. Rebuilds references between objects (including nested structures and shared objects).

Key idea:

> Pickle doesn’t just store “data”; it stores **how to get back to the right class**, plus the **state** to stuff into it.

That’s why module paths matter so much.

------

## 2. A concrete nested example (step-by-step)

Let’s build something simple but nested so we can see how unpickling works recursively.

### Example code

**app.py**

```
import pickle

class Address:
    def __init__(self, city, country):
        self.city = city
        self.country = country

class User:
    def __init__(self, name, address, tags):
        self.name = name
        self.address = address      # nested object
        self.tags = tags            # list of strings

address = Address("Paris", "France")
user = User("Alice", address, tags=["admin", "beta_tester"])

data = {
    "owner": user,
    "members": [
        user,  # same object reference
        User("Bob", Address("Lyon", "France"), tags=["viewer"])
    ]
}

with open("data.pkl", "wb") as f:
    pickle.dump(data, f)
```

We’re pickling a structure with:

- A dict (`data`)
- Containing:
  - `"owner"` → a `User` instance
  - `"members"` → a list with:
    - The **same** `User` instance (`owner`)
    - Another `User` with its own `Address`

### What happens on `dump`?

Conceptually, pickle walks this graph:

1. Sees `data` (a `dict`)
2. Sees `"owner"` → `User("Alice", Address(...), [...])`
   - Records: class = `app.User`
   - Saves constructor/state info (e.g. via `__dict__`)
   - Sees nested `Address`:
     - Records `app.Address`, saves its state (`city`, `country`)
   - Sees `tags` list, saves its elements
3. Sees `"members"` → a `list`
   - First element is **same `User("Alice", ...)` object**:
     - pickle **does not duplicate** it; it records a reference
   - Second element: another `User` → same process again

The resulting pickle stream encodes:

- “Create a dict”
- “Key 'owner' → object #1 (User from app.User with this state…)”
- “Key 'members' → list [ref to object #1, object #2 (User), etc.]”
- “Object #2 has nested Address, etc.”

### What happens on `load`?

In another script:

```
import pickle
from app import User, Address  # must be importable from same module path

with open("data.pkl", "rb") as f:
    data = pickle.load(f)
```

Step-by-step:

1. Pickle reads: “this is a dict” → creates an empty dict.
2. For `"owner"`:
   - Reads: class `app.User`
   - Imports `app`, gets `User`
   - Constructs a `User` instance (bypassing or custom-calling `__init__` depending on reduce/state)
   - Restores its state:
     - `name = "Alice"`
     - `address` → will itself be another object to unpickle
     - `tags = ["admin", "beta_tester"]`
3. For that nested `address`:
   - Reads: class `app.Address`
   - Imports `app`, gets `Address`
   - Constructs `Address`
   - Restores `city = "Paris"`, `country = "France"`
4. For `"members"`:
   - Creates an empty list
   - First element: “reference to object #1”
     - So `members[0] is data["owner"]` is **True**
   - Second element: another `User` + nested `Address`, reconstructed the same way.

Result:

```
data["owner"] is data["members"][0]   # True
data["members"][1].address.city       # "Lyon"
```

So you don’t just get “same-looking” objects back; you get the **same structure and references** rebuilt.

That’s nested deserialization in action.

------

## 3. Common pitfalls (and how not to shoot yourself)

Now the sharp edges.

### Pitfall 1: Module path / version changes

Pickle encodes the **real module path**, not how you imported it.

If your class is defined in `app.py` as `app.User`, pickle stores:

```
module = "app"
name = "User"
```

If you later:

- Move `User` to `app/models.py`
- Rename the module
- Or upgrade a library that changes internal module paths (very common in libraries like scikit-learn)

then:

```
pickle.load(...)
# -> ModuleNotFoundError or AttributeError
```

**How to avoid / mitigate:**

- **Keep class locations stable** once you’ve pickled them.
- Avoid pickling objects of:
  - Local classes (`def f(): class C: ...`)
  - Lambdas
  - Things tied to “private” modules (`_something`) that may change.
- For libraries:
  - **Pin versions** in production.
    - Use `requirements.txt`, `poetry.lock`, Conda env, Docker, etc.
  - Store metadata with the pickle (e.g. the library versions) and **refuse to load** if mismatched.

------

### Pitfall 2: Cross-version / long-term storage

Pickle is a **Python- and implementation-detail–dependent** format. It’s not meant as a long-term, language-agnostic storage format.

You can’t rely on:

- Loading Python 3.9 pickles in some hypothetical Python 4.2 safely.
- Reading them from Java / Go / JavaScript directly.

**Mitigations:**

- For long-term / cross-system storage:
  - Store **data**, not Python objects:
    - JSON / CSV / Parquet / NumPy `.npy` / `.npz`
  - For ML models:
    - Export to standard formats like **ONNX** instead of raw pickle of Python objects.
- For Python-only but long-lived:
  - Document required Python and dependency versions.
  - Treat pickles like compiled artifacts, not universal truth.

------

### Pitfall 3: Security (this one is serious)

`pickle.load` can execute **arbitrary code** during unpickling.

If the pickle file is malicious, loading it is equivalent to:

```
eval(untrusted_input)
```

Yes, that bad.

**Rule:**

> Never `pickle.load` data from an untrusted or unauthenticated source.

**Safer options:**

- For untrusted input: use `json`, `yaml` (with safe loader), messagepack, etc.
- If you must use pickle internally:
  - Restrict who/what can write those files.
  - Use signing / checksums to ensure integrity.

------

### Pitfall 4: Big objects & performance

Pickling huge nested objects (especially with NumPy arrays) can be:

- Slow
- Memory-heavy
- Producing large files

**Mitigations:**

- Use `protocol=pickle.HIGHEST_PROTOCOL` for more efficient binaries.
- For numeric-heavy data:
  - Use `joblib` (more efficient array handling).
  - Or dedicated formats (`.npy`, `.npz`, HDF5, Parquet).

(Just remember: `joblib` still inherits the **same class path / versioning issues** as pickle.)

------

## Practical checklist

When using `pickle.dump` / `pickle.load` in real projects:

1. **Assume** pickles are **environment-specific**.
2. **Pin** Python + library versions where pickles are produced and consumed.
3. Avoid pickling:
   - Lambdas
   - Local functions/classes
   - Frequently-moving internal classes
4. Don’t use pickle as a public or untrusted input format.
5. For long-term / cross-platform needs:
   - Serialize **data and parameters**, not Python objects.
   - Or use standardized formats (ONNX, etc.) for domain-specific artifacts.

Used with clear boundaries, pickle is perfectly fine—and very convenient.
 Used as a generic, permanent, portable format… it will eventually bite.