---
layout: post
title: Breadth first search
description: Reviewing the "breadth first search" algorithm.
tags:
  - Algorithms
---
There are many search algorithms for many cases. Lately, I've been working on a gameplay system that heavily relies on search algorithms that can work on a graph-based structure. 

Most of the time, I was able to get away with a breadth-first search. In this post, I will try to explain breadth first search by including some C programming.

### Organizing the data
To run BFS, we need to store nodes, adjacency lists, and graphs in structures. Let's take a look at the following structures that we defined:

```c
struct bfs_node {
    int data;
    struct bfs_node* next;  
};  
  
struct bfs_list {  
    struct bfs_node* head;  
};  
  
struct bfs_graph {  
    int nodeCount;  
    struct bfs_list* list;  
};
```