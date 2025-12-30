---
layout: post
title: Building Vec2GC using GenAI 
---
---# Leveraging GenAI to Write Custom Code for Graph-Based Clustering

Over a recent long weekend, I revisited a partially completed project: implementing the **Vec2GC** algorithm for graph-based clustering. I had originally worked on this some time ago and wanted to reimplement it from scratch—this time with a cleaner, more maintainable design than the version written for the original paper.

Earlier in 2025, I had started building an initial version using the Claude web interface but never found the time to complete it. During this break, I decided to pick it up again and push it to completion. Over the span of four days, I iterated on the code with the help of GenAI and eventually got it working on a corpus of roughly **500k documents**—all on a **16 GB MacBook Pro**.

The initial implementation was functional but suboptimal, with several missing features and scalability issues. Through multiple iterations, GenAI helped refine the design, optimize performance, and fix subtle bugs, resulting in a significantly more robust and scalable system.

---

## Problem Overview: Vec2GC

The **Vec2GC** algorithm performs clustering on a document corpus using embeddings. At a high level, it works as follows:

1. Construct a graph where each node represents a document embedding.
   - An edge is created between two nodes if their cosine similarity exceeds a predefined threshold.
2. Apply a community detection algorithm to identify clusters within the graph.
3. Recursively apply community detection to each cluster until stopping criteria are met, resulting in a hierarchical clustering structure.

This recursive, graph-based approach enables the discovery of fine-grained semantic structure in large embedding spaces—but it also introduces significant computational and memory challenges.

---

## GenAI-Driven Code Improvements

GenAI played a key role in iteratively improving the implementation. While the first version it generated was intentionally simple, successive prompts and refinements led to a much more scalable and production-ready design.

### Chunked Processing

The first major challenge was **scale**. Running similarity computations for hundreds of thousands of embeddings on a local machine with limited memory quickly exposed bottlenecks.

The initial optimization replaced a single, monolithic similarity matrix with **chunked similarity computation**. Instead of computing all pairwise similarities at once, GenAI proposed breaking the computation into smaller chunks that could be processed independently and, where possible, in parallel.

This approach addressed immediate memory constraints by avoiding the need to materialize the full similarity matrix in memory. It also opened the door to parallel execution across CPU cores, reducing overall computation time while preserving clustering quality.

However, while chunking helped, it still required storing large intermediate similarity blocks in memory, which became the next bottleneck.

---

### Memory Optimization

The next major design change eliminated the **full n×n similarity matrix** altogether.

Rather than computing and storing all similarities upfront, the updated implementation computes similarities **on the fly during graph construction**. This reduces memory complexity from $O(n^2)$ to $O(n × d)$, where $d$ is the embedding dimension. 

Key aspects of this redesign included:

- Streaming similarity computation instead of full matrix materialization  
- Batched dot-product calculations for normalized embeddings  
- A configurable `batch_size` parameter to balance memory usage and performance  
- One-time normalization of embeddings to avoid redundant computation

This change made it possible to process datasets that were previously infeasible on a local machine, while producing identical clustering results.

#### Sequential Subgraph Processing

Once similarity computation was optimized, the next memory bottleneck emerged during **recursive clustering**.

In the original design, when a graph was split into multiple communities, all resulting subgraphs were created and kept in memory simultaneously. This caused peak memory usage to grow rapidly with the branching factor of the hierarchy.

The improved design processes communities **sequentially**:
1. Create a subgraph for a single community  
2. Perform recursive clustering  
3. Explicitly delete the subgraph and trigger garbage collection  
4. Move on to the next community  

This ensures that only one subgraph exists in memory at any given time per recursion level. In practice, this reduced peak memory usage by **60–80%**, depending on the structure of the hierarchy. This optimization was critical for running deep recursive clustering on a machine with limited RAM.

---

### Critical Bug Fixes

In addition to performance optimizations, GenAI helped identify and fix several bugs—one of which was particularly subtle and critical.

The issue involved **node ID mapping during recursive subgraph creation**. The algorithm was incorrectly mixing coordinate systems between the current subgraph’s node IDs and the original graph’s node IDs. While this did not trigger any runtime errors or logs, it resulted in nonsensical (“garbage”) clusters.

After observing anomalous clustering results, I suspected an issue with node ID handling. A simple prompt—_“check your node ID mapping”_—was enough for GenAI to trace the logic, identify the inconsistency, and fix the bug. The corrected implementation now consistently maintains mappings between local subgraph node IDs and original graph node IDs throughout recursion.

This was a good reminder that silent correctness bugs can be far more dangerous than obvious crashes—and that GenAI can be an effective reasoning partner in diagnosing them.

---

## Conclusion

Building this system with GenAI was both productive and genuinely enjoyable. Rather than acting as a simple code generator, GenAI increasingly felt like a **peer**—someone to discuss design trade-offs with, debate optimization strategies, and review complex logic.

This experience was notably different from earlier interactions, where hallucinations and subtle bugs were more common. With the latest Claude 4.5 release, I observed a clear shift in reliability and depth, particularly for:
- Design-level discussions  
- Rapid prototyping  
- Code review and bug fixing  

At this point, it would be hard to justify *not* using GenAI as part of the software development workflow. I’m looking forward to tackling more weekend projects with GenAI as a collaborative coding partner in the near future.
