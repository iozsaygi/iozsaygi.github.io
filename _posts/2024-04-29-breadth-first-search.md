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

### Allocation
We need a way to handle the allocation of nodes in our graph. It is pretty straight-forward.
```cpp
struct bfs_node* bfs_createNode(int data) {  
    struct bfs_node* node = (struct bfs_node*) malloc(sizeof(struct bfs_node));  
    node->data = data;  
    node->next = NULL;  
    return node;  
}

struct bfs_graph* bfs_createGraph(int nodeCount) {  
    assert(nodeCount >= 0);  
  
    struct bfs_graph* graph = (struct bfs_graph*) malloc(sizeof(struct bfs_graph));  
    graph->nodeCount = nodeCount;  
    graph->list = (struct bfs_list*) malloc(nodeCount * sizeof(struct bfs_list));  
  
    for (int i = 0; i < nodeCount; i++) {  
        graph->list[i].head = NULL;  
    }  
  
    return graph;  
}  
  
void bfs_addEdge(struct bfs_graph* graph, int source, int destination) {  
    assert(graph != NULL);  
  
    struct bfs_node* node = bfs_createNode(destination);  
    node->next = graph->list[source].head;  
    graph->list[source].head = node;  
  
    node = bfs_createNode(source);  
    node->next = graph->list[destination].head;  
    graph->list[destination].head = node;  
}
```

Well, to sum up the allocations:
1. We are allocating nodes (or vertices) with the given data.
2. Then we have a utility function to create the actual graph; it also creates an empty adjacency list.
3. At last, there is a utility to add an edge node to the graph.

### Execution
The BFS itself is pretty straight-forward; I used [this](https://en.wikipedia.org/wiki/Breadth-first_search) page a lot during the implementation. Since C has no built-in queue, I had to simulate it with basic data types.
```cpp
void bfs_execute(struct bfs_graph* graph, int startingNode) {  
    assert(graph != NULL);  
    assert(startingNode >= 0);  
  
    bool* visited = (bool*) malloc(graph->nodeCount * sizeof(bool));  
    for (int i = 0; i < graph->nodeCount; i++) {  
        visited[i] = false;  
    }  
  
    int* queue = (int*) malloc(graph->nodeCount * sizeof(int));  
    int front = 0, rear = 0;  
  
    visited[startingNode] = true;  
    queue[rear++] = startingNode;  
  
    while (front < rear) {  
        int currentNode = queue[front++];  
  
        struct bfs_node* temp = graph->list[currentNode].head;  
        while (temp != NULL) {  
            int adjNode = temp->data;  
            if (!visited[adjNode]) {  
                visited[adjNode] = true;  
                queue[rear++] = adjNode;  
            }  
            temp = temp->next;  
        }  
    }  
  
    free(visited);  
    free(queue);  
}
```

### Wrapping up
Nowadays, I don't have that much time to study specific algorithms, so it was really nice to be able to study BFS. The full implementation can be found in my [data structures and algorithms repository](https://github.com/iozsaygi/dsaa/blob/main/src/breadth_first_search.c).

Thank you for reading.