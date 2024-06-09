# CIC - Cloud infrastructure and computing LE4 mini challenge Report
Didnt want to write the report in word, so here it is in markdown as the repo README.

FYI: The notebook itself is also commented and this report is just a summary of the notebook.

## Problem Statement
In our SAN project, we noticed that proximity presitge was a metric that was quite computationally expensive. It has a time complexity of O(V^2 + VE) where V is the number of nodes and E is the number of edges, Ontop of that it needs a Breadth first search O(E + V) to calculate the shortest paths between a node and every other node. Even at a downsampled graph of only 20000 nodes (~170000 in our original graph) it took half an hour to compute. Scaled up to our whole graph it would take days to compute on a single process which was not feasible for us.

P.S. We did know about the `cugraph` project to add a cuda backend to networkx, but we had issues with the instalation (I have issues with my WSL and windows install, and my project partner, not for cic, doesnt have a machine with a cuda compatible GPU) therefore we subsampled the graph to make it more manageable. Which also gave me the opportunity to try and parallelize the computation for CIC.

## Parrallelization Implementation and Optimization
To parallelize the computation of proximity prestige, I used `dask` with a local cluster to run batches of nodes. I split the nodes into batches matching the number of cores on my machine and ran the computation on each batch in parallel. 

My plan was to parallelize both the proximity prestige calculation and the shortest path calculation. But for some reason I just couldnt get the proximity prestige function to run on the dask cluster. So the results will just cover the shortest path calculation.

## Results
Below are results of doubling the number of workers on the dask cluster and the time taken to compute the proximity prestige for a 10000 node graph.

```yaml
================================================================================
Batch size: 10000
========================================
Calculated shortest path lengths for 10000 nodes in 87.77 seconds.
Calculated Proximity Prestige for 10000 nodes in 5.90 seconds. 
================================================================================
Batch size: 5000
========================================
Calculated shortest path lengths for 10000 nodes in 53.33 seconds.
Calculated Proximity Prestige for 10000 nodes in 6.11 seconds. 
================================================================================
Batch size: 2500
========================================
Calculated shortest path lengths for 10000 nodes in 35.83 seconds.
Calculated Proximity Prestige for 10000 nodes in 6.44 seconds. 
================================================================================
Batch size: 1250
========================================
Calculated shortest path lengths for 10000 nodes in 30.36 seconds.
Calculated Proximity Prestige for 10000 nodes in 7.21 seconds.
```
As the number of worker processes increases, there is a clear reduction in the compute time. Although the scaling is not linear, there is still a significant reduction in compute time. I was able to reduce the compute threefold by increasing the number of workers from 1 to 8. With an average 43% reduction in compute time for each doubling of the number of workers.

### More results:
All of these were done on a cluster with 8 workers and 2 threads per worker

```yaml
================================================================================
Batch size: 1250
========================================
Calculated shortest path lengths for 10000 nodes in 132.69 seconds.
Calculated Proximity Prestige for 10000 nodes in 7.53 seconds.
========================================
Calculated shortest path lengths for 10000 nodes in 29.02 seconds.
Calculated Proximity Prestige for 10000 nodes in 7.38 seconds. 
================================================================================
Batch size: 2500
========================================
Calculated shortest path lengths for 20000 nodes in 1803.55 seconds.
Calculated Proximity Prestige for 20000 nodes in 32.11 seconds.
========================================
Calculated shortest path lengths for 20000 nodes in 299.53 seconds.
Calculated Proximity Prestige for 20000 nodes in 31.31 seconds.
================================================================================
Batch size: 3125 (Didnt run this without dask so no comparison)
========================================
Calculated shortest path lengths for 25000 nodes in 786.07 seconds.
Calculated Proximity Prestige for 25000 nodes in 155.03 seconds.
```

## Conclusion and shortfalls
To conclude, the parallelization of the shortest path calculation was successful and showed a significant reduction in compute time as the number of workers increased. However, I wasnt able to parallelize the proximity prestige calculation which also has a significant time complexity and not great scaling.

Some of my shortfalls and mistakes throughout the project were:
- Not using the correct dask functions and data types (I tried to use dask bags instead of delayed functions and futures).
- First only parallelizing the proximity prestige calculation (which is the least computationally expensive part of the code and it didnt even work) instead of the BFS.
- Making every single node a future instead of using batches. Lead to many many memory errors, crashes and 10x longer compute times on very small graphs.
- Not setting up the local cluster correctly if at all.
- Trying to use dask delays on the default networkx implementation of calculating the shortest paths which didnt work at all, and is not ideal as the networkx implementation uses generators which afaik dont work well in dask delayed objects.
