+++
title="Graph"
date=2021-01-18

[taxonomies]
categories = ["Data Structure"]
tags = ["graph", "BFS", "DFS"]
+++

A graph is a data structure that represents relationships using **nodes, also called vertices**, and **edges**.

```text
G = (V, E)

V = vertices, nodes
E = edges, relationships between vertices
```

A tree is also a kind of graph. More precisely, it can be viewed as a connected undirected graph with no cycles.

## When graphs are needed

Graphs are useful when the core of the data is the relationship between items.

- Social network: person = vertex, friendship = edge
- Map: city/intersection = vertex, road = edge
- Dependency graph: package/module = vertex, dependency = edge
- Web: page = vertex, hyperlink = edge
- Workflow/state machine: state = vertex, transition = edge

If arrays and lists focus on order, graphs focus on connection.

## Types of graphs

### Undirected Graph

The edges have no direction.

```text
A -- B
```

If `A` is connected to `B`, then `B` is also connected to `A`. This fits models such as Facebook friendships.

### Directed Graph

The edges have direction.

```text
A -> B
```

Just because `A` points to `B` does not mean `B` points to `A`. This fits models such as Twitter follows, package dependencies, and web links.

### Weighted Graph

The edges have values such as cost or distance.

```text
A --(7)-- B
```

Weighted graphs are used to represent road distance, network latency, transfer cost, similarity score, and similar values.

### Cyclic / Acyclic Graph

A cycle is a path where you start at a vertex, move along edges, and can return to the same vertex.

A directed graph with no cycles is called a DAG, or Directed Acyclic Graph. DAGs appear often in build dependencies, course prerequisites, and job scheduling problems.

### Connected Graph

In an undirected graph, if every vertex can reach every other vertex, it is a connected graph. If the graph has several disconnected parts, those parts are called connected components.

In a directed graph, when vertices can reach each other while respecting direction, they are called strongly connected.

## Graph representation

Representation is very important when solving graph algorithms. Let `V` be the number of vertices and `E` the number of edges.

### Adjacency Matrix

An adjacency matrix stores whether an edge exists, or the edge weight, in a two-dimensional array.

```text
matrix[u][v] = true
matrix[u][v] = weight
```

| Operation | Complexity |
|---|---:|
| Memory | O(V^2) |
| Edge existence check | O(1) |
| Iterate all neighbors of a vertex | O(V) |

Advantages:

- It is easy to implement.
- You can immediately check whether `u` and `v` are connected.
- It is suitable for dense graphs, meaning graphs with many edges.

Disadvantages:

- Sparse graphs waste a lot of memory.
- To find neighbors, you must scan the entire row.

### Adjacency List

An adjacency list stores a list of connected neighbors for each vertex.

```text
0: [1, 2]
1: [0, 3]
2: [0]
3: [1]
```

In Rust, it is usually represented like this.

```rust
let graph: Vec<Vec<usize>> = vec![
    vec![1, 2],
    vec![0, 3],
    vec![0],
    vec![1],
];
```

For a weighted graph, store the neighbor and weight together.

```rust
let graph: Vec<Vec<(usize, i32)>> = vec![
    vec![(1, 7), (2, 3)],
    vec![(0, 7), (3, 2)],
    vec![(0, 3)],
    vec![(1, 2)],
];
```

| Operation | Complexity |
|---|---:|
| Memory | O(V + E) |
| Edge existence check | O(deg(u)), average O(1) with a set |
| Iterate all neighbors of a vertex | O(deg(u)) |

Advantages:

- It is memory-efficient for sparse graphs.
- It works well for algorithms that iterate neighbors, such as BFS and DFS.

Disadvantages:

- It can be slower than a matrix for problems that frequently ask only whether a specific edge exists.

### Edge List

An edge list stores only the list of edges.

```rust
let edges = vec![
    (0, 1, 7),
    (0, 2, 3),
    (1, 3, 2),
];
```

It fits algorithms such as Kruskal MST, where edges are sorted and processed. But if you often need to find the neighbors of a specific vertex, it is inefficient because you must scan the whole edge list every time.

## Representation comparison

| Representation | Memory | Edge check | Neighbor traversal | Best fit |
|---|---:|---:|---:|---|
| Adjacency matrix | O(V^2) | O(1) | O(V) | Dense graph, fast edge lookup |
| Adjacency list | O(V + E) | O(deg) | O(deg) | Sparse graph, BFS/DFS |
| Edge list | O(E) | O(E) | O(E) | Edge-centered algorithms, Kruskal |

Most coding-test graph problems use sparse graphs, so an adjacency list is the default choice.

## BFS

BFS, or Breadth-First Search, visits vertices closest to the start first. It uses a queue.

In an unweighted graph, BFS can find the **shortest number of edges** from the starting point.

```rust
use std::collections::VecDeque;

fn bfs(graph: &[Vec<usize>], start: usize) -> Vec<Option<usize>> {
    let mut distance = vec![None; graph.len()];
    let mut queue = VecDeque::new();

    distance[start] = Some(0);
    queue.push_back(start);

    while let Some(current) = queue.pop_front() {
        let next_distance = distance[current].unwrap() + 1;

        for &next in &graph[current] {
            if distance[next].is_none() {
                distance[next] = Some(next_distance);
                queue.push_back(next);
            }
        }
    }

    distance
}
```

With an adjacency list, the complexity is `O(V + E)`.

Common use cases:

- Unweighted shortest path
- Finding connected components
- Bipartite graph check
- Level-order traversal
- Finding the minimum number of moves in a grid

## DFS

DFS, or Depth-First Search, follows one path all the way down, then backtracks. It can be implemented with recursion or with a stack.

```rust
fn dfs(graph: &[Vec<usize>], current: usize, visited: &mut [bool]) {
    visited[current] = true;

    for &next in &graph[current] {
        if !visited[next] {
            dfs(graph, next, visited);
        }
    }
}
```

With an adjacency list, the complexity is `O(V + E)`.

Common use cases:

- Cycle detection
- Finding connected components
- Topological sort
- Backtracking
- Tree/graph traversal
- Building block for strongly connected component algorithms

## Representative algorithms

For graph problems, the algorithm is usually chosen by asking, "What do we need to find?"

| Problem | Representative algorithm |
|---|---|
| Unweighted shortest path | BFS |
| Non-negative weighted shortest path | Dijkstra |
| Shortest path with negative edges | Bellman-Ford |
| All-pairs shortest path | Floyd-Warshall |
| Minimum spanning tree | Kruskal, Prim |
| Topological order | DFS or Kahn's algorithm |
| Strongly connected components | Kosaraju, Tarjan |
| Bipartite check | BFS/DFS coloring |
| Connectivity / cycle in undirected graph | DFS, Union-Find |

## Tree vs Graph

| Tree | Graph |
|---|---|
| A special form of graph | A more general relationship model |
| Connected + acyclic | May or may not be connected |
| If there are `n` nodes, there are `n - 1` edges | Edge count is much more flexible |
| Exactly one path exists between two nodes | There can be zero, one, or many paths |
| Usually has root/parent/child | May have no root |

Even if a problem looks like a tree problem, if the input is given as arbitrary edges instead of parent-child relationships, treat it as a graph and think about cycle handling and `visited` tracking.

## Common mistakes

- Adding an edge in only one direction for an undirected graph
- Marking `visited` when popping from the queue instead of when pushing, causing duplicate enqueues
- Using BFS on a weighted graph
- Using Dijkstra with negative-weight edges
- Mixing up cycle detection methods for directed and undirected graphs
- Ignoring stack overflow risk when recursive DFS runs on a deep graph

## Things to remember

- A graph is a data structure that represents relationships.
- Sparse graphs fit adjacency lists, and dense graphs fit adjacency matrices.
- BFS uses a queue, while DFS uses a stack or recursion.
- BFS is suitable for unweighted shortest path.
- For weighted shortest path, choose Dijkstra, Bellman-Ford, or another algorithm based on the weight conditions.
- A tree is a special form of graph.

## Ref

- Graph data structure: <https://en.wikipedia.org/wiki/Graph_(abstract_data_type)>
- Graph theory: <https://en.wikipedia.org/wiki/Graph_theory>
- Adjacency list: <https://en.wikipedia.org/wiki/Adjacency_list>
- Adjacency matrix: <https://en.wikipedia.org/wiki/Adjacency_matrix>
- CP-Algorithms - Breadth First Search: <https://cp-algorithms.com/graph/breadth-first-search.html>
- CP-Algorithms - Depth First Search: <https://cp-algorithms.com/graph/depth-first-search.html>
- CP-Algorithms - Dijkstra: <https://cp-algorithms.com/graph/dijkstra.html>
- CP-Algorithms - Minimum Spanning Tree: <https://cp-algorithms.com/graph/mst_kruskal.html>
