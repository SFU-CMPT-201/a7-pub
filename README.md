# Assignment 7: Memory Layout (Continued), Data Types, and Alignment

This assignment has two parts---the first part continues the discussion on the memory laout. The
second part discusses data types and memory alignment.

## Important Note

You ***should not*** copy and paste the code or command examples. You have to type them yourself
and understand the meanings. If you copy and paste, you ***will not*** get any credit for this
assignment.

## Part 0: Memory Layout (Continued)

Let's start with the first part. From the diagram below, we covered the text segment, the data
segment, the BSS segment, and the stack segment in the previous assignment. We will cover the
kernel address space, the memory mapping segment, and the heap segment.

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

### Kernel Address Space

As discussed in the first lecture, a typical operating system operates in two modes---kernel mode
and user mode. User applications, such as your web browser, terminal, file explorer, etc., all run
in user mode. Core kernel components, such as the file system, the network stack, device drivers,
etc., all run in kernel mode. The kernel address space is only used by the kernel components and
user programs cannot access it. If a user program accesses this address space, the OS will terminate
the program with a segmentation fault. Although there is much to be said about the kernel address
space, we will not get into the discussion because the scope of this course is user-level systems
programming.

### Task 0: Understanding the Memory Mapping Segment

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

void foo(void) {
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

### Task 1: Understanding the Heap Segment

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

* `malloc()` takes a single argument, `size`, and allocates `size` bytes of memory on the heap. It
  returns `void *`, which is a generic memory address pointer.
* `calloc()`, on the other hand, takes two arguments, `nmemb` and `size`, and allocates an array of
  `nmemb` elements, where each element is of `size` bytes. It also returns `void *`. Again, `void *`
  is a generic memory address pointer.
* `realloc()` takes two arguments, `ptr` and `size`, and changes the size of previously allocated
  memory pointed to by `ptr` to `size` bytes.
* `free()` takes a single argument, `ptr`, and frees memory pointed to by `ptr`.
* `malloc()` does not initialize the allocated memory, while `calloc()` does with 0. Thus, reading
  `malloc()`-allocated memory is *undefined behavior* and returns garbage values, while reading
  `calloc()`-allocated memory returns 0. For `realloc()`, if the new size is larger than the old
  size, the newly-allocated portion is not initialized.
* `malloc()`, `calloc()`, and `realloc()` all return a pointer to the *lowest* address of the
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

#### Accessing the Heap

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

#### Manual Memory Management and Memory Allocator

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

#### Memory Leak

The most common mistake is called *memory leak*, which occurs when you allocate memory on the heap
but don't deallocate it properly. To understand this more, let's create a file named `memory_leak.c`
and add a new target `memory_leak` in the Makefile you created earlier. Write the following program.

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

char *read_user_input() {
  // Allocate a buffer
  size_t buffer_size = 100;
  char *buffer = (char *)malloc(buffer_size * sizeof(char));

  // Check if the allocation was successful
  if (buffer == NULL) {
    perror("malloc() failed");
    return NULL;
  }

  // Read a user input
  printf("Enter your input: ");
  if (fgets(buffer, buffer_size, stdin) == NULL) {
    perror("fgets() failed");
    return NULL;
  }

  // Return the buffer that contains the user input
  return buffer;
}

int main() {
  char *input = read_user_input();

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
`malloc()` but when `fgets()` fails to read a user input, the whole `read_user_input()` function
returns `NULL` without freeing the memory. What this means is that the `malloc()`-allocated memory
is still marked as "in use" and the memory allocator will not see it as available memory. In other
words, by not freeing the memory properly, we have effectively reduced the size of memory usable by
the memory allocator. In this particular example, this is less of a problem because the program will
terminate soon after the function returns. However, if this is a long-running program and
`read_user_input()` is repeatedly called with failed `fgets()`, we will keep reducing the size of
the heap memory usable by the memory allocator.

Note that the above code demonstrates only one example. There are other ways to leak memory, e.g.,
assigning a new value to a pointer without freeing the memory it previously pointed to.

Since this is a common problem, `AddressSanitizer` implements checks for it. As usual, you can add
`-fsanitize=address` as a Clang option to enable it. (Another well-known tool you can use is
[Valgrind](https://valgrind.org/), but `AddressSanitizer` does similar things and [is often much
faster](https://developers.redhat.com/blog/2021/05/05/memory-error-checking-in-c-and-c-comparing-sanitizers-and-valgrind#performance).) Generally, you should use sanitizers such as
`AddressSanitizer`, `UndefinedBehaviorSanitizer`, `ThreadSanitizer`, when developing software since
they can detect common and serious problems such as memory leaks. However, remember from earlier
assignments that sanitizers only detect problems at run time *if they get triggered*. Thus, you also
need to provide right inputs that trigger problems, with separate mechanisms such as fuzzing or
manually-crafted test cases.

In order to avoid leaking memory, you need to trace *all possible execution paths* in your program
and make sure that you free memory on all of those. Programs have multiple possible execution paths
when they have conditionals or loops. We can easily visualize this by drawing a diagram that shows
all possible execution paths. The diagram below is an example of the above function
`read_user_input()`.

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
`read_user_input()`, which makes it difficult for a programmer to mentally keep track.

#### Use After Free, Double Free, and Null Pointer Dereference

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

### Memory Safety

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
runtime* that performs many tasks on behalf of your program including memory management. Although
they do not completely eliminate memory safety problems (e.g., memory leak), they do improve memory
safety significantly. This is a trade-off---higher-level languages provide convenience and more
safety at the expense of performance and flexibility. Therefore, when you choose a language for a
programming task, you need to make a decision based on the features your task requires.

## Part 1: Data Types, Alignment, Padding, and Packing

One of the trickiest parts of writing low-level programs that directly manipulate memory is that
often times, you need to think about byte-level details. In the previous assignments, we
side-stepped this issue because we mostly used a single-byte type such as `uint8_t`. However, C has
many types of various sizes and also allows you to define your own custom types with `struct`,
`union`, `enum`, and `typedef`. When you use these types, it is important to understand how these
types are represented in memory and how to use these using pointers. In this assignment, you will
learn more about this.

### Task 0: Representing Data Types in Memory

One way to conceptualize a type is to think of it as imposing a structure over *raw bytes*. (We're
using "structure" in a general sense here, not as a reference to `struct`.) In other words, if you
have a consecutive series of bytes, how you interpret the bytes depends on the type you use to
access it. Even with the same set of bytes, you can interpret it differently using different types.
Type casting allows you to achieve this, although it has limitations---it's not always possible to
perform type casting on all types. For example, user-defined custom types like `struct` do not
permit direct type casting. However, using pointers provides us with much more flexibility.

To understand this, create a file named `types.c` and also a Makefile that includes a target named
`types`. Make sure you `record` as well. Write the following program in `types.c`.

```c
#include <stdint.h>
#include <stdio.h>

struct as_struct {
  uint16_t l2b; // the lowest two bytes
  uint8_t n1b;  // the next one byte
  uint8_t h1b;  // the highest one byte
};

union as_union {
  uint16_t r2b; // reading two bytes
  uint32_t r4b; // reading four bytes
  uint8_t r1b;  // reading one byte
};

int main(void) {
  uint32_t i = 0xBEEFCAFE;
  uint16_t *ptr16 = (uint16_t *)&i;
  struct as_struct *as = (struct as_struct *)&i;
  union as_union *au = (union as_union *)&i;

  printf("i: 0x%X\n", i);
  printf("ptr16[0]: 0x%hX, ptr16[1]: 0x%hX\n", ptr16[0], ptr16[1]);
  printf("lowest two bytes: 0x%hX, next one byte: 0x%hhX, highest one byte: "
         "0x%hhX\n",
         as->l2b, as->n1b, as->h1b);
  printf(
      "reading four bytes: 0x%X, reading two bytes: 0x%hX, reading one byte: "
      "0x%hhX\n",
      au->r4b, au->r2b, au->r1b);
}
```

Compile and run the code. You should get the following output:

```bash
i: 0xBEEFCAFE
ptr16[0]: 0xCAFE, ptr16[1]: 0xBEEF
lowest two bytes: 0xCAFE, next one byte: 0xEF, highest one byte: 0xBE
reading four bytes: 0xBEEFCAFE, reading two bytes: 0xCAFE, reading one byte: 0xFE
```

In the code, we interpret the same block of 4 bytes in four different ways by imposing four
different structures (i.e., types) over the block. The first is `uint32_t`, the second is an array
of `uint16_t`, the third is a `struct`, and the fourth is a `union`. The diagram below visualizes
this. The leftmost side shows the highest address byte and the rightmost side shows the lowest
address byte.

```bash
                   (highest)                    (lowest)
As raw (four) bytes:   | 0xBE | 0xEF | 0xCA | 0xFE |

As `uint32_t`:         |------------ i ------------|

As `uint16_t`:         |-- ptr16[1] -|-- ptr16[0] -|

As `struct as_struct`: |- h1b-|- n1b-|---- l2b ----|

As `union as_union`:   |----------- r4b -----------|
                                     |---- r2b ----|
                                            |- r1b-|
```

Before we examine each case, let's first discuss the rule that determines how we read the bytes in a
*single* type. This rule is called *byte ordering* or *endianness*. There are two popular approaches
to this, called *little-endian* and *big-endian*.

* In a little-endian system, the least significant byte (the "little end") of a multi-byte value is
  stored at the lowest memory address.
* In a big-endian system, the most significant byte (the "big end") of a multi-byte value is stored
  at the lowest memory address.

Our VM is a little-endian system because it is based on x86-64. Thus, `int i` in the above code is
store as shown below (which is also shown in the diagram above). The least significant byte is
stored at the lowest memory address.

```bash
                   (highest)                    (lowest)
As raw (four) bytes:   | 0xBE | 0xEF | 0xCA | 0xFE |
```

If it were a big-endian system, it would look like the following.

```bash
                   (highest)                    (lowest)
As raw (four) bytes:   | 0xFE | 0xCA | 0xEF | 0xBE |
```

This rule applies not only to `int` but also to all other multi-byte types, e.g., `short`,
`int16_t`, `long`, etc. However, it is only for a single type, not for a composite type such as
`struct`.

Now, let's go back to our code and examine the four cases of interpreting bytes.

* `uint32_t i`: In the code, we first define a 4-byte variable (`i`) and initialize the
  corresponding 4-byte block with the value of `0xBEEFCAFE`. This means that whenever we use `i`, we
  access those 4 bytes as a single unit.
* `uint16_t *ptr16 = (uint16_t *)&i`: By assigning the address of `i` to a `uint16_t` pointer
  (`ptr16`), we are basically saying that we want to interpret the memory beginning at the address
  of `i` as a 2-byte value (`uint16_t`). This means that when we access the memory pointed to by
  `ptr16` (i.e., `*ptr16` or `ptr16[0]`), we access two bytes as a single unit. We can do this even
  though those two bytes are part of the larger 4-byte block that belongs to `i`. Later in the code,
  we even use `ptr16` as an array and access the next two bytes.
* `struct as_struct *as`: In this case, we impose a 4-byte `struct` over the memory beginning at the
  address of `i`. In C, the first member of a `struct` occupies the lowest address, while the last
  member of a `struct` occupies the highest. Employing a `struct` in this manner enables us to group
  individual bytes in different ways. In our example, we group the lowest two bytes together (as a
  `uint16_t`, while keeping the highest two bytes separate (as a `uint8_t`).
* `union as_union *au`: Similar to the `struct` case, we impose a 4-byte `union` over the memory
  beginning at the address of `i`. Since all members of a `union` occupy the same memory space, each
  member reads from the lowest byte.

The following code demonstrates that we can do the exact same thing with heap-allocated memory. (You
do not need to try the code.)

```c
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>

struct as_struct {
  uint16_t l2b; // the lowest two bytes
  uint8_t n1b;  // the next one byte
  uint8_t h1b;  // the highest one byte
};

union as_union {
  uint16_t r2b; // reading two bytes
  uint32_t r4b; // reading four bytes
  uint8_t r1b;  // reading one byte
};

int main(void) {
  void *void_ptr = malloc(4);                           // Heap allocation of raw 4 bytes
  uint32_t *ptr32 = (uint32_t *)void_ptr;               // Using uint32_t
  uint16_t *ptr16 = (uint16_t *)void_ptr;               // Using uint16_t
  struct as_struct *as = (struct as_struct *)void_ptr;  // Using struct as_struct
  union as_union *au = (union as_union *)void_ptr;      // Using as_union
  *ptr32 = 0xBEEFCAFE;

  printf("*ptr32: 0x%X\n", *ptr32);
  printf("ptr16[0]: 0x%hX, ptr16[1]: 0x%hX\n", ptr16[0], ptr16[1]);
  printf("lowest two bytes: 0x%hX, next one byte: 0x%hhX, highest one byte: "
         "0x%hhX\n",
         as->l2b, as->n1b, as->h1b);
  printf(
      "reading four bytes: 0x%X, reading two bytes: 0x%hX, reading one byte: "
      "0x%hhX\n",
      au->r4b, au->r2b, au->r1b);
}
```

The only difference between the above code and the code earlier is the use of the stack vs. the
heap. In the code, we allocate 4 bytes in the heap and then use four different types to interpret
the same bytes differently. The interpretations themselves are exactly the same as the earlier code.

One important caveat here is that this kind of byte re-interpretation is *very rare* and you should
avoid it unless it is absolutely necessary. We discuss it here to help you deepen your understanding
of low-level memory manipulation. However, this often leads to brittle code that does not work
across different platforms or when compiled with different compilers, which the next task discusses.

### Task 1: Alignment, Padding, and Packing

Another important aspect of low-level memory manipulation is memory alignment, padding, and packing,
particularly related to `struct`s. Let's explore each of these concepts.

#### Memory Alignment and Padding

Memory alignment refers to the requirement that certain data types should be stored in memory at
addresses that are multiples of their size in bytes. This alignment requirement is imposed by
hardware, and violating it can lead to performance penalties or even program crashes on some
CPU architectures.

For example, on our VM (based on x86-64):

* A `char` can be stored at any memory address.
* A `short` (2 bytes) should be stored at an address that is a multiple of 2 (2-byte aligned).
* An `int` and a `float` (4 bytes) should be stored at an address that is a multiple of 4 (4-byte
  aligned).
* A `long`, a `double`, and any pointer type (8 bytes) should be stored at an address that is a
  multiple of 8 (8-byte aligned).

Padding, then, is the process of adding unused bytes (padding bytes) to a `struct` to satisfy
alignment requirements. These padding bytes are inserted by the compiler to ensure that each member
of the `struct` is properly aligned.

To experiment with this, create a file named `alignment.c` and also a Makefile that includes a
target named `alignment`. Make sure you `record` as well. Write the following program in
`alignment.c`.

```c
#include <stdio.h>

struct align_example {
  char a;   // 1 byte
  int b;    // 4 bytes (requires 4-byte alignment)
  short c;  // 2 bytes (requires 2-byte alignemnt)
  double d; // 8 bytes (requires 8-byte alignment)
};

void align(void) {
  struct align_example ex = {'0', 0, 0, 0.1};

  printf("sizeof(char): %lu, sizeof(int): %lu, sizeof(short): %lu, "
         "sizeof(double): %lu\n",
         sizeof(char), sizeof(int), sizeof(short), sizeof(double));
  printf("ex.a address: %p\nex.b address: %p\nex.c address: %p\nex.d address: "
         "%p\n",
         (void *)&ex.a, (void *)&ex.b, (void *)&ex.c, (void *)&ex.d);
  printf("sizeof(struct align_example): %lu\n", sizeof(struct align_example));
}

int main(void) { align(); }
```

Compile it and run it. You will get an output similar to the following:

```bash
sizeof(char): 1, sizeof(int): 4, sizeof(short): 2, sizeof(double): 8
ex.a address: 0xfffffa151cd0
ex.b address: 0xfffffa151cd4
ex.c address: 0xfffffa151cd8
ex.d address: 0xfffffa151ce0
sizeof(struct align_example): 24
```

In the code, we print out the sizes of `char`, `int`, `short`, and `double` to confirm their sizes.

The output shows that the compiler adds padding bytes between `a` and `b` to ensure that `b` is
4-byte aligned, and between `c` and `d` to ensure that `d` is 8-byte aligned.

* `ex.a` is at `0xfffffa151cd0` and it is a 1-byte type (`char`), but `ex.b` does not start at the
  next byte (`0xfffffa151cd1`). Instead, it starts at `0xfffffa151cd4` due to the 4-byte alignment
  requirement. (It is the next possible address that is a multiple of 4.)
* Between `ex.b` and `ex.c`, there is no padding necessary because `ex.c` is a 2-byte type and the
  next byte after `ex.b` (`0xfffffa151cd8`) satisfies the 2-byte alignment requirement, i.e., the
  address is a multiple of 2.
* `ex.c` is at `0xfffffa151cd8` and it is a 2-byte type, but `ex.d` does not start at
  `0xfffffa151cda`. Instead, it starts at `0xfffffa151ce0` due to the 8-byte alignment requirement.
  (It is the next possible address that is a multiple of 8.)
* Hence, the size of this `struct` is larger than the sum of the individual members due to alignment
  requirements.

There are two important implications of this.

* When you allocate a `struct` in the heap, you should not manually count the number of bytes based
  on the members. You should always use `sizeof()`, e.g., `malloc(sizeof(struct
  alignment_example))`.
* This alignment and padding behavior depends on the compiler/platform/architecture. If your code
  makes certain assumptions about this behavior, it is likely to not work if you use a different
  compiler or run the code on a different platform. Thus, it is very important to understand this
  clearly before doing anything with it (e.g., performing byte re-interpretation as we did earlier).

#### Packing

Packing is the process of eliminating or minimizing padding within a `struct` to reduce its size in
memory. This can be useful in situations where memory efficiency is critical, such as when dealing
with data structures for file I/O or network communication.

You can use compiler-specific directives or attributes to control packing. For example, in Clang,
you can use `#pragma pack` to control `struct` packing.

To experiment with this, add the following `struct` and function in the file you created earlier
(`alignment.c`) and call the function from its `main()`.

```c
#pragma pack(push, 1) // Push current packing setting and set packing to 1 byte alignment
struct pack_example {
  char a;   // 1 byte
  int b;    // 4 bytes (no padding now)
  short c;  // 2 bytes
  double d; // 8 bytes (no padding now)
};
#pragma pack(pop) // Restore previous packing setting

void pack(void) {
  struct pack_example ex = {'0', 0, 0, 0.1};

  printf("ex.a address: %p\nex.b address: %p\nex.c address: %p\nex.d address: "
         "%p\n",
         (void *)&ex.a, (void *)&ex.b, (void *)&ex.c, (void *)&ex.d);
  printf("sizeof(struct pack_example): %lu\n", sizeof(struct pack_example));
}
```

Compile and run it. You should get an output similar to the following.

```bash
sizeof(char): 1, sizeof(int): 4, sizeof(short): 2, sizeof(double): 8
ex.a address: 0xffffe25bbc50
ex.b address: 0xffffe25bbc54
ex.c address: 0xffffe25bbc58
ex.d address: 0xffffe25bbc60
sizeof(struct align_example): 24
ex.a address: 0xffffe25bbc60
ex.b address: 0xffffe25bbc61
ex.c address: 0xffffe25bbc65
ex.d address: 0xffffe25bbc67
sizeof(struct pack_example): 15
```

In this code, the `#pragma pack` directives instruct the compiler to pack the `struct` members
without adding padding bytes between them. As the output shows, there is no padding between member
addresses and the size of `struct pack_example` is the sum of the sizes of its members.

Note that packing `struct`s can lead to reduced performance on some architectures because unaligned
memory accesses may be slower. For example, if a single memory read/write is aligned at 8-byte
boundaries and if you pack your memory in a way that a single unit (e.g., an `int` or a `long`)
crosses a boundary, you may need to perform two memory operations to read/write it instead of one.
Therefore, packing should be used judiciously, e.g., when memory efficiency is more critical than
performance.

Once everything is done, make sure you stop recording.

## Submission

Make sure you use git to push all the files/directories you created, including `.nvim/` and
`.record/`, for grading.

As with previous assignments, make sure you run `a7-checker` to check if your work is ready for
grading. `a7-checker` performs the following.
* It checks if you have correct recording files. You will receive a 0 if this fails.
* It checks if you have used `nvim` correctly. You will receive a 0 if this fails.
* It flags if you have copied and pasted anything. You will receive a 0 if this fails.
* It performs some basic checks for the tasks above.
* Note that if a check fails, it does not always tell you why. You need to debug it by yourself
  and find out.
* The checker is not a grader but if it says that everything is good and if you are done with
  the above tasks, you will get the full credit.
* It is entirely *your responsibility* to run `a7-checker` before submitting your work. If you
  don't do it, you may get an unexpected grade.
