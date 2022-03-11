---
layout: post
title:  Data Mining Course
description: Summary of the course on data mining replete with links etc.
img:
date: 2020-06-13  +1900
---

Pursuing a data mining course currently. While some topics lean towards algorithms, some other topics fall into distributed computing etc. Overall, there is a large collection of interesting topics that the Prof. has collected over the semester and I would like to store them unto posterity.

Here's to data mining:

At a high level, the course looks into the streaming and computational data models and solves a variety of problems in that space. In terms of problems, we have looked at the clustering problem, link analysis (like PageRank etc.) and social network analysis.

# Streaming Model of Computation
The streaming model works on the assumption of extremely limited memory (as compared to the size of the data). It takes the approach of seeing the data, working on it (modifying state etc.) and forgetting it. The computation power, on the other hand, is assumed to be unlimited (at least wrt memory).

Typically, a stream consists of a universe of elements, a long array of items and a query that is asked on the items seen so far.

## Uniform Sampling
*Query: Choose an item uniformly at random from the stream*

We use the technique of reservoir sampling and extend it to choosing *k items UAR from the stream*.

## AMS algorithm for second frequency moment

# Clustering Problem
Clustering is a well known problem of partioning points into clusters.

# Parallel/Distributed Model of Computation
"All models are wrong, but some are useful"

In the MPI (Message Passing Interface) style of parallel computation model, we have the MPC (Massively Parallel Computation) model and the k-machine model.

## Parallel k-means++ algorithm
The k-means++ algorithm initializes the set of clusters by picking points based on their contribution to the error. It finds a set C of k clusters (at the end of k iterations) which is a good approximation on the final set of cluster centers. However, it is sequential - the choice of the next center depends on the current set of centers.

The k-means parallel algorithm samples each point uniformly at random wrt the error on the previous set of centers (which it oversamples).

The [k-means parallel algorithm](http://theory.stanford.edu/~sergei/papers/vldb12-kmpar.pdf) optimizes by choosing a larger set of points in each round (oversampling) to take a logarithmic number of iterations, while the approximation guarantees on the results still hold.

Moreover, the algorithm can be parallelized in the MapReduce paradigm or the MPC model.

# Link Analysis - Page Rank
PageRank is a well studied and completely well known algorithm.

[Link](tba) is a good link explaining the Page Rank from computations point of view. 

The main ideas of PageRank are listed here:
1. Weightage based on number of outgoing links.
2. Weightage based on importance of a link.
3. In case a set of pages have no outgoing links, it becomes a Spider Trap (since the web pages are usually scoured by spiders). A random walker has a very high chance of ending up in a spider trap without being able to come out. 

For this, we add a transition probability, according to the random surfer model, where we assume that a surfer either follows links or jumps to a completely random page.

Using Markov Chains and random walks, the PageRank can be viewed as a stationary distribution. If we perform a random walk on the graph, the PageRank can be measured as the probability that the walk is at a particular vertex i, at a particular point of time.

## PageRank in the k-machine model
For a large number of vertices, we can solve the PageRank problem in a distributed setting. We use the CC (Congested Clique) model, where the n vertices (representing the n webpages) are distributed across the k machines and use the "think like a vertex" paradigm.

Each node starts with $O(\frac{log n}{r})$ random walk tokens and performs walks on the web graph. Each token operates independently of each other and chooses between a neighbor or a random vertex (termination).

The algorithm runs until all tokens terminate. Each node tracks the count of tokens that run through it and use it to measure the probability of a token landing up in its page. 

## Topic Sensitive Page Rank
Basically, we divide the total set of webpages into a broad set of topics and assign a PageRank value wrt each topic. We use the user's browsing history and use PR values of that topic.

The main idea, is that for a topic-sensitive PageRank, we assume that random surfers will end up only in pages related to that topic instead of any random page in the web.

## Link Spam

The best way to alleviate link spam is to use TrustRank. The TrustRank algorithm uses a reduced teleport set of trusted pages.

## HITS: Hypertext Induced Topic Selection
[Link](http://pi.math.cornell.edu/~mec/Winter2009/RalucaRemus/Lecture4/lecture4.html) explains HITS as a more natural method of searching, which divides webpages into hubs and authorities. 

HITS algorithm assigns each page a hub score and an authority score which are defined recursively. A good authority page is linked to by many hubs. A good hub page links to many authorities.

Authority weights is the dominant eigen vector of AtA. Hub weights is the dominant eigenvector of AAT.

# Social Network Analysis
## Clustering Problem - Betweenness
The clustering problem on a social network graph tries to split the graph into clusters (or communities) such that there are more links within communities that those between communities.

The betweenness of an edge is the measure of a role an edge plays between communities. It is measured as the proportion of shortest paths between all pairs of edges which pass through that particular edge. A high betweenness value indicates a bridge edge.

The Girvan-Newman Algorithm performs BFS considering all nodes as root and assigns credit starting from the leaf to the root. 

## Graph Partitioning - Spectral Graph Theory
Given a graph, we consider the problem of partitioning the vertices into 2 balanced sets within minimum interaction between the 2 sets. If we define Cut(S,T) as the number of edges going between S and T and Vol(S) as the total number of edges with at least one vertex in the set S.

We define a normalized cut value to ensure that the volumes of the 2 sets are more or less equal.

There is an interesting algorithm using the Laplacian Matrix, which involves assigning each edge the value (xi - xj)^2, where the value xi is simply +1 for set S and -1 for set T.

Edges within set S and set T contribute 0 to the score, whereas edges between contribute 4. The total value comes out to 4.Cut(S,T)

It turns out however, that it corresponds to the eigenvector corresponding to the second lowest eigenvalue.

## Affiliation Model
It is an abstract method of generating social N/W graphs given communities. Communities usually consist of a set of members/individuals. Membership in a community is a parameter model. 

We define p_c as the probability of an edge between 2 nodes within a community C. 

We use MLE to compute the likelihood that a given graph is generated with these parameters.

### Aside - On Maximum Likelihood Estimation
MLE considers a model along with parameters which together generate some data.

The observations/data is known and the model is formulated by domain experts, so the parameters are the only unknowns. 

MLE allows us to estimate the model parameters under the assumption that the data was generated by that model.

## Simrank - PageRank for Social Networks

## Triangle Counting - Enumeration
In social networks, a triangle is when a friend of a friend is also a friend. Social Networks tend to have a higher number of triangles than a random graph. The triangle count of a graph helps determine to what extent a graph resembles a social network graph.

The brute force algorithm for enumerating the number of triangles is O(n^3) to test all triplets of vertices. For sparser graphs,
we look at the heavy hitter algorithm which is only O(m^3/2)

# Dimensionality Reduction
We consider the orthogonal problem of dimensionality reduction. A well known problem in data mining and machine learning, when the dataset has a large number of dimensions, many of which are redundant, we can reduce the data to the k most essential dimensions.

Specifically, we consider the following handpicked popular methods:

## Principal Component Analysis
While PCA is a very popular method, especially in machine learning, I have never considered it in the following manner, as mentioned in LRU.

We consider the orthonormal matrix E, a vector of the orthonormal eigenvectors of the dataset sorted in descending order. We can rotate the original matrix by multiplying it with the matrix E.

The benefit of such a rotation, is that the early axes are more dominant and hold more information.

If we take a reduced k dimensional approximation of the rotated dataset (using a reduced basis), we can retain more information while losing lesser data.

## Singular Value Decomposition

## Johnson Lindenstrauss Theorem


# Resources

