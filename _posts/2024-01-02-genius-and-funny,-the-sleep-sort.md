---
layout: post
title: Genius and funny, the sleep sort
description: Reviewing the genius sleep sort algorithm in C
tags:
  - Algorithms
---
I was playing around with sorting algorithms and then I saw the humorous sleep sort algorithm. The one that creates a thread for each element in the array. Yes, to sort an array, we are going to create a new thread for each element, and each thread sleeps for an amount of time that is equal to the value of the corresponding array element. It sounds crazy, but it just works.

Let's take a closer look at this algorithm and try to implement it in C by using the Windows API for threading.

### Function definitions
We'll first implement signatures for the functions we will be using. Do not forget to include ``windows.h`` to access Windows API functions for thread management.

#### 1. Thread function
Here's the first function that we will be implementing; it will be called from a separate thread that will be created later on. Basically, we are taking data as ``void*``, but we will cast it to ``int*`` to sleep the thread.
```c
unsigned int __stdcall sleepSort_thread(void* data);
```

Also, I was wondering what the meaning of ``__stdcall`` was during the implementation. It turns out to be a Microsoft-specific keyword for the compiler. Will be declared as NULL for other environments, like UNIX.

It specifies that the callee is responsible for the stack clean-up; a more detailed explanation can be found [here](https://en.wikipedia.org/wiki/X86_calling_conventions).

#### 2. Thread manager function
This is the actual function that will create a separate thread for each element in the list.
```c
void sleepSort_execute(int* array, size_t length);
```

It takes an array pointer and requires the length of the array in order to create the correct number of threads. Very basic.

### Function implementations
Now that we have defined function signatures, let's take a look at actual implementations. Lots of comment lines can be found for further explanation.

#### 1. Thread function
Here is the implementation of the function that will be executed on a separate thread.
```c
unsigned int __stdcall sleepSort_thread(void* data) {
    // Cast the given void pointer into integer pointer to figure out sleep duration.
    int* cast = (int*) data;

    // Assign the actual duration of sleep to some integer variable.
    int sec = *cast;

    // Sleep the thread by assigned second value with Windows API function.
    Sleep(sec * 1000);

    // Print the value after sleep process ends for thread.
    printf("%d, ", sec);
}
```

Basically, we are casting void pointer into integer pointer to figure out how many seconds a thread will be suspended. After that, we are calculating the seconds by multiplying the casted value by ``1000``, then actually pausing the thread by using the ``Windows`` API function. Finally, printing the casted value after sleeping.

#### 2. Thread manager function
Now let's take a look at the actual function that will manage the threads. (Well, porting this to UNIX is still a to-do of mine.) 
```c
void sleepSort_execute(int* array, size_t length) {
    assert(array != NULL);

    // Create thread handle buffer to store thread for each element in the array.
    HANDLE* threadHandles = (HANDLE*) malloc(sizeof(HANDLE) * length);
    assert(threadHandles != NULL);

    // Start a new thread for each element in the array.
    for (size_t i = 0; i < length; i++) {
        threadHandles[i] = (HANDLE) _beginthreadex(NULL, 0, sleepSort_thread, &array[i], 0, NULL);
        assert(threadHandles[i] != NULL);
    }

    // Wait for all threads to finish.
    WaitForMultipleObjects((DWORD) length, threadHandles, TRUE, INFINITE);

    // Close allocated thread handles.
    for (size_t i = 0; i < length; i++) {
        CloseHandle(threadHandles[i]);
    }

    // Clear resources allocated for the thread handles.
    free(threadHandles);
}
```

The code is pretty well documented, but let's have a general overview of it.
- We are starting by creating a ``HANDLE`` buffer to store threads that we'll be creating for each element in the array.
- For each element in the array, we are creating a new thread and assigning that thread to the related thread handle.
- Then we'll simply wait for all threads to finish executing. Each thread will sleep for the duration of the exact element that it is working with. It means a single thread is going to wait three seconds if the data that is passed is an integer with a value of three.
- After we're done waiting for each thread, we'll close the threads by shutting down the related thread handles.
- Finally, we are clearing the resources we allocated for threads.

### Resources
- [Sleep Sort - The King of Laziness](https://www.geeksforgeeks.org/sleep-sort-king-laziness-sorting-sleeping/)
- [x86 calling conventions](https://en.wikipedia.org/wiki/X86_calling_conventions)
- [What is __stdcall?](https://stackoverflow.com/questions/297654/what-is-stdcall)

### Conclusion
I wasn't expecting to meet with Sleep Sort while I was studying algorithms. It was a really fun thing to implement. It also gave me a rough introduction to multi-threading in C. The full implementation is available in my [data structures and algorithms](https://github.com/iozsaygi/dsaa/blob/main/src/sleep_sort.c) repository. 

Also, before ending it up, this blog was the first post of 2024, opening the year with data structures and algorithms. Pretty legit way to start a year, I guess.

Thank you for tuning in!