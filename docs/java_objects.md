---
title: "The Invisible Steps: How Java Really Allocates Your Objects" 
date: 2025-07-16
math: true
draft: false
---

In Java, when you allocate an object using the `new` keyword, a complex series of steps takes place behind the scenes involving **bytecode interpretation or JIT compilation, memory management, and synchronization with the garbage collector**.

## 1. `new` Bytecode Instruction

When you write:

```java
A obj = new A();
````

The Java compiler compiles this to JVM bytecode like:

```
new A           // allocate memory, pushes uninitialized object reference onto stack
dup             // duplicates the reference, one for the constructor, one for the variable 'obj'
invokespecial   // calls the constructor (e.g., A.<init>())
astore_1        // stores the object reference into local variable 'obj'
```

This sequence of bytecodes is then executed by either the interpreter or the Just-In-Time (JIT) compiled code.

---
## 2. Class Metadata Lookup

The JVM looks up the **class metadata** (loaded in the method area or metaspace, depending on the JVM) for class `A`. This metadata includes:
- **Size of the object**: How much memory is needed for an instance of `A`.
- **Field layout**: Where each field is located within the object's memory.
- **Pointer to the class vtable (virtual method table)**: Used for dynamic dispatch of methods.
- **Constructor information**: Details about how to initialize the object.

**Note:** If the class isn't loaded yet, it triggers **class loading, linking, and initialization**. This involves:
- **Loading:** Finding the `.class` file and loading its bytecode.
- **Linking:**
    - **Verification:** Ensuring the bytecode is valid and safe.
    - **Preparation:** Allocating memory for static fields and initializing them to default values.
    - **Resolution:** Replacing symbolic references with direct references (e.g., method calls to actual memory addresses).
- **Initialization:** Executing the class's static initializers and static blocks.
---
## 3. Memory Allocation

After getting information about class A. The actual memory for the object is allocated **in the heap**, usually in the **young generation (Eden space)**. There are two common strategies:
### (I) Thread-Local Allocation Buffers (TLABs)
In a multi-threaded JVM environment, if all threads were to allocate memory directly into a **shared heap**, it would require **synchronization (locking)** on each allocation to avoid conflicts (i.e., multiple threads writing to the same region of memory). This synchronization would be a significant performance bottleneck.
A **TLAB** is a small chunk of the heap that the JVM assigns **exclusively to a single thread**. The thread can then allocate memory within its TLAB **without any synchronization**, just by bumping a pointer.
#### TLAB Assignment:
When a thread starts or when it needs to allocate memory:
- The JVM allocates a fixed-size memory region from the heap (usually from the **Eden** space in the Young Generation)
- This memory region becomes the **TLAB** for that thread.
#### Allocation Inside TLAB:
Now the thread can allocate objects by **incrementing a pointer**:

```
// Pseudo-mechanism
object_address = TLAB.top;
TLAB.top += object_size;
```
- No locks needed.
- Super-fast — it's just a pointer move.
#### TLAB Exhaustion:
If the thread's TLAB runs out of space:
- The JVM tries to allocate a **new TLAB** for the thread (again from Eden).
- If Eden is also full, **garbage collection** may be triggered to free up space.
### (II) Global Allocation (if TLAB is full or disabled)

- If a TLAB is exhausted and a new one cannot be obtained, or if the object being allocated is larger than the maximum TLAB size, or if TLABs are disabled (which is rare in modern JVMs), the JVM falls back to a **synchronized heap allocation** path.
- This involves acquiring a global heap lock or using atomic operations (like Compare-And-Swap, CAS) to ensure thread safety when multiple threads try to allocate directly from the shared heap. This is significantly slower than TLAB allocation.

Note :Most JVMs (like **HotSpot**, **OpenJ9**) use **TLABs** to allow fast object allocation without locking. Each thread gets a small chunk of the heap, and the `new` operation simply bumps a pointer inside the TLAB — **very fast**.

---
## 4. Zeroing and Initialization

After memory is successfully allocated:
- The memory for the new object is **zeroed out** (all bits set to 0). This means primitive fields are set to their default values (e.g., `0` for `int`, `0.0` for `double`, `false` for `boolean`), and reference fields are set to `null`. This is a security and correctness measure, preventing uninitialized memory from being exposed to Java code.
- The **object header** is initialized. This header contains crucial metadata about the object, including:
    - **Mark Word:** Stores hash code, GC age, and locking information (used for `synchronized` blocks and biased locking).
    - **Class Pointer (Klass Pointer in HotSpot, Type Pointer in OpenJ9):** A pointer to the object's class metadata in the Method Area/Metaspace, allowing the JVM to know the object's type, its fields, and methods.

---

## 5. Constructor Invocation

- The `invokespecial` bytecode instruction, which was placed on the stack by the compiler, is now executed. This calls the actual Java constructor of the class.
- The constructor then initializes the fields with programmer-defined values, overriding the default zeroed values. For example, if a field `int x = 10;` is declared, the constructor will set `x` to `10`.

---

## 6. Write Barrier (Optional)

In some modern garbage collectors (like G1, ZGC, Shenandoah in **HotSpot**; various concurrent GC policies in **OpenJ9**), a **write barrier** may be applied when storing object references.

- A write barrier is a small piece of code inserted by the JIT compiler before or after an object reference is written to a field. Its purpose is to track changes in the object graph that are relevant to concurrent garbage collection algorithms.
    
- For example, in generational garbage collectors, write barriers are used to maintain "remembered sets" or "card tables," which track references from older generation objects to younger generation objects. This helps the GC efficiently find roots for minor collections without scanning the entire heap. In concurrent collectors like ZGC or Shenandoah, write barriers are used to ensure the consistency of the heap view during concurrent marking or compaction phases.

[More Details](https://adityaanand7.github.io/research/general/write_barriers/)

---

## 7. Escape Analysis & Stack Allocation (Optimization)

This is a powerful optimization performed by the JIT compiler.

- The JIT compiler performs **escape analysis** to determine if an object's lifetime is confined to a single method or thread. If the object never "escapes" the method it's created in (i.e., its reference is not returned, stored in a field, or passed to other threads), then the JVM might apply further optimizations.
    
- If an object is determined to be "non-escaping," the JVM can employ **scalar replacement** or **stack allocation**:
    
    - **Scalar Replacement:** Instead of allocating the entire object on the heap, the JIT compiler might break the object down into its constituent fields (scalars) and treat them as independent local variables. These scalar variables can then be allocated directly on the thread's stack. This entirely avoids heap allocation, reducing GC pressure and improving cache locality.
        
    - **Stack Allocation:** In some cases, the entire object _might_ be allocated on the stack. In those cases instead of breaking the object whole object is allocated on the stack frame of its allocating method. **OpenJ9** extensively uses this optimization.

    Want to learn more about the optimization `stack allocation`. Read our paper ["Optimistic Stack Allocation and Dynamic Heapification in Managed Runtimes."](https://dl.acm.org/doi/pdf/10.1145/3656389)
