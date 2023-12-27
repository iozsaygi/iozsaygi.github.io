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
We'll first implement signatures for the function we will be using. Do not forget to include ``windows.h`` to access Windows API functions for thread management.