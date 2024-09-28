# Java - Memory management

## Key points
- **Garbage collector** is in charge of freeing the memory when not required anymore.
- Programmers don't have to manage memory but should know how the GC works to write *high performance programs* and avoid *memory leaks*.

## Java Memory structure

|Area   |Description  | Role  |
|-------|--------------|-------|
|Heap   |• Instanciated during the JVM startup.<br/>• Only 1 per JVM process.|• Stores *class instances* and *arrays*.<br/>• References are stored in **stack**.<br/><br/>`MyClass c = new MyClass()` creates the `MyClass` instance `c` in the *heap* and the reference `c` in the *stack*.|
|Method|• Created on JVM startup.|• Stores *class structures*, *method data*, *constructor field data*, *interfaces* and *special methods*.*|
|JVM Stack   |• Created when a **Thread** is created.<br/>|• Stores data and partial results.   |
|Native method Stack   |• Created for each **Thread** when created.<br/>• Non-Java||

## Garbage collector
- **Costly** because it causes threads and processes to be **paused**.
  - There are *garbage collector tuning* algorithms to optimize this process.
  - For example, *generational garbage collector* that adds **age** to objects and group them so that the garbage collection work can be **distributed**.
  - Currently, *generational garbage collector* is used.
- We can use `System.gc()` or `Runtime.gc()` to request garbage collection but **the final decision is up to the JVM**.