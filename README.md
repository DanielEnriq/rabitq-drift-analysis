# RaBitQ Drift Analysis

This repository studies how a deployed **RaBitQ-style vector search index** degrades when the data distribution changes after the index is built.

RaBitQ is a randomized quantization method for approximate nearest neighbor search. In a static setting, vectors are normalized relative to a centroid, randomly rotated, compressed into sign codes, and used to estimate distances or inner products efficiently. This makes RaBitQ attractive for large-scale vector databases, where storing and searching full high-dimensional embeddings can be expensive.

But real vector search systems are rarely static. New embeddings are inserted over time. Old embeddings may be deleted or replaced. As the database changes, the empirical centroid of the stored vectors can drift away from the centroid used when the index was originally built. This creates a mismatch between the geometry used to construct the stored codes and the geometry the system should use at query time.

This project asks:

> What exactly breaks when a RaBitQ index is reused under centroid drift, and can any part of the index be repaired cheaply?

## Motivation

Vector search is now a core component of modern machine learning systems, including:

- retrieval-augmented generation systems,
- semantic search engines,
- recommendation systems,
- embedding databases,
- long-term memory systems for agents,
- multimodal search over images, text, audio, and video.

These systems often rely on approximate nearest neighbor search to retrieve relevant vectors from large embedding collections. To reduce memory and latency, practical systems compress vectors using quantization methods.

However, many theoretical guarantees for quantized vector search are stated for a fixed dataset and a fixed normalization geometry. In deployment, the dataset changes. Streaming inserts can shift centroids, alter residual directions, and cause stored compressed codes to become stale.

The central issue is not only that the database evolves. The issue is that a static quantization guarantee may be applied outside the geometric frame in which it was originally valid.

## Main Idea

The project separates centroid drift into two deployment policies:

### 1. Full-Freeze Policy

The system freezes both:

- the stored data codes,
- the routing centroids used by the index.

This is the cheapest deployment policy because no part of the index is updated after construction.

Under this policy, within-cell RaBitQ distance estimation remains calibrated in the old frame, but retrieval can fail because drift changes the correct cell assignment and query routing geometry. The main failure mode is therefore not local estimation error, but **inter-cell routing drift**.

### 2. Centroid-Tracking Policy

The system keeps centroids current online, but leaves stored sign codes stale until an explicit refresh.

This avoids accumulating routing drift in the same way, but introduces a different problem: the query is interpreted in the current centroid frame while the stored code was constructed in the old centroid frame. This creates a deterministic representation-drift bias.

The key observation is that this within-cell stale-code problem admits a partial repair. Under a drift certificate, only sign-code coordinates whose old rotated magnitude is small enough can possibly have flipped. Refreshing exactly those coordinates recovers the current sign code without rewriting every coordinate.

## Contributions

This repository develops and studies:

1. **A drift-separation view of RaBitQ under streaming inserts**  
   Centroid drift affects distance estimation, representation staleness, and routing/assignment through distinct channels.

2. **A full-freeze analysis**  
   Under frozen routing and frozen codes, the deployed estimator remains internally consistent in the build-time frame, while retrieval degradation enters through boundary-crossing and routing drift.

3. **A centroid-tracking analysis**  
   Under current centroids and stale stored codes, the estimator decomposes into a static RaBitQ error term plus a deterministic representation-drift bias.

4. **Certified partial refresh**  
   A deterministic certificate identifies which sign-code coordinates may have changed under drift. Coordinates outside this set are provably stable and do not need to be recomputed.

5. **Repair-cost asymmetry**  
   The two deployment policies route drift into different channels. Full-freeze accumulates inter-cell errors that are hard to repair exactly without inspecting many points. Centroid-tracking routes drift into a within-cell code-staleness problem that admits cheap exact repair.

## Why This Matters

Large vector databases are increasingly used as dynamic memory systems. In such systems, embeddings are not inserted once and forgotten. They arrive over time, reflect changing data distributions, and may be searched under shifting query workloads.

If compressed indexes are rebuilt too often, deployment becomes expensive. If they are never refreshed, search quality may degrade. Understanding which parts of the index can be safely left stale, which parts need updating, and which errors admit cheap certificates is important for building efficient dynamic vector search systems.

This project is a step toward a more precise theory of **freshness in compressed approximate nearest neighbor search**.

## Repository Structure

```text
rabitq-drift-analysis/
├── paper/          # Research writeup / PDF
├── src/            # Simulation and analysis code
├── notebooks/      # Exploratory experiments and visualizations
├── figures/        # Generated figures and diagrams
├── results/        # Experimental outputs
└── README.md