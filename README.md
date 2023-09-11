# Assignment 8: Memory Layout (Continued)

This assignment continues our discussion on the memory layout. From the diagram below, we covered
the text segment, the data segment, the BSS segment, and the stack segment in the previous
assignment. We will cover the kernel address space, the memory mapping segment, and the heap segment
in this assignment.

```bash
.──────────────────────.
|                      | Address 2^n - 1
| Kernel address space | (n is the number of address bits)
|                      |
+──────────────────────+
|                      |
| Stack                |
| (grows ↓)            |
|                      |
+──────────────────────+
|                      |
|                      |
+──────────────────────+
|                      |
| Memory mapping       |
| (grows ↓)            |
|                      |
+──────────────────────+
|                      |
|                      |
+──────────────────────+
|                      |
| Heap                 |
| (grows ↑)            |
|                      |
+──────────────────────+
|                      |
+──────────────────────+
|                      |
| BSS                  |
| (Uninitialized       |
|  global              |
|  or static data)     |
|                      |
+──────────────────────+
|                      |
| Data                 |
| (Initialized         |
|  global              |
|  or static data)     |
|                      |
+──────────────────────+
|                      |
| Text (Code)          |
|                      |
+──────────────────────+
|                      |
|                      |
|                      | Address 0
'──────────────────────'
```

## Kernel Address Space

As discussed in the first lecture, a typical operating system operates in two modes---kernel mode
and user mode. User applications, such as your web browser, terminal, file explorer, etc., all run
in user mode. Core kernel components, such as the file system, the network stack, device drivers,
etc., all run in kernel mode. The kernel address space is only used by the kernel components and
user programs cannot access it. If a user program accesses this address space, the OS will terminate
the program with a segmentation fault. Although there is much to be said about the kernel address
space, we will not get into the discussion because the scope of this course is user-level systems
programming.

## Task 0: Understanding the Memory Mapping Segment

Memory mapping is also a big topic and we will discuss it separately later in the semester. For now,
let's look at one important component in this region, which is shared libraries. If you remember
from our earlier discussion, when an OS starts executing a program, the OS first loads the program
into memory. When that happens, the OS dynamically links the program with the shared libraries that
the program needs. The memory mapping region, among other things, stores these shared libraries.
Let's do a simple activity to understand this more.

Create a file named `memory_mapping.c` and write the following program. Create a `Makefile` that has
a target named `memory_mapping` that is produced by `make memory_mapping`. As always, make sure that
you are doing this in the same directory as this `README.md` is in. Also make sure that you
`record`.

```c
#include <stdio.h>

int foo(void) {
  printf("foo() address:      %p\n", foo);
}

int main(void) {
  int stack_var = 0;

  printf("main() address:     %p\n", main);
  printf("printf() address:   %p\n", printf);
  printf("stack_var address:  %p\n", &stack_var);
  foo();
}
```

As you may recall, `printf()` is a function in the C standard library or `libc`, and it is
dynamically linked to your program because it is shared. Thus, if we print out the address of
`printf()` like above, we should see an address that is somewhere in between the `main()` function
(which resides in the text segment) and `stack_var` (which resides in the stack segment).

To verify this, compile and run the program. You should see an output similar to the following:

```bash
main() address:     0xaaaacbe607b8
printf() address:   0xffff9b8d09d0
stack_var address:  0xffffdb569e1c
foo() address:      0xaaaacbe60794
```

As the addresses show, `printf()` is located in between `main()` and `stack_var`. Furthermore,
`foo()` is located very closely to `main()` because it also resides in the text segment.

(You can stop recording and come back later, or continue on.)

## Task 1: Understanding the Heap Segment

*The heap segment* or simply *the heap* is a region of memory used when a program calls memory
allocation functions. As you may know, there are four popular C standard library functions related
to this, `malloc()`, `calloc()`, `realloc()`, and `free()`. It is **extremely important** to
understand how to use these functions correctly and what the implications are.

Let's first examine the signatures of these functions.

```c
void *malloc(size_t size);
void *calloc(size_t nmemb, size_t size);
void *realloc(void *ptr, size_t size);
void free(void *ptr);
```

Here are a few important points about these functions.

* `malloc()` takes a single argument, `size`, and allocates `size` bytes of memory on the heap.
* `calloc()`, on the other hand, takes two arguments, `nmemb` and `size`, and allocates an array of
  `nmemb` elements, where each element is of `size` bytes.
* `realloc()` takes two arguments, `ptr` and `size`, and changes the size of previously allocated
  memory pointed to by `ptr` to `size` bytes.
* `free()` takes a single argument, `ptr`, and frees memory pointed to by `ptr`.
* `malloc()` does not initialize the allocated memory, while `calloc()` does with 0. Thus, reading
  `malloc()`-allocated memory is *undefined behavior* and returns garbage values, while reading
  `calloc()`-allocated memory returns 0. For `realloc()`, if the new size is larger than the old
  size, the newly-allocated portion is not initialized.
* `malloc()`, `calloc()`, and `realloc() all return a pointer to the *lowest* address of the
  allocated memory. On error, or the requested size is 0, they return `NULL`. Thus, it is *very*
  important to check if `malloc()`, `calloc()`, or `realloc()` returns `NULL` and handle it
  appropriately.
* As the diagram above shows, the heap segment is placed in between the BSS segment and the memory
  mapping segment, and it grows upward. This means that as your program requests access to more
  memory on the heap using functions like `malloc()`, the size of the heap grows from a lower
  address to a higher address.
* Under the hood, `malloc()`, `free()`, etc. are all part of `libc` (the C Standard Library) and
  allow your code to interact with a *memory allocator* implemented in `libc`. Memory allocation
  (with `malloc()`, etc.) is essentially a process where the memory allocator marks a portion of
  memory as "in use" and returns the lowest address of the portion of the memory. Memory
  deallocation (with `free()`) is essentially a process where the memory allocator marks the portion
  of memory (pointed to by the argument of `free()`) as "available". In other words, the memory
  allocator keeps track of which heap space is already used or available for future use.

Let's do a few activities to understand the heap more. Make sure you `record` if you are not
recording already.

### Accessing the Heap

The first important thing to understand regarding the heap is that you need to use a *pointer
variable* to access the heap, i.e., your access to the heap is *indirect*. This means that you use a
variable located in some other segments, e.g., the stack, to access the heap segment. To understand
this more, consider the following code. (You don't need to write this yourself.)

```c
#include <stdio.h>
#include <stdlib.h>

int main(void) {
  int *heap_ptr = calloc(1, sizeof(int));
  if (heap_ptr == NULL) {
    perror("calloc() failed");
    exit(1);
  }
  printf("heap_ptr address: %p\n", &heap_ptr);
  printf("heap_ptr value:   %p\n", heap_ptr);
  printf("*heap_ptr value:  %d\n", *heap_ptr);
}
```

If you run this code, you will get an output similar to the following:

```bash
heap_ptr address: 0xfffff5ba4898
heap_ptr value:   0xaaaaf70c12a0
*heap_ptr value:  0
```

The purpose of this is to explain the following points.

* The variable `heap_ptr` is a local variable and a pointer.
* You use `heap_ptr` to access the heap. In other words, your access to the heap is indirect through
  a pointer located outside the heap.
* The value of `heap_ptr` is an address within the range of addresses of the heap segment. The code
  is asking `calloc()` to allocate one `int`-sized block of memory. Then `calloc()` returns a memory
  address that points to the newly-allocated 4 bytes of memory on the heap.
* The address of the variable `heap_ptr` is a higher value than the value of the variable. This is
  because the stack occupies higher addresses than the heap in memory.

The visualization of this is the following.

```bash
| ...              |
+──────────────────+
| 0xaaab0f1ff2a0   | 0xffffc1b17618 (the location of `head_ptr`)
| (Stack, grows ↓) |
+──────────────────+
| ...              |
+──────────────────+
| (Heap, grows ↑)  |
| 0x0              | 0xaaab0f1ff2a3
| 0x0              | 0xaaab0f1ff2a2
| 0x0              | 0xaaab0f1ff2a1
| 0x0              | 0xaaab0f1ff2a0 (where `head_ptr` points to)
+──────────────────+
| ...              |
```

As shown, your access to the heap is via a pointer variable.

### Manual Memory Management and Memory Allocator

Using the heap in a C/C++ program is considered *manual* memory management. This is because you need
to write a piece of code yourself to request memory allocation on the heap (by calling functions
like `malloc()`). In addition, you need to write a piece of code yourself to release the memory (by
calling `free()`).

This is different from using other segments with local and global variables. You never allocate or
deallocate memory space for local or global variables by calling allocation or deallocation
functions. You just declare those variables and the memory space is managed for you. If you remember
from earlier assignments, global variables use the BSS and the data segments, and they are there
*all the time*. The BSS and data segments get allocated when the program starts to run and they
never get deallocated. Local variables, on the other hand, use a stack frame and every time a new
function is called, a stack frame is allocated for the local variables in the function. When the
function execution is done, the entire stack frame is popped (i.e., deallocated) from the stack.
Because of this, you do not need to worry about allocating or deallocating memory space for local
variables.

Since you have to manually manage the heap memory, it is extremely easy to make mistakes and manage
the heap incorrectly. Let's do a few activities to understand this more.

### Memory Leak

The most common mistake is called *memory leak*, which occurs when you allocate memory on the heap
but don't deallocate it properly. To understand this more, let's create a file named `memory_leak.c`
and add a new target `memory_leak` in the Makefile you created earlier. Write the following program.

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

char *readUserInput() {
  // Allocate a buffer
  size_t bufferSize = 100;
  char *buffer = (char *)malloc(bufferSize * sizeof(char));

  // Check if the allocation was successful
  if (buffer == NULL) {
    perror("malloc() failed");
    return NULL;
  }

  // Read a user input
  printf("Enter your input: ");
  if (fgets(buffer, bufferSize, stdin) == NULL) {
    perror("fgets() failed");
    return NULL;
  }

  // Return the buffer that contains the user input
  return buffer;
}

int main() {
  char *input = readUserInput();

  if (input != NULL) {
    printf("You entered: %s\n", input);
    free(input);
  }

  return 0;
}
```

Once you write the program, our linter will immediately complain that there is a potential leak and
the location is inside the `fgets()`'s `if` block. If you look at the code carefully, you will also
realize that we are not freeing the memory correctly there. The function allocates memory with
`malloc()` but when `fgets()` fails to read a user input, the whole `readUserInput()` function
returns `NULL` without freeing the memory. What this means is that the `malloc()`-allocated memory
is still marked as "in use" and the memory allocator will not see it as available memory. In other
words, by not freeing the memory properly, we have effective reduced the size of memory usable by
the memory allocator. In this particular example, this is less of a problem because the program will
terminate soon after the function returns. However, if this is a long-running program and
`readUserInput()` is repeatedly called with failed `fgets()`, we will keep reducing the size of the
heap memory usable by the memory allocator.

Note that the above code demonsrates only one example. There are other ways to leak memory, e.g.,
assigning a new value to a pointer without freeing the memory it previously pointed to.

Since this is a common problem, `AddressSanitizer` implements checks for it. As usual, you can add
`-fsanitize=address` as a Clang option to enable it. Generally, you should use this and other
sanitizers when developing software since they can detect common and serious problems such as memory
leaks.

In order to avoid leaking memory, you need to trace *all possible execution paths* in your program
and make sure that you free memory on all of those. Programs have multiple possible execution paths
when they have conditionals or loops. We can easily visualize this by drawing a diagram that shows
all possible execution paths. The diagram below is an example of the above function
`readUserInput()`.

```bash
                     +-----------------+
                     | Allocate Memory |
                     | (malloc)        |
                     +-----------------+
                              |
                              v
                     +------------------+
                     | Check Allocation |
                     | (buffer == NULL) |
                     +------------------+
     (Allocation Failed)      |   (Allocation Succeeded)
              +---------------+--------------+
              |                              |
              v                              v
 +----------------------+          +-----------------+
 | Print Error Message  |          | Read User Input |
 | ("malloc() failed")  |          | (fgets)         |
 | and Return NULL      |          +-----------------+
 +----------------------+                    |
              +---------------+--------------+
              |                              |
              v (Read Succeeded)             v (Read Failed)
    +-------------------------+    +-----------------------+
    | Return Allocated Buffer |    | Print Error Message   |
    | (Buffer Later Freed)    |    | ("fgets() failed")    |
    +-------------------------+    | and Return NULL       |
                                   +-----------------------+
```

If you examine all possible paths as the diagram shows, you can check which path is missing a
`free()` call.

As a side note, it is generally not a good idea to allocate a buffer and return it inside a
function. It is much better to receive an allocated buffer as an argument, fill it, and return it.
This makes it clear which function needs to take care of buffer allocation, deallocation, and
errors. In the above program, this responsibility is distributed across both `main()` and
`readUserIput()`, which makes it difficult for a programmer to mentally keep track.

### Use After Free, Double Free, and Null Pointer Dereference

*Use after free* refers to a case where you free a previously-allocated memory block and then access
it again by mistake. *Double free* refers to a case where you free a previously-allocated memory
block and then free it again by mistake. *Null pointer dereference* refers to a case where you use a
null pointer by mistake as if it had a regular address value. All of these are *undefined behavior*
and common sources of bugs and security vulnerabilities. It is absolutely critical to avoid these
issues when you write your code.

To experiment with one of the problems, use after free, create a file named `use_after_free.c` and
also add a Makefile target with the same name `use_after_free`. Enable `AddressSanitizer` for Clang
as we want to check a use-after-free error. Write the following program in `use_after_free.c`.

```c
#include <stdio.h>
#include <stdlib.h>

void free_memory(int *ptr) {
  free(ptr); // Free the memory
}

int main() {
  int *ptr = (int *)malloc(sizeof(int));

  if (ptr == NULL) {
    perror("Memory allocation failed");
    return 1;
  }

  *ptr = 42;

  free_memory(ptr);
  printf("Value after freeing: %d\n", *ptr); // Accessing freed memory

  return 0;
}
```

Compile and run it. `AddressSanitizer` should detect a problem and display an error message
regarding `heap-use-after-free`.

This is a contrived example to just demonstrate the problem. However, it is not difficult to imagine
a scenario where you pass a buffer to a function, and the function deallocates the buffer either by
mistake or as part of its normal execution. When the function returns, you might still use the
buffer without realizing that the buffer has been freed. (Again, it is generally not a good idea to
distribute the buffer management responsibility across different functions.)

You can easily come up with simple examples of double free and null pointer dereference to
experiment with those problems and see if our linters and sanitizers can detect those problems. We
highly encourage you to do so.

## Memory Safety

Generally, all of the problems related to memory we discussed so far are called *memory safety*
issues. These include stack corruption, buffer overflow, uninitialized variable access, memory leak,
use after free, double free, and null pointer dereference. All of these issues lead to bugs and
vulnerabilities and we need to make every effort to avoid such problems.

The best practices to avoid memory safety problems are as follows.

* Always initialize pointers and variables before using them.
* Make sure to check whether or not a pointer is NULL before using it.
* Ensure proper memory allocation and deallocation using `malloc()`, `calloc()`, `realloc()`, and
  `free()`.
* Use functions that prevent buffer overflows and overreads, like `fgets()`, `strncpy()`, and
  `snprintf()`.
* Always make sure to check array bounds.
* Avoid using pointers to already-deallocated memory.
* Employ static analysis tools (e.g., linters) and memory checkers (e.g., sanitizers) to identify
  potential issues.

It is worth noting that directly manipulating memory is a feature only available in low-level,
systems programming languages like C/C++, Rust, etc. It is not possible in other higher-level
languages like Java or Python. These higher-level languages come with a component called *language
runtime* that performs many tasks on behalf of your program including memory management. This is a
tradeoff---they provide convenience and more safety at the expense of performance and flexibility.
Therefore, when you choose a language for a programming task, you need to make a decision based on
the features your task requires.

Now, you can stop recording and submit all the files you created for this assignment including
`.record/` and `.nvim/`.

# Next Steps

You need to accept the invite for the next assignment (A9).

* Get the URL for A9. Go to the URL on your browser.
* Accept the invite for Assignment 9 (A9).
* If you are not in `units/03-memory` directory, go to that directory.
* Clone the assignment repo.
