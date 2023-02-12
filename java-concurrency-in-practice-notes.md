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

## 6. Task execution
- concurrent applications are organized around the execution of tasks. Task should be independent and has clear boundaries. Good example of task is single request to the server
- threads need to be created in *bounded* amount, creating too much requests may harm the program. Also creating each thread brings some overhead (time, stack memory consumption / thread creation in general is an expensive operation)
- executor pattern is commonly used ot decouple task submission and its execution
- *ExecutorService*:
    - owns thread pool where incoming tasks can be executed 
    - provide shutdown policy - immediate or gracefull(do not schedule any new tasks, allow already running tasks to be completed)
- data rendering (example of a program where extracting parallel tasks is not that obvious): 
    - use single thread, render all text at first (leave blanks for images) and then fetch and render images (naive)
    - use two threads and two independent tasks, first to fetch end render text, and second to fetch end render images, when reader reaches image that is not yet rendered it wait until completion
    - event better option is to download each image in parallel, and publish downloaded images to queue so that whenever download is completed it gets rendered
    - conclusions: improvement in parallel processing is gained only when we have multiple, fine-grained and independent tasks that can be divided between multiple workers
- sometimes there is no point in waiting for task completion longer as the result is no longer needed, that's why *timing* feature of parallel, *Feature* tasks is important, after timeout all resources used by thread should be freed

## 7. Cancellation and shutdown
 - naive apporach to task cancellation will be to use boolen flag and pool change of that flag, however it will not work for blocking tasks, as the task will be not able to detect the change
 - that's why interruptioins were introduced, it's a way to notify one thread by another that it is exepcted to stop execution. JVM doesn't guarantee how fast blocking task will be interruped but it happens reasonably fast. For threads in blocked state *InterruptedException* is thrown, but in remaining cases interruption simply sets the flag, and it's running code responsiblity to react to that flag
 - interruption is just an request that the thread interrupt itself at the next convenient opportunity 
 - aproach that most of libraries take is throwing *InterruptedException* in response to an interrupt. Task get out of the way as quikly as possible and communicate the interruption back to the caller so that code higher up to the call stack can take further action. Exception can be thrown immediately or after the task completes some action.
 - there should be no assumption made on interruption handling policy between task and thread that run that task (they should enclose that logic into their own scope)
 - encapsulation practices dictate that you should not manipulate an thread that you do not own
 - one way to stop a thread consuming queue is to send poison pill message to that queue
 - when thread handling task queue is stopped we probably want to get tasks that were not completed, and resume them in another thread. Thats why the status of that tasks is returned, and that's also argument for having tasks indempotent (as the task that was started but not completed on the first thread can then run again on another thread). 
 - well behaving task should never cause thread to fail, espectially when tasks are delegated to thread pools. On the other side thread pools should be resistant to such tasks and try to catch all unhandled runtime exception. Handler for such a tasks should be registered, by default it is simply `print(...)`
 - JVM shoutdown is initiated when last nondaemon thread termiantes or for example `System.exit` appears. First all registered *shutdown hooks* are called. Shutdown hooks should be thread safe. Deamon threads are normal threads, with the only difference that JVM shout them down immediately, regardless of the state they are in (they are used for example for garbage collection). 
 - using interruption for anything but cancellation should be avoided
 
## 8. Applying thread pools
- thread pools work best when tasks are homogeneeous and independent
- thread pool with too small amount of resources can stuck when used for long-running task 
- size of thread pool should be strictly related to number of processors, so that all resources are fully consumed. The more blocking task we perform the higher should be amount of threads we use as threads that perform blocking operation will be not schedulable. Even if we have very computation extensive operations we should add another extra thread in case if one of the runners fail
- tasks that are waiting for assignment to thread pool are stored in a queue. Queue can be bounded or unbounded, bounded ones have saturation policies assigned - default one is to discard task and throw an exception
- real thread pools implementations have dynamic size, they can decrease number of threads when they are idling or increase when the load is high

## 9. GUI aplications
- multithreading in the GUI is problematic as we have two directional flows, user actions are detected by OS and propagated to app, and tha app initiates some actions in response that are propagated to OS. It very often leads to deadlocks
- long running tasks are typically delegated into background worker threads. They are decomposed into multiple subtasks that are scheduled after completion of some steps (initiation, waiting, presenting the results)
- thread safe data model:
    - versioned collections: copies are created when data is updated so that current iterator rely on the same data they saw at the beginning of the interation
    - split data models: data split into two layers, presentation and shared, shared one is thread-safe and lies in the background. Updates are done safely to shared layer, shared layer creates update events that represent changes and they that are passed to presentation layer

# 10. Avoid  liveness hazards
- locks are quite common in database transactions and thats way they are automatically detected and recovered from. Transactions that cause locks are aborted
- the program is free of deadlocks if all thread acqurie resources in the same, globally specified order
- good idea is to sort objects that we want to acquire lock on by some globally unique value like object hash, this will alow to avoid any deadlocks
- its risky to calling external method from synchronized method as it can try to acquire locks that we already hold, should be done carefully! *Open calls* is technique that states that we should always call external methods without keeping any lock on our side 
- keeping synchronized block small also help to avoid locking situations
- timed locking is a technique to timeout the lock acquisition if it was not possible in requested time period
- dead-locks can be debugged with help of thread dumps
- starvation happens when thread needs to wait long time to receive resource it needs. *Thread API* provides control over thread scheduling priorities, but it should be used carefully as it may lead to confusing situations, another negative aspect is that JVM thread prorities are mapped to OS thread priorities in different ways, so the program can behave differently on different platforms
- *livelock* is a situation when threads are not truly blocked but they still cannot make progress because they perform operation that always fail (eg. try to apply recovery mechanism for unrecoverable error)
