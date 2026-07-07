+++
title="Graph 자료구조"
date=2021-01-18

[taxonomies]
categories = ["Data Structure"]
tags = ["graph", "BFS", "DFS"]
+++

Graph는 **node(vertex)** 와 **edge** 로 관계를 표현하는 자료구조다.

```text
G = (V, E)

V = vertices, nodes
E = edges, relationships between vertices
```

Tree도 graph의 한 종류다. 정확히는 cycle이 없는 connected undirected graph를 tree라고
볼 수 있다.

## Graph가 필요한 순간

Graph는 "데이터 사이의 관계"가 핵심일 때 쓴다.

- Social network: 사람 = vertex, 친구 관계 = edge
- Map: 도시/교차로 = vertex, 도로 = edge
- Dependency graph: package/module = vertex, dependency = edge
- Web: page = vertex, hyperlink = edge
- Workflow/state machine: state = vertex, transition = edge

Array나 list가 "순서" 중심이라면, graph는 "연결" 중심이다.

## Graph 종류

### Undirected Graph

Edge에 방향이 없다.

```text
A -- B
```

`A`가 `B`와 연결되어 있으면 `B`도 `A`와 연결되어 있다. Facebook 친구 관계 같은 모델에
어울린다.

### Directed Graph

Edge에 방향이 있다.

```text
A -> B
```

`A`가 `B`를 가리킨다고 해서 `B`가 `A`를 가리키는 것은 아니다. Twitter follow, package
dependency, web link 같은 모델에 어울린다.

### Weighted Graph

Edge에 비용이나 거리 같은 값이 붙는다.

```text
A --(7)-- B
```

도로 거리, 네트워크 latency, 환승 비용, similarity score 등을 표현할 때 쓴다.

### Cyclic / Acyclic Graph

Cycle은 어떤 vertex에서 시작해서 edge를 따라 이동했을 때 다시 자기 자신으로 돌아올 수 있는
경로다.

Cycle이 없는 directed graph를 DAG(Directed Acyclic Graph)라고 한다. Build dependency,
course prerequisite, job scheduling 같은 문제에서 많이 나온다.

### Connected Graph

Undirected graph에서 모든 vertex가 서로 도달 가능하면 connected graph다. 연결되지 않은
부분이 여러 개 있으면 connected component가 여러 개 있다고 말한다.

Directed graph에서는 방향까지 고려해서 서로 도달 가능한 경우 strongly connected라고 한다.

## Graph 표현 방법

Graph 알고리즘을 풀 때는 표현 방법이 매우 중요하다. Vertex 수를 `V`, edge 수를 `E`라고
하자.

### Adjacency Matrix

2차원 배열로 edge 존재 여부나 weight를 저장한다.

```text
matrix[u][v] = true
matrix[u][v] = weight
```

| 작업 | Complexity |
|---|---:|
| Memory | O(V^2) |
| Edge 존재 확인 | O(1) |
| 어떤 vertex의 모든 neighbor 순회 | O(V) |

장점:

- 구현이 쉽다.
- `u`와 `v`가 연결되어 있는지 바로 확인할 수 있다.
- Dense graph(edge가 매우 많은 graph)에 적합하다.

단점:

- Sparse graph에서는 memory 낭비가 심하다.
- Neighbor를 찾으려면 한 row 전체를 훑어야 한다.

### Adjacency List

각 vertex마다 연결된 neighbor 목록을 저장한다.

```text
0: [1, 2]
1: [0, 3]
2: [0]
3: [1]
```

Rust로는 보통 이렇게 표현한다.

```rust
let graph: Vec<Vec<usize>> = vec![
    vec![1, 2],
    vec![0, 3],
    vec![0],
    vec![1],
];
```

Weighted graph라면 neighbor와 weight를 같이 저장한다.

```rust
let graph: Vec<Vec<(usize, i32)>> = vec![
    vec![(1, 7), (2, 3)],
    vec![(0, 7), (3, 2)],
    vec![(0, 3)],
    vec![(1, 2)],
];
```

| 작업 | Complexity |
|---|---:|
| Memory | O(V + E) |
| Edge 존재 확인 | O(deg(u)), set을 쓰면 평균 O(1) |
| 어떤 vertex의 모든 neighbor 순회 | O(deg(u)) |

장점:

- Sparse graph에 memory 효율적이다.
- BFS/DFS처럼 neighbor를 순회하는 알고리즘에 좋다.

단점:

- 특정 edge 존재 여부만 자주 묻는 문제에서는 matrix보다 느릴 수 있다.

### Edge List

Edge 목록만 저장한다.

```rust
let edges = vec![
    (0, 1, 7),
    (0, 2, 3),
    (1, 3, 2),
];
```

Kruskal MST처럼 edge를 정렬해서 처리하는 알고리즘에 잘 맞는다. 하지만 특정 vertex의
neighbor를 자주 찾아야 한다면 매번 edge 전체를 훑어야 해서 비효율적이다.

## 표현 방법 비교

| 표현 | Memory | Edge check | Neighbor traversal | 잘 맞는 경우 |
|---|---:|---:|---:|---|
| Adjacency matrix | O(V^2) | O(1) | O(V) | Dense graph, 빠른 edge lookup |
| Adjacency list | O(V + E) | O(deg) | O(deg) | Sparse graph, BFS/DFS |
| Edge list | O(E) | O(E) | O(E) | Edge 중심 알고리즘, Kruskal |

대부분의 코딩 테스트 graph 문제는 sparse graph가 많아서 adjacency list가 기본 선택이다.

## BFS

BFS(Breadth-First Search)는 시작점에서 가까운 vertex부터 방문한다. Queue를 사용한다.

Unweighted graph에서 BFS는 시작점으로부터의 **최단 edge 수**를 구할 수 있다.

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

시간 복잡도는 adjacency list 기준 `O(V + E)`다.

자주 쓰이는 문제:

- Unweighted shortest path
- Connected component 찾기
- Bipartite graph 검사
- Level-order traversal
- Grid에서 최소 이동 횟수 구하기

## DFS

DFS(Depth-First Search)는 한 경로를 끝까지 들어갔다가 되돌아온다. Recursion 또는 stack으로
구현한다.

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

시간 복잡도는 adjacency list 기준 `O(V + E)`다.

자주 쓰이는 문제:

- Cycle detection
- Connected component 찾기
- Topological sort
- Backtracking
- Tree/graph traversal
- Strongly connected component 알고리즘의 기본 재료

## 대표 알고리즘

Graph 문제는 보통 "무엇을 구해야 하는가"로 알고리즘을 고른다.

| 문제 | 대표 알고리즘 |
|---|---|
| Unweighted shortest path | BFS |
| Non-negative weighted shortest path | Dijkstra |
| Negative edge가 있는 shortest path | Bellman-Ford |
| All-pairs shortest path | Floyd-Warshall |
| Minimum spanning tree | Kruskal, Prim |
| Topological order | DFS 또는 Kahn's algorithm |
| Strongly connected components | Kosaraju, Tarjan |
| Bipartite check | BFS/DFS coloring |
| Connectivity / cycle in undirected graph | DFS, Union-Find |

## Tree vs Graph

| Tree | Graph |
|---|---|
| Graph의 특수한 형태 | 더 일반적인 관계 모델 |
| Connected + acyclic | connected일 수도 있고 아닐 수도 있음 |
| node `n`개면 edge `n - 1`개 | edge 개수 제한이 훨씬 자유로움 |
| 두 node 사이 path가 정확히 하나 | path가 0개, 1개, 여러 개일 수 있음 |
| 보통 root/parent/child가 있음 | root가 없을 수도 있음 |

Tree 문제처럼 보여도 입력이 "parent-child" 관계가 아니라 임의 edge 목록으로 주어진다면
graph로 보고 cycle, visited 처리를 생각해야 한다.

## 흔한 실수

- Undirected graph에서 edge를 한 방향만 추가함
- `visited` 처리를 queue에 넣을 때 하지 않고 pop할 때 해서 중복 enqueue가 많아짐
- Weighted graph에 BFS를 사용함
- Dijkstra에 negative weight edge를 사용함
- Directed graph와 undirected graph의 cycle detection 방식을 섞음
- Recursion DFS에서 graph가 깊을 때 stack overflow 가능성을 무시함

## 기억할 점

- Graph는 관계를 표현하는 자료구조다.
- Sparse graph는 adjacency list, dense graph는 adjacency matrix가 잘 맞는다.
- BFS는 queue, DFS는 stack 또는 recursion을 사용한다.
- BFS는 unweighted shortest path에 적합하다.
- Weighted shortest path는 weight 조건에 따라 Dijkstra, Bellman-Ford 등을 골라야 한다.
- Tree는 graph의 특수한 형태다.

## 참고

- Graph data structure: <https://en.wikipedia.org/wiki/Graph_(abstract_data_type)>
- Graph theory: <https://en.wikipedia.org/wiki/Graph_theory>
- Adjacency list: <https://en.wikipedia.org/wiki/Adjacency_list>
- Adjacency matrix: <https://en.wikipedia.org/wiki/Adjacency_matrix>
- CP-Algorithms - Breadth First Search: <https://cp-algorithms.com/graph/breadth-first-search.html>
- CP-Algorithms - Depth First Search: <https://cp-algorithms.com/graph/depth-first-search.html>
- CP-Algorithms - Dijkstra: <https://cp-algorithms.com/graph/dijkstra.html>
- CP-Algorithms - Minimum Spanning Tree: <https://cp-algorithms.com/graph/mst_kruskal.html>
