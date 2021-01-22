# Java Memory Allocation: Stack vs Heap and other areas


![Image for post](/home/ejungon/Documents/收集的文章/Java Memory Allocation.assets/1*rwdye-8w97N-v9I8p6q3AQ.png)

Image from [Jamesdbloom blod](https://blog.jamesdbloom.com/JVMInternals.html#constant_pool)

As a developer, one should know about the memory allocations that how  variables, function calls, execution instructions or object creation  does and what goes where in memory. As a Java developer, we should know  about the **Java Virtual Machine** (JVM) internals that how the run-time data areas works so that we have better grip over the common errors like **OutOfMemoryError** or **StackOverflowError** and also to improve the app performance. We will learn about the types of run-time memory areas and how they work.

## Terms for understanding:

- **JVM (Java Virtual Machine)** is by definition a virtual machine that simulates what a real machine  does. It has an instruction set, a virtual computer architecture and an  execution model. It is capable of running code written with this virtual instruction set.
- **HotSpot** is an implementation of JVM concept or specification. It was originally developed by Sun and now it is owned by Oracle. There are other  implementations of the JVM specification, like [**JRockit**](http://en.wikipedia.org/wiki/JRockit), [**IBM J9**](http://en.wikipedia.org/wiki/IBM_J9), among many **[**[**others**](https://en.wikipedia.org/wiki/List_of_Java_virtual_machines)**]**.

# Types of JVM run-time data areas:

1. **Program Counter (PC) Register**
2. **JVM Stack (Stack memory)**
3. **Native Method Area**
4. **Heap Memory**
5. **Method Area**
6. **Run-time Constant Pool**

The above six types can be classified into 2 groups.

- First three areas are **Per-Thread** area
- Last three areas are **Shared-to-all-Thread** areas.

## **Per-Thread Area:**

It means the elements or contents are only accessible to the current thread only.

## Shared-to-all-Thread Area:

It means anyone can read or write the element or content of this areas.

# 1. Program Counter (PC) Register

PC Register stores the memory address of JVM instructions currently  executed. This is like a pointer to the current instruction in sequence  of instructions of a program. As Java supports multi-threading, a PC  Register is created every time when a thread is created. Once the thread is finished, The PC Register will also be destroyed. That’s why, its  categorized in **Per-Thread Area** group.

> If the currently executing method is ***native method\***, the PC Register will be undefined.

# 2. JVM Stack (Stack Memory)

Like PC Register, JVM Stack is also created when a thread is created. That’s why, its also grouped in **Per-Thread Area**. Each JVM Stack contains multiple block of JVM frames that stores  method-specific values that are short-lived. It stores the local  primitives and reference variables and follows the **LIFO (Last-In-First-Out)** principle so the currently executed method is at the top in the stack.  When a method is invoked, a stack frame is created and pushed in the  stack that stores the information of invoked method. As soon as method  finishes execution, the stack frame will be vanished and when a thread  is finished, the stack memory will be reclaimed. This memory is **thread-safe** as each thread has its own stack. Memory size can be managed in two ways, **Fixed** and **Dynamic** (expand as per needed).

The total memory allocated to JVM Stack is very less as compared to **Heap Memory.** It throws **java.lang.StackOverflowError** when the memory is full. It usually happens on recursive operations.

## JVM Stack size

> “Many Java Virtual Machine publishers reduced the default size of a thread’s call stack from 1MB to 256KB.”

Each thread will have the same amount of stack memory size. The default size can be customized by command **-Xss**

![Image for post](/home/ejungon/Documents/收集的文章/Java Memory Allocation.assets/1*A5bXTdz0UZ_FzO1XrXOwWA.png)

# 3. Native Method Area

Like Stack memory, JVM allocates memory area for native methods also  (methods that written other than Java). Its created per-thread. When a  native method executes, all thread related data are saved into this  area. Native method stack is only available when JVM supports native  methods otherwise it is not available. Likewise JVM Stack, it also can  be of two types: fixed and dynamic.

# 4. Heap Memory

Heap area is created at VM (not JVM but OS VM) startup. The JVM allocates  Java heap memory from the OS and then manages the heap space for the  Java application. Whenever an object is created of classes or arrays.  The JVM sub-allocates a contiguous area of the heap memory to store it.  Heap area is the primary storage inside JVM for the runtime data and **shared to all thread**. As the heap memory is shared to all threads, anyone can create into or access from this area. This is where a **garbage collection (GC)** comes timely and checks for the unused objects and remove them to  claiming the memory back and this is one of the best feature in Java.

## Memory Generations:

HotSpot VM’s garbage collector uses generational garbage collection. It separates the JVM’s memory into **young generation** and **old generation.**

## **Young Generation:**

It is further divided into two parts: **Eden** and **Survivor.** Every object starts its life from **Eden space.** When GC does his job, it moves alive objects from Eden space to Survivor space and removed the other un-referenced objects.

## Old Generation:

It also has two parts: **Tenured** and **Permanent** (PermGen)**.** GC moves alive objects from Survivor to Tenured area. And the permanent generation contains the metadata of the classes, methods and the VM.

## **Code-Cache (Virtual or reserved)**

The HotSpot Java VM also includes a code cache, containing memory that is used for compilation and storage of native code.

![Image for post](/home/ejungon/Documents/收集的文章/Java Memory Allocation.assets/1*UgLvuUz7EuuIZXYS17Wr9w.png)

Image by[ Java Honk](http://javahonk.com/how-many-types-memory-areas-allocated-by-jvm/)

# 5. Method Area

Method area is created at the JVM startup and shared to all threads. It is a  logical part of heap area and give controls to JVM implementer to decide not mandated by the specification that means its size can be decided  according to requirements. It contains **per-class** structures and fields like method local data, static fields, method and constructor codes and runtime constant pool OR simply **type/class information**. It throws **OutOfMemoryError** if its area is insufficient during runtime. Though its a part of heap, It may or may not be garbage collected.

# 6. Run-time Constants Pool

It is part of every `.class` file that contains constants needed to run the code of that class. It is  created by the compiler and maintained by JVM that resolves the  references of the constant pool during runtime.

These constants include literals specified by the programmer and symbolic  references generated by compiler. Symbolic references are basically  names of classes, methods and fields referenced from the code. These  references are used by the JVM to link your code to other classes it  depends on.

Consider an example from stack-overflow:

![Image for post](/home/ejungon/Documents/收集的文章/Java Memory Allocation.assets/1*sUQmNLhWyRwT0R85HmAq-Q.png)

Example by [axtavt](https://stackoverflow.com/users/103154/axtavt)

That’s all. Hope you will have grabbed some information from this article.

If you want to go more in details about the JVM Internals, you can go [**here**](https://blog.jamesdbloom.com/JVMInternals.html)