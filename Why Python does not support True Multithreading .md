# Can Multiple Threads in a Single Process Be Executed in Parallel on Multiple Cores?

Yes, **multiple threads** in a single process can be executed **in parallel on multiple cores**, but whether this actually happens depends on a few factors, particularly the **Global Interpreter Lock (GIL)** in languages like **Python**. 

## 1. In General (Non-Python Environments)
In languages like **Java**, **C++**, **Go**, and **Rust**, **multiple threads** can indeed run in parallel across multiple CPU cores. Here's how it works:

- **Threads** are lightweight units of execution within a process, and a **process** can have multiple threads running concurrently.
- If a system has **multiple cores**, an operating system (OS) with proper thread management can assign different threads to different cores.
- This allows true **parallelism**: multiple threads in the same process can run on different cores simultaneously. For example, on a quad-core machine, you can have up to 4 threads running at the same time, assuming no other system constraints.

## 2. Python and the Global Interpreter Lock (GIL)
Python, however, has a **Global Interpreter Lock (GIL)** that **limits true parallel execution** of threads, but this is specific to **CPython**, the reference implementation of Python.

- **GIL**: The GIL ensures that only one thread executes Python bytecode at a time. Even though you can create multiple threads in a Python process, only one thread can execute Python code at any given moment.
- **Why the GIL exists**: It simplifies memory management and garbage collection in CPython, but at the cost of parallelism in multi-threaded applications.
  
In **CPU-bound** tasks (tasks that require a lot of computation), **Python threads** do **not run in parallel** because of the GIL. However, in **I/O-bound** tasks (like waiting for data from a network or file system), Python threads can still be useful, as the GIL is released during I/O operations, allowing other threads to run while waiting for the I/O operation to complete.



## 2.Why GIL exists in Python 
Python developed in 1991 was developed when CPUs used to be mostly single core. So to keep a very simple GC model a simple memory model was introduced where objects themselves keep a track of references that ar eporinting to them  :
1. Reference counts - Every object in memory keeps a count of how many things are referring to it. The moment the reference count for an object hits 0, a calls the __del__() of that object which deallocates the memory it acquired. 
2. Cyclic refereces - They are removed in GCs (we will come to them later)

While developing python the focus was more on optimizing on the speed of memroy deallocation rather than multi core usage (this did not even exist then). If multi threading as a core concept was to be introduced in python, it would have required to accquire a lock on the object first and then delete it (as other thread might be just about to refer to it), hence not introducing multi-threading was a choice to optimize memory mgtm efficiently in terms of time and resource (obtaininga  lock would be resource and time intensive).

Now since Python wanted a model that best supported for CPU of a single core and has super fast deallocation of memory using reference counts (to optimize on memory part) hence it 

```python
a = ["apple"]
b = a

# while request is being served the reference count for a = 1 and for b = 0
# one the request is served, the reference count of a is 0 and so for b = 0
```
3. Cyclic counts - For objects that are referenceing to themselves, these are run again by the request therad (not everytime but when a certain threshhold of time or memory allocxation is breached) where it checks for cycles and if it can be deleted or not. 

```python
a = ["apple"]
a.append(a)


# one the request is served, the reference count to a will be 1 at any given pont of time.
```



## 3. Workaround for Parallelism in Python
For **true parallelism** in Python (especially for CPU-bound tasks), there are a few approaches:

### Multiprocessing
Instead of using threads, you can use the `multiprocessing` module in Python, which creates **separate processes**, each with its own Python interpreter and memory space. Each process can run on a different core and can truly execute in parallel.

```python
import multiprocessing

def task():
    print("Task is running")

if __name__ == "__main__":
    processes = []
    for _ in range(4):
        process = multiprocessing.Process(target=task)
        processes.append(process)
        process.start()

    for process in processes:
        process.join()
