# Java concurrency in practice

## 1. Introduction
- history: in the past only one program was run on the computer. When OS was introduced it enabled cooperation of multiple programs on single machine. OS and multiple programs provides fairness, full resources utilization and convenience. 
  - threads: share memory, file handlers but have own stack, program counters and local variables
- existence of threads solves the problem of idling during IO actions
- makes is possible to decouple complex programs into smaller tasks that can be run independently
- livenes hazards: e.g. infinite pool or dead locks = program doesn’t move forward
- performance hazards: e.g costs of context switching 
- threads over abstractions are used in many frameworks

## 2. Thread safety
- if multiple threads access the same mutable variable without synchronization program becomes broken, to avoid that we can:
    - don't share the state variable across threads 
    - make the state immutable
    - use synchronization whenever accessing the state
- OO principles like immutability and encapsulation helps to write thread safe code
- class is thread-safe if it behaves correctly when accessed from multiple threads, regardles of how they use that class (serially or interleaving)
- stateless objects are always thread-safe
- atomicity = closing all operations on an object in one action, other thread cannot see state of that object during modifications of that action
- *race condition* occurs when the correctness of a computation depends on the relative timing or interleaving of multiple threads by the runtime
- having compund actions that have more that one steps usage of atomic data structures in not sufficient as it still can cause race conditions, we need lock with broader scope
- *synchronized* block has two parts: reference to an object that will serve as the lock and a block of code that will be guarded by that block (static synchronized methods take thc Class object for the lock )
- reentracy: when the same thread calls more than one synchronized methos on a single object it is not blocked
- every shared, mutable variable should be guarded by exactly one locka (commong pattern is to synchornize path that accesses that variable)
- synchronized block should be as small as possible and contain no long running operations, avoid network or I/O calls!
- *writing correct concurrent programs is primarly about managing access to shared, mutable state*

## 3. Sharing objects
- we want not only to prevent one thread from modyfing the state of an object but also ensure that when a thread modifies the state of an object, other threads can *see* the changes that were made
- in the absence of synchronization the compiler, processor and runtime can do some downrigh wierd things to the order in which operations appear to execute
- it's not safe to use nonvolatile *long* and *double* values as 64 bit read can be trated as two indpendent 32 bit operations that can represent two different states (state can change between these tow operation - so the first 32 bits can be from old state and another 32 bits from the new state)
- when threads *synchronize* on a common lock they all see the most up-to-date values of shared mutable vairiable
- *volatile* variable notifies compiler and runtime that this variable is shared and operations on it should not be reordered with other memory operations. Read of a volatile varialbe always returns the most recent write by any thread. Use cases for *volatile* are completion, interruption or status flags.
- keep objects confined to single thread
- stack confinement is a special case of thread confinement in which an object can only be reached through local variables. It happens by default when we use primitively typed local vairables and for objects we need to store references in local variables and not let them to escape. 
- ThreadLocal<T> allows you associate a per-thread value and can be conceptually treated as Map<Thread, T>. *get* always returns value recently passed to *set* from the currently executing thread. Commonly used to keep transaction state in different frameworks.
- *final* fields cannot be modified, it is a good practice to make all fields *final* unless they need to be mutable 
- to provide atomicity we can wrap data we want to operate on atomically with immutable data class e.g. to provide atomic write to two variables put them into signle immutable object and read/write consistent values from it
- thread-safe collection ensures that every element written by thread A is visible in subsequent read from thread B
 
## 4. Composing objects
- object fields repersent the state it holds, that state should be encapsulated and thread-safe interface that transforms the object only between valid states should be exposed
- encapsulating data within an object confines access to the data to the object's methods - making it easier to control
- common pattern to convert non-thread-safe classes like e.g. collections into thread-safe is to add a wrapper that will synchronize access to inner state - it's called *monitor pattern* 
- sometimes it is a good practice to use private lock object created inside the class to prevent external user from acquiring lock that synchronizes class state (disallow invalid usage)
- if a class is composed of multiple independent thread-safe state variables and has no operation that have any invalid state transitions, then it can delegate thread safety to the underlying state variables
- when exteding a thread-safe class we should reuse the same synchronization mechanism to protect added methods. Also when adding helper method to let's say *list* in some *ListHelper* it's incorrect to use *synchronized(this)* as we want to lock on *list* instead of *ListHelper* (use *synchronized(list)* instead)
- document a class's thread safety guarantees for its clients ; documet its synchronization policy for its maintainers

## 5. Building blocks
- synchronized collections are thread-safe but still can cause some isses, for example they don't prevent the situation when one thread modifies collection and another deletes some items in the same time, this can lead to error
- concurrent collections are fully save to use, error mentioned above doesn't occur, but view of the collection can be temporarily unconsistent or stale
- to enable concurrent processing we can use producer/consumer pattern, it splits computations in two parts - that decouples them and allow to run in parralel. Objects can be passed from producer to consumer via queue, it is thread-safe as these threads accesses shared object sequentially
- synchronizers:
    - latches: delay the progress of threads until it reaches terminal state. Once latch reaches terminal sstate it cannot change state again
    - *FeatueTask*
    - semaphores
    - barriers
- efficient, scalable result cache:
    - naive implementation is to use *HashMap* with *synchronized* block covering computation (it's not effective as only one thread can perform operations at single time)
    - better approach is to use *ConcurrentHashMap* (problem: if two threads request the same value at the same time they can duplicate computation)
    - to improve that we can replace values with *Future* to mark that some value is already beeing calculated, inserting of *Future* should be done in synchronized block
