---
title: "Golang's Garbage Collection Simplified"
draft: false
date: "2024-07-08 06:00:00"
---


Garbage collection (GC) is one of those magical features in Go that helps manage memory automatically, so we don’t have to manually allocate and deallocate memory. Understanding how Go’s GC works can help you write more efficient and optimized code. Let's dive into the basics of Go's garbage collector, simplified for entry-level Go engineers.

## What is Garbage Collection?

Garbage collection is a form of automatic memory management. The garbage collector attempts to reclaim memory occupied by objects that are no longer in use by the program, preventing memory leaks and improving performance.

## How Go's Garbage Collector Works

Go uses a **concurrent mark-and-sweep** garbage collection algorithm. This means it runs alongside your program (concurrent) and involves two main phases: **mark** and **sweep**.

### 1. Mark Phase

During the mark phase, the GC identifies which objects are still in use. It starts from a set of root objects (global variables, stack variables, and CPU registers) and marks all reachable objects.

**Example: Marking Reachable Objects**

```go
package main

import "fmt"

type Node struct {
    value int
    next  *Node
}

func main() {
    root := &Node{value: 1}
    second := &Node{value: 2}
    root.next = second
    fmt.Println(root, second)
    // Here, root and second are reachable and will be marked
}
```

In this example, the `root` node points to the `second` node. During the mark phase, both `root` and `second` are reachable and will be marked by the garbage collector.

### 2. Sweep Phase

In the sweep phase, the GC reclaims memory occupied by objects that are no longer marked as reachable.

**Example: Sweeping Unreachable Objects**

```go
package main

import "fmt"

func main() {
    a := make([]int, 10)
    b := make([]int, 20)
    a = nil // a is now eligible for garbage collection
    fmt.Println(b)
    // Here, the memory for `a` will be reclaimed in the sweep phase
}
```

In this example, the slice `a` is set to `nil`, making it eligible for garbage collection. The memory occupied by `a` will be reclaimed in the sweep phase.

## Advanced Concepts

### Pointers vs Non-Pointers

Pointers reference other objects, making those objects reachable. Non-pointers (simple values) do not reference other objects and hence do not affect reachability.

**Pointers Example:**

```go
package main

import "fmt"

type Node struct {
    value int
    next  *Node
}

func main() {
    root := &Node{value: 1}
    second := &Node{value: 2}
    root.next = second
    fmt.Println(root, second)
    root = nil // root is set to nil, making the node unreachable
    runtime.GC() // Trigger garbage collection
    fmt.Println("Garbage collection triggered")
}
```

In this example, setting `root` to `nil` makes the `Node` structure it pointed to unreachable. The garbage collector will clean up this memory during its sweep phase.

**Non-Pointers Example:**

```go
package main

import "fmt"

func main() {
    a := 10
    b := 20
    c := a + b
    fmt.Println(c) // c is a non-pointer value
    runtime.GC()   // Trigger garbage collection
    fmt.Println("Garbage collection triggered")
}
```

In this example, variables `a`, `b`, and `c` are simple integer values. The garbage collector does not need to do anything special for these non-pointer values as they do not reference other objects.

### Variables Used in Multiple Goroutines

When variables are used across multiple goroutines, Go's garbage collector ensures proper synchronization to prevent premature collection of shared objects.

**Example: Variables in Multiple Goroutines**

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

type SharedData struct {
    value int
}

func main() {
    var wg sync.WaitGroup
    shared := &SharedData{value: 42}

    for i := 0; i < 5; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            fmt.Printf("Goroutine %d: %d\n", id, shared.value)
            time.Sleep(1 * time.Second)
        }(i)
    }

    wg.Wait()
    shared = nil // shared is set to nil, making the object unreachable
    runtime.GC() // Trigger garbage collection
    fmt.Println("Garbage collection triggered")
}
```

In this example, the `SharedData` struct is used across multiple goroutines. The garbage collector tracks such references to ensure that the shared object is not collected prematurely. After all goroutines complete and the `shared` variable is set to `nil`, the object becomes unreachable and can be collected.

### Handling Object References

Go uses a write barrier to manage references during the mark phase. This ensures that any changes to object references are correctly tracked.

**Example: Write Barrier**

```go
package main

import (
    "fmt"
    "runtime"
)

type Node struct {
    value int
    next  *Node
}

func main() {
    root := &Node{value: 1}
    second := &Node{value: 2}
    third := &Node{value: 3}
    root.next = second

    runtime.GC() // Trigger garbage collection before updating references
    fmt.Println("Garbage collection triggered before update")

    // Update references
    second.next = third

    runtime.GC() // Trigger garbage collection after updating references
    fmt.Println("Garbage collection triggered after update")
}
```

In this example, the write barrier ensures that the change in `second.next` is properly tracked by the garbage collector, preventing the `third` node from being prematurely collected.

## Conclusion

Understanding Go's garbage collector helps in writing efficient and optimized code. By knowing how it handles pointers, non-pointers, and variables across goroutines, you can better manage memory and improve the performance of your Go programs. Happy coding!
