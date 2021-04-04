# homework3

Implementing Data Structures with Traits and Generics

## Clippy and Rustfmt

Reminder: please run `cargo clippy` and `cargo fmt` on all your code before submitting, and deal with all warnings (yellow text), not just errors (red text).

## Overview

For this assignment, we will implement a graph data structure in Rust.

No code will be given for the assignment. Additionally, don't use any crates
unless you check on Piazza first, other than `serde`.

The assignment is divided into three parts: (1) the data structure itself, (2) advanced functionality (including custom trait implementations related to the data structure), and (3) an application of your data structure to some target problem.

## Part 1: The Data Structure

Your graph data structure should be generic in two arguments: the vertex label type `V` and the edge label type `E`. You should allow directed edges (edges in each direction) and self-loop edges; however, you should prohibit having multiple edges between the same source and target vertex.

### Basic Methods

Your `Graph<V, E>` should support at least the following basic methods.
It should also support more advanced search functions; more on this under "Advanced methods".

- Creating a new empty graph;

- Adding a vertex with a given vertex label;

- Adding an edge between two existing vertices with a given edge label;

- *Ensuring* a vertex with a given label exists (or adding it if it does not exist);

- *Ensuring* an edge between two existing vertices (or adding it if it does not exist);

- Removing an existing edge;

- Removing a vertex (and all associated edges);

- Getting the out-degree or in-degree (number of outgoing edges or ingoing edges) of a vertex;

- A vertex iterator which iterates through all vertex labels in your graph: I recommend you use the `impl Iterator` syntax to help with this. Some syntax to get started: `fn iter_vertices(&self) -> impl Iterator<Item = &V> + '_`. The `impl Iterator` means you are hiding the return type, but you are asking the type checker to verify that whatever the return type is, it implements the Iterator trait. The `'_` lifetime placeholder is to make the borrow checker happy since the iterator object has to have read-only access to `&self` over its existence. You can also add explicit `'a'` lifetimes if needed for your use case.

- Two edge iterators: `iter_sources` and `iter_targets` which, given a `&V`, iterate through all source edges and target edges from that vertex in the graph. You will want the iterator to return `(&E, &V)`, i.e. both the edge label and the source/target vertex.

- A `merge_vertices` function which merges vertices `v1` and `v2` into just `v1`, and moves all edges from and to `v2` to be from and to `v1` (respectively).

- Existence checks: `has_vertex` (accepting one `&V` argument) and `has_edge` (accepting two `&V` arguments), returning `bool`.

- A `get_edge` function to get the edge label between two vertices, or `None` if there is no edge.

- An invariant check `fn assert_invariant()` which uses `debug_assert!` to (exhaustively) check any invariants you are assuming about your data structure.

### Implementation Details

To avoid worrying about difficult lifetime issues, I recommend that you use identifiers to represent vertices and edges.
To do this, define new types: `struct VertexIden(usize)` and `struct EdgeIden(usize)`.
Whenever a vertex or edge is added to your graph, first assign it a new label;
then use that label internally to represent the vertex or edge.
For example you could store the original labels in a `Vec<Option<V>>` (`Option` to allow vertex removal), but the actual graph
using HashMaps.
The type wrappers ensure that you can't accidentally try to use a vertex as an edge or
vice versa; you have to do `.0` to get the underlying `usize` value.

Make sure that most of your methods are efficient (`O(1)` insert and remove), but `merge_vertices` will necessarily be less efficient.
`HashMap` or a similar data structure is necessary for this.

- **Edit:** Your graph will need ways to convert between identifiers and the original objects: so you need a way to go from `V` to `VertexIden` and from `VertexIden` to `V`. Instead of a `Vec` as mentioned above, you could use `HashMap<VertexIden, V>` and `HashMap<&V, VertexIden>`. Note that the latter has a reference to `V` as the HashMap key: to get this to work, you will need your Graph to have a lifetime, like `Graph<'a, V, E>`, and then you can have `HashMap<&'a V, VertexIden>`. Alternatively, if you want, you can do the assignment requiring `Clone` and use `Clone` when a vertex is added; in this case you can get away with `HashMap<VertexIden, V>` and `HashMap<V, VertexIden>`. See the other **Edit** below.
Either way, I recommend starting out in your implementation by implementing the functions which go between identifiers and the original objects. So try to write your methods like
`get_vertex_iden(&self, &V) -> VertexIden`
and `get_vertex(&self, VertexIden) -> &V`.
If you can get these two functions working (and the same thing for edges), then that should solidify the main design, and the rest of the assignment should go more smoothly.

### Standard Library Traits

**Avoid unnecessary trait bounds.** Your `Graph` should not require any trait bounds on `V` and `E` by default. In particular, it should be usable even if `V` and `E` don't implement `Clone`.

- **Edit:** Some clarification on this: you can require other bounds, like `Eq` and `Hash`. Additionally, if you want a slightly easier task, I am allowing you to require `V: Clone` and `E: Clone` if you prefer, but make sure that `.clone()` is only used sparingly. That is, you should only need to clone a vertex or edge when it is added to the graph, and not anywhere else.

Please `#[derive(...)]` or implement `Clone`, `Debug`, `Display`, `Index`, and `IndexMut` for your graph. Although it should not require any trait bounds by default, your graph will require trait bounds in order to implement these traits: for example `Clone` won't be implemented unless `V` and `E` implement `Clone`.

For `Index` and `IndexMut`, it makes the most sense to accept a `(&V, &V)` index, and then return a reference to the edge between those vertices.

### Argument Ownership

**Avoid the use of clone and unnecessary owned arguments.** That means that, for example, if you have a function `fn add_vertex` it should take a `v: V` (to avoid cloning), but if you have `add_edge` it should take `v1: &V` and `v2: &V` (and `edge_label: E`): it needs ownership over the edge, but not ownership over the vertices.
On the other hand, your `ensure_vertex` function needs `v: V` since it needs to insert the vertex if it doesn't exist.
Or if you want to be fancy, `ensure_vertex` can take a function argument `f: impl Fn() -> V` that is not called unless needed, called with a closure like `move || v`.

## Part 2: Advanced Functionality

For this part, write the following advanced methods and custom traits.

### Serialization and Deserialization

Use `serde` to derive the Serialize and Deserialize
traits for your object.
Then implement two derived functions:

- `fn save_to_file(&self, filename: &str) -> Result<(), String>`

- `fn load_from_file(filename: &str) -> Result<Self, String>`

Write at least one unit test for these (see unit testing below).

### DFS and BFS: Utility Structs

We want our graph to support more interesting functionality, like using a DFS or BFS to check reachability between vertices.
For this, implement two utility structs, one for DFS and one for BFS.
Your structs will need to have, as one of the fields, a `next` function
that gives the next items from a current item that can be used during search.
For this, we can use function traits in Rust:
`next` will be a function implementing the `Fn` trait.
Here is a starting point:

```Rust
struct DFS<T: Eq + PartialEq + Hash, F: Fn(T) -> Vec<T>> {
    next: F,
    visited: HashSet<T>
    to_visit: Vec<T>,
}
```

Implement a `new` function to create a new `DFS`.
Then, your `DFS` function should implement the `Iterator` trait:

```Rust
impl<T: Eq + PartialEq + Hash, F: Fn(T) -> Vec<T>> Iterator for DFS<T, F> {
    ...
}
```

Do the same thing for a `BFS` struct.
Then, implement corresponding methods for your `Graph` structure,
using the DFS and BFS structs.
**Implement both vertex searches and edge searches, in both forwards and backwards directions.**
For example, the edge search could look something like this:

```Rust
impl<V, E> Graph<V, E> {
    fn edge_bfs_forward(&self, start_vertex: &V) -> impl Iterator<Item = &V> + 'a {
        ...
    }
}
```

Internally, it would use a `BFS` struct over `VertexIden`.

This may require playing around with function closure and lifetime compiler errors
if you get down the wrong path.
Don't give up and be patient! Definitely post to Piazza if you get stuck.

### Reachability via The Reachable Trait

We can use traits in Rust to abstract behavior in an implementation-independent
way.
Now that you have the DFS functionality, implement a trait which works for
a general graph-like collection:

```Rust
trait Reachable {
    type T;

    fn can_reach(start: &T, end: &T) -> bool;
    fn distance(start: &T, end: &T) -> Option<usize>;
}
```

Implement `Reachable` for your `Graph` struct. You can use vertex reachability
and ignore edge reachability for this part.
The `distance` should be the minimum distance from one vertex to the other.

Additionally, implement at least two derived methods for `Reachable`.
Some ideas:

- `fn can_reach_eachother(start: &T, end: &T)`

- `fn is_closer_than(start: &T, end: &T, dist: usize) -> bool`

### The Summary trait

Finally, implement a trait called `Summary`, similar to what we saw in class
(Lecture 6 part 2)
that allows summarizing an object in `n` lines or fewer.
I.e. the core method should be `fn summarize(&self, lines: usize) -> String`.
Implement `Summary` for `Graph<V, E>` assuming the trait bounds
`V, E: Summary`.

## Part 3: Using your Graph object

Think of a cool example application for your graph object!
Maybe you have a bunch of `People` objects, and you want to form
friendships between them, where the graph tracks if one person is a friend
of another.
Something more extreme would be to use your graph to implement a compiler
for a very basic language, where the graph is the control flow graph of the
language.
Or, if you like to go with something more mathematical, you could use your graph to
implement some basic graphs, like cycle graphs, path graphs, etc.

This can be a relatively short demonstration, you don't need to implement
a full-fledged set of features, just a small file with a proof-of-concept
and a few examples in the unit tests or main function.

## Comments and Requirements

### Error Handling

We will be less pedantic about error handling on this assignment than on the previous one.
Your graph insertions/deletions don't need to have a `Result` type; instead use `Option`
when a getter-type method could fail, and use your judgment depending on the method
when a setter-type can fail:
either use `assert!` in the code or fall back to some default functionality (e.g. don't
insert the object).

### Modules

Sort your code into modules! The general rule is one struct or trait per file.
Smaller structs (like `VertexIden` etc.) or related traits can go in the same file as
each other.
Remember to include all the modules in `lib.rs` or else they won't be compiled.

### Unit Tests

Write at least one unit test for every function or method that you implement.
Traits do not have unit tests, but there should be unit tests for the
types implementing that trait.

## Submission

This assignment is submitted via GitHub classroom. The deadline is Wednesday, April 7, 2021 at 11:59pm Eastern. (**Edit:** updated deadline.)
