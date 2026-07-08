+++
title="Tree"
date=2023-05-05

[taxonomies]
categories = ["Data Structure"]
tags = ["tree", "binary-tree", "DFS", "BFS"]
+++

A tree is a hierarchical data structure made of nodes and edges. From a graph perspective, it is a **connected acyclic graph**: every node is connected, and there are no cycles.

![tree](/images/DS/Tree/Treedatastructure.png)
source: <https://www.geeksforgeeks.org/introduction-to-tree-data-structure-and-algorithm-tutorials/>

## Core properties

The three most important properties of a tree are:

1. A tree with `n` nodes has `n - 1` edges.
2. Between any two nodes, exactly one path exists.
3. Adding one edge creates a cycle, and removing one edge disconnects the tree.

In programming, the word tree usually means a **rooted tree**, which has a root. The root defines parent-child relationships, and depth increases as nodes get farther from the root.

## Terminology

| Term | Meaning |
|---|---|
| Root | The starting node with no parent |
| Parent | The node directly above another node |
| Child | The node directly below another node |
| Sibling | A node with the same parent |
| Leaf | A node with no children |
| Internal node | A node that is not a leaf |
| Degree | Number of children |
| Depth | Number of edges from the root to the node |
| Height | Number of edges from the node to the farthest leaf |
| Subtree | A partial tree rooted at a specific node |

For example, the root's depth is `0`, and a leaf's height is `0`. Some problems define height by the number of nodes instead of the number of edges, so in interviews or algorithm problems, it is best to confirm the definition first.

## Types of trees

### Binary Tree

A binary tree is a tree where each node has at most two children. The children are usually called `left` and `right`.

### Binary Search Tree

A binary search tree is a binary tree with an ordering rule.

```text
all values in left subtree < root value < all values in right subtree
```

Because of this rule, search, insert, and delete can be `O(log n)` on average. But if the tree becomes skewed to one side, its height becomes `n`, and those operations can degrade to `O(n)`.

### Self-balancing Search Tree

A self-balancing search tree adjusts itself after insert/delete operations so its height does not grow too large.

- AVL tree
- Red-black tree
- B-tree

Important note: **a B-tree is not a binary tree.** It is a multi-way balanced search tree where one node can have multiple keys and multiple children. It commonly appears in database indexes and filesystems.

### Full / Complete / Perfect Binary Tree

These three terms look similar, so they are easy to confuse.

| Type | Condition |
|---|---|
| Full binary tree | Every node has either 0 or 2 children |
| Complete binary tree | Every level except the last is full, and the last level is filled from the left |
| Perfect binary tree | Every internal node has 2 children, and all leaves are at the same depth |

A perfect binary tree is always both full and complete. But a full tree is not necessarily complete, and a complete tree is not necessarily full.

## Traversal

Traversal is a method for visiting every node in a tree exactly once. If a tree has `n` nodes, every traversal is basically `O(n)`.

![Tree](/images/DS/Tree/full_binary_tree.png)

### DFS: Depth-First Search

DFS goes all the way down in one direction, then comes back. It can be implemented with the recursive call stack or with an explicit stack.

In a binary tree, DFS traversals are named by when the root is visited.

| Traversal | Order | Example |
|---|---|---|
| Preorder | Root -> Left -> Right | A-B-D-E-H-I-C-F-G |
| Inorder | Left -> Root -> Right | D-B-H-E-I-A-F-C-G |
| Postorder | Left -> Right -> Root | D-H-I-E-B-F-G-C-A |

The way to remember this is simple. Left always comes before Right. If Root comes first, it is preorder. If Root is in the middle, it is inorder. If Root comes last, it is postorder.

Inorder traversal is especially important for binary search trees. Traversing a BST inorder produces values in sorted order.

```rust
fn inorder(node: Option<&Box<TreeNode>>, output: &mut Vec<i32>) {
    if let Some(node) = node {
        inorder(node.left.as_ref(), output);
        output.push(node.val);
        inorder(node.right.as_ref(), output);
    }
}
```

The extra space for DFS is `O(h)` from the recursive call stack, where `h` is the tree height. For a balanced tree it is `O(log n)`, and for a tree skewed to one side it is `O(n)`.

### BFS: Breadth-First Search

BFS is also called level-order traversal. It starts from the root, visits nodes at the same depth first, then moves down to the next level.

![Tree](/images/DS/Tree/binary_tree_level_order.png)

BFS is implemented with a queue. It appears often in problems involving level-by-level tree processing, shortest edge counts, and serialization.

```rust
use std::collections::VecDeque;

fn level_order(root: Option<&Box<TreeNode>>) -> Vec<i32> {
    let mut result = Vec::new();
    let mut queue = VecDeque::new();

    if let Some(root) = root {
        queue.push_back(root);
    }

    while let Some(node) = queue.pop_front() {
        result.push(node.val);

        if let Some(left) = node.left.as_ref() {
            queue.push_back(left);
        }

        if let Some(right) = node.right.as_ref() {
            queue.push_back(right);
        }
    }

    result
}
```

The extra space for BFS is proportional to the maximum number of nodes stored in the queue. If the tree's maximum width is `w`, the space is `O(w)`. In a complete binary tree, the last level contains many nodes, so the worst case can be `O(n)`.

## Representation

### Linked node

This is the most common representation. Each node points to its children.

```rust
#[derive(Debug)]
pub struct TreeNode {
    pub val: i32,
    pub left: Option<Box<TreeNode>>,
    pub right: Option<Box<TreeNode>>,
}

impl TreeNode {
    pub fn new(val: i32) -> Self {
        Self {
            val,
            left: None,
            right: None,
        }
    }
}
```

In Rust tree problems on LeetCode, you often see `Option<Rc<RefCell<TreeNode>>>` because of shared ownership and interior mutability. But when explaining the tree concept itself, `Box<TreeNode>` is simpler.

### Array

Trees with almost no empty spaces, such as complete binary trees, are easy to represent with an array.

If the root index is `0`:

```text
left child  = 2 * i + 1
right child = 2 * i + 2
parent      = (i - 1) / 2
```

Heaps use this representation.

```rust
struct ArrayTree {
    nodes: Vec<Option<i32>>,
}

impl ArrayTree {
    fn left_index(parent: usize) -> usize {
        2 * parent + 1
    }

    fn right_index(parent: usize) -> usize {
        2 * parent + 2
    }
}
```

The array representation has fast index calculation and good cache locality. But for sparse trees, many empty slots are needed, so memory is wasted.

## Complexity

| Operation | General binary tree | Balanced BST | Skewed BST |
|---|---:|---:|---:|
| Search | O(n) | O(log n) | O(n) |
| Insert | O(n), or O(1) if the position is known | O(log n) | O(n) |
| Delete | O(n) | O(log n) | O(n) |
| Traversal | O(n) | O(n) | O(n) |

The important thing in tree problems is to first distinguish whether it is a binary tree, a BST, or a balanced tree. If you mix those up, it is easy to get the time complexity wrong.

## Where trees are used

- Filesystem directory structures
- HTML/XML DOM
- Compiler AST, or Abstract Syntax Tree
- Database indexes such as B-tree and B+tree
- Priority queue, or heap
- Prefix search, or trie
- Range query structures such as segment tree and Fenwick tree

## Things to remember

- A tree is a connected graph with no cycles.
- In a rooted tree, parent-child relationships exist.
- A binary tree and a binary search tree are different.
- A balanced BST keeps height low so search/insert/delete stay close to `O(log n)`.
- BFS uses a queue, while DFS uses a stack or recursion.
- Array tree representation fits complete binary trees and heaps, but is inefficient for sparse trees.

## Ref

- Tree data structure: <https://en.wikipedia.org/wiki/Tree_(data_structure)>
- Binary tree: <https://en.wikipedia.org/wiki/Binary_tree>
- Binary search tree: <https://en.wikipedia.org/wiki/Binary_search_tree>
- Self-balancing binary search tree: <https://en.wikipedia.org/wiki/Self-balancing_binary_search_tree>
- B-tree: <https://en.wikipedia.org/wiki/B-tree>
- Binary heap: <https://en.wikipedia.org/wiki/Binary_heap>
- CP-Algorithms - Segment Tree: <https://cp-algorithms.com/data_structures/segment_tree.html>
- GeeksforGeeks - Binary tree array implementation: <https://www.geeksforgeeks.org/binary-tree-array-implementation/>
