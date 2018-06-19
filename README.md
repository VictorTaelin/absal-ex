### Problem description

The input is a graph where every node has exactly 3 edges coming out of 3 labelled ports (port A, port B, port C) and a single 32-bit label. If this graph has any edge connecting two nodes through ports A, then those nodes form an "active pair", and must be rewritten due to one of the two following rules:

![](rewrite_rules_a_sk.JPG)

Here, the triangles represent nodes, and the lone circles represent ports (i.e., parts of other nodes elsewhere). The first rule is used if the node labels are different. The second one if they are identical. That could cause different nodes to become active pairs. If that is the case, those must be rewritten too. That process goes on until there is no active pair left. 

### Example

![](full_example.JPG)

The graph to the left is the input. Notice there are two nodes connected through ports of label 0: those are active pairs. The graph to the right is the next step of the computation: both of those nodes were rewritten, the bottom ones using the first rule, and top ones using the second rule. Now there are 3 active pairs.

### Parallel implementation

That problem is remarkably parallel, because, at any point in time, thousands of active pairs can be rewritten independently. The naive algorithm, thus, is simple. First, the graph is represented as a buffer of 32-bit unsigned integers. Each quadruple of uints form a node. The first 3 values of the quadruple are the ports of that node, and they point to the ports of the corresponding node. The last value is the tag. So, for example, the following graph:

![](simple_example.JPG)

Is represented as:

```javascript
//            node_A                          node_B 
//            port_A  port_B  port_C  label   port_A  port_B  port_C    label   
var buffer = [     4,      2,      1,      0,       0,      6,      5,      1]
//               i=0     i=1     i=2     i=3      i=4     i=5     i=6     i=7
```

The graph has 2 nodes, so the buffer has 4 * 2 = 8 uints. The index 0 of the buffer represents the A port of the node A. Thus, `buffer[0] = 4`, because 4 is the position of the A port of node B. Similarly, `buffer[1] = 2` and `buffer[2] = 1`, because the port B and the port C of the node A are connected. The algorithm them just continuously rewrittes that buffer as such:

1. Find the initial list of active pairs;

2. Rewrite them in parallel;

3. Visit neighbor of rewritten nodes, looking for more active pairs;

4. If any is found, go to 2.

There is one problem, though. Notice the following graph:

![](problem.JPG)

Here, two threads are trying to rewrite two active pairs, `B-C` and `D-E`, in parallel. The problem is that thread 1 attempts to rewrite a port of the node D, which is currently being read by the thread 2! Or, in other words, while the rewrite rules are local, they can intersect. The following scenario can happen:

1. Thread 1 loads nodse B, C

2. Thread 2 noads nodes D, E

3. Thread 1 rewrites nodes B, C

4. Now, D points to A, not C

5. But kernel 2 loaded the old version of D

6. Thread 2 rewrites D, E with the information it has

7. A now points to node D, which was erased

8. F now points to node C, which was erased

9. Invalid state

To solve that problem, I perform the reduction in two steps. The first step, `redex()`, applies fully local rewrite rules to active pairs:

![](local_rewrites.PNG)

Notice that, here, P, Q, R and S (i.e., ports on different nodes) aren't affected. Instead, they keep pointing to the same location, but that location now points to where they should point. Also, the rewritten nodes aren't erased yet; instead, they are kept on memory as temporary nodes. The second steps, `visit()`, takes place after the first one (i.e., a global synchronization). It spawns one thread for each neighbor port of rewritten nodes. Each thread travels through the graph until it finds a port in a non-temporary node. Then, it rewrites the starting ports to point to the final one. It also marks the temporary node for garbage collection (i.e., allows that space to be allocated) and, if both ports are 0, then it activates the step 1 again for that active pair. Here is an example:

![](local_rewrites_ex.JPG)

Notice that, on the first step, `redex()`, two threads rewrite nodes `B-C` and `D-E` locally. Their effect areas don't intersect, and nodes `A` and `F` aren't affected, for example. The second step is, then, responsible for completing the rewrite, collecting garbage nodes and starting step 1 again if new active pairs are present. On the next step, `visit()`, 8 threads start walking through the graph starting from ports `Ab, Ac, Db, Dc, Cb, Cc, Fb, Fc` respectivelly (here, `Ab` represents port `B` of node `A`). The thread starting from `Ab`, for example, walks through this path: `Ab -> Bc -> Db -> Fc`. When it arrives at `Fc` (a non-temporary node), it connects `Ab` to it. It also marks nodes `B` and `D` for garbage collection. Finally, since `Ab <-> Fc` isn't an active pair, it stops. If it was `Aa <-> Fa` instead, it would start a `redex()` thread for it.

This algorithm is implemented on the [branch `parallel_test_3` of the absal-rs repository](https://github.com/moon-project/absal-rs/tree/parallel-test-3). The OpenCl code is on `main.rs`. Despite being a GPU newbie and probably doing a lot of innefficient things (like using `atomic_inc()` to implement global stacks, and reading data from the GPU between each kernel call), I managed to achive about 16m rewrites/s on Intel Iris Graphics 550, exactly the same performance I got on my sequential Rust implementation in a 3.3 GHZ Intel Core i7. My hope is that an expert in GPUs would be able to improve that number considerably.

### Testing

The purpose of this algorithm is to reduce lambda calculus terms optimally. It can, thus, be seen as the runtime of a programming language. It can be tested by converting lambda-terms to buffers on the format I described above, reducing it with the implementation, and then translating those buffers back to lambda-terms. If the output corresponds to the normal form of the lambda-term, then the test passes. There are, thus, literally infinite programs with infinitely different characteristics you could use to test it. I have Rust and JavaScript code capable of doing that, and a bunch of examples covering different kinds of programs if needed. The program used on the `parallel-test-3` branch of absal-rs (line 197 of main.rs) is probably good enough to test the performance, though. Another option is to just build some random graphs directly and test against a simpler, sequential reducer. Since the algorithm is pretty straightforward, such a thing could easily be implemented in any language you prefer.










