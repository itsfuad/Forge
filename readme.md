## Forge Language - Core Design Documentation

### 1. Memory Model & Regions

**Regions** are the fundamental memory management units. They enforce ownership, mutation safety, and lifetime rules.

#### Region Types:

* **Heap (`heap`)**: Dynamic memory. Objects can be moved, mutated via pointers, and destroyed explicitly.
* **Stack (`stack`)**: Local scope. Automatically destroyed at end of scope.
* **Static (`static`)**: Global constants or long-lived objects. Immutable by default.
* **Thread-Local (`tls`)**: Lives for the thread's lifetime. Not sharable across threads unless explicitly `shared`.
* **Shared (`shared`)**: Accessible across threads. Only immutable or properly synchronized mutable access.

> **Rule:** Regions are **linear**. They cannot be copied; they can only be moved or borrowed via pointers.

#### Example:

```forge
region heap;
region stack;

let p1: Point@heap = Point{ .X=0, .Y=0 };
let p2: Point@stack = Point{ .X=1, .Y=1 };
```

---

### 2. Pointers & Ownership

* `*T@R` → mutable borrow / pointer to object in region `R`.
* `T@R` → moves ownership of object. Compiler invalidates previous aliases after move.
* Ownership moves propagate automatically, ensuring compile-time safety.

#### Borrow vs Move Example:

```forge
fn inc(p: *Point@heap) {
    p.X += 1;
}

fn consume(p: Point@heap) {
    destroy p;
}

let p1: Point@heap = Point{ .X=0, .Y=0 };
let a = &p1;
inc(a);       // safe
consume(p1);  // ownership moved, all previous pointers invalid
```

> **Rule:** Any alias to a moved value is invalidated automatically.

---

### 3. Sequential Mutable Aliases & RAII

* RAII-style **last-use tracking** allows sequential mutable aliases without explicit scoping.
* A mutable alias is valid until its **last use**. After that, the compiler invalidates it.
* Prevents use-after-free even in sequential code.

#### Example:

```forge
let a = &p1;
inc(a);    // last use of 'a'
let b = &p1;
inc(b);    // allowed
print(p1.X);  // OK, a and b lifetimes respected
```

* Passing an alias to a function that consumes or deletes the object invalidates it immediately.

---

### 4. Value-Level Exclusivity & Parallel Safety

* Mutable access is **exclusive per value**, not per region.
* Compiler enforces that only one mutable alias exists at any time.
* Sequential aliases allowed via RAII-style drop; parallel mutations allowed only for independent values.

#### Safe Parallel Example:

```forge
spawn { inc(&p1); }
spawn { inc(&p2); } // allowed
```

#### Unsafe Parallel Example:

```forge
spawn { inc(&p1); }
spawn { inc(&p1); } // ❌ compile-time error
```

* Compiler guarantees **no data races** at compile-time.

---

### 5. Core Principles for Beginners

1. **Regions = ownership units**. Cannot copy, can move, or borrow.
2. **Mutable pointers (`*T@R`) = temporary access**. Compiler tracks last use.
3. **Ownership moves (`T@R`) = transfer value**. Aliases invalidated automatically.
4. **Sequential mutation safe** due to RAII-style lifetime tracking.
5. **Parallel mutation safe** via value-level exclusivity.

> This model avoids verbose annotations, explicit lifetimes, or `writes(heap)` style declarations, while remaining **low-level and safe**.

---

### 6. Optional / Advanced Features

* `unsafe { ... }` block allows bypassing alias/move rules explicitly.
* User-defined regions for scratch memory or persistent storage.
* Shared immutable regions for thread-safe access.

---

### 7. Summary

Forge’s memory and concurrency model revolves around **regions, value-level ownership, and RAII-style lifetimes**. This provides:

* Compile-time safety
* Beginner-friendly syntax
* Support for low-level systems programming

> Core syntax: `*T@R` for pointers, `T@R` for ownership move, RAII-style alias drop, and region types controlling lifetime and concurrency.
