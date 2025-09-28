
### 1.Introduction
- In the previous course, Introduction to Neo4j Graph Data Science, you learned about the graph catalog and how to create and managed graph projections.
- In this module, you will learn about the robust collection of graph algorithms available in GDS and how to use them on real data.
- We will start this module off with a lesson on algorithms tiers and execution modes so you have an understanding of general usage patterns.
- After that we will cover our 5 biggest algorithm categories.

The module outline is as follows:
- Algorithm Tier and Execution Modes
- Centrality and Importance Algorithms
- Path Finding Algorithms
- Community Detection Algorithms
- Node Embeddings
- Similarity Algorithms


### 2.Algorithm tier and Execution Modes:
- GDS algorithms are classified into three tiers: alpha, beta, and production.

- **Production-quality:** 
- Indicates that the **algorithm has been tested in regard to stability and scalability.** 
- Algorithms in this tier are prefixed with **gds.<algorithm>**.

- **Beta:** 
- Indicates that **the algorithm is a candidate for the production-quality tier.** 
- Algorithms in this tier are prefixed with **gds.beta.<algorithm>**.

- **Alpha:** 
- Indicates that **the algorithm is experimental and might be changed or removed at any time.**
- Algorithms in this tier are prefixed with **gds.alpha.<algorithm>.**

### 3.Execution modes:
- GDS algorithms have **4 executions modes** which determine how the results of the algorithm are handled.
- **stream:** **Returns** the **result of the algorithm** as a **stream of records**.
- **stats:** **Returns** a **single record of summary statistics**, but **does not write to the Neo4j database or modify any data.**
- **mutate:** **Writes the results of the algorithm** to the **in-memory graph projection** and **returns a single record of summary statistics**.
- **write:** **Writes the results of the algorithm** back the **Neo4j database** and **returns a single record of summary statistics**.

- **NOTE:**
- **Only production tier algorithms guarantee the existence of all execution modes.**

### 4.Memory Estimation
- As the **size of data grows**, a ubiquitous challenge for Data Science practitioners is figuring out 
- **how much memory is required to support** their **analytics and machine learning workflows**. 
- This can often **require a lot of experimentation** and **trial and error**.
- To circumvent this, **GDS offers an estimation procedure** which allows you to **estimate the memory needed for using an algorithm**
- on your data BEFORE actually executing it.
- To use the estimation procedure for different algorithms and execution modes you can simply append the command with **.estimate**.

- **NOTE:**
- Only production tier algorithms guarantee the existence of estimation procedures across all execution modes.

- **SYNTAX**
```
CALL gds[.<tier>].<algorithm>.<execution-mode>[.<estimate>](
	graphName: STRING,
	configuration: MAP
)
```

- **Check your understanding**
- 1. Alpha Tier Algorithms
- What kind of support can you expect from alpha tier algorithms?
```
Alpha tier algorithms are experimental and might be changed or removed at any time
```

- 2. Which of the following execution modes writes algorithm results back to the Neo4j database?
```
Write
```

### 5. Centrality and Importence
- **Centrality algorithms** are used to determine the **importance of distinct nodes in a graph**
- Common use cases of centrality include:

- **Recommendations:** 
- Identify and recommend the most influential or popular items in your content or product offering catalog.

- **Supply chain analytics:**
- **Find the most critical node in your supply chain**, whether it be a supplier in a network,
- a raw material that is part of a manufactured product, or a port in a route.

- **Fraud & Anomaly Detection:**
- **Find users with many shared identifiers or who otherwise act as a bridge between many communities**

### 6. Degree Centrality Example
- **Degree centrality** is one of the **most ubiquitous** and **simple centrality algorithms**. 
- **It counts the number of relationships a node has**. In the GDS implementation, 
- we specifically **calculate out-degree centrality** which is the **count of outgoing relationships from a node**.
- Below is an example of using **degree centrality** to **count the number of movies each actor has acted in**.
- During this module, you will be using a movie recommendations dataset that contains information about movies, actors, and users who have rated movies.

- To use a **GDS algorithm**, you must first create a **graph projection**. 
- **A projection is an in-memory graph you can quickly query and manipulate.**


- Create a graph projection of Actor and Movie nodes:
```
CALL gds.graph.project(
  'proj',
  ['Actor','Movie'],
  'ACTED_IN'
  );
```

- Then stream the degree centrality to find the actors who have acted in the most movies:
```
//get top 5 most prolific actors (those in the most movies)
//using degree centrality which counts number of `ACTED_IN` relationships

CALL gds.degree.stream('proj')
YIELD nodeId, score
RETURN
  gds.util.asNode(nodeId).name AS actorName,
  score AS numberOfMoviesActedIn
ORDER BY numberOfMoviesActedIn DESCENDING, actorName LIMIT 5
```

### 7. Pagerank Example
- Another common centrality algorithm is **PageRank**. 
- **PageRank **is a good algorithm **for measuring the influence of nodes in a directed graph**,
- particularly **where the relationships imply some form of flow of movement such as in payment networks, supply chain and logistics**,
- communications, routing, and graphs of website and links.

- **PageRank** was originally developed by Google co-founders Larry Page and Sergey Brin at Stanford University in 1996
- as part of a research project about a new kind of search engine. 
- **It has since been used by Google Search to rank web pages in their search engine results.**

- In summary, **PageRank** **estimates** the **importance of a node by counting the number of incoming relationships from neighboring nodes**
- **weighted by the importance and out-degree centrality of those neighbors**. 
- The underlying assumption is that more important nodes are likely to have proportionately more incoming relationships from other important nodes.

- Below is an example of applying PageRank to find the most influential persons in the Director → Actor network from movies
- released on or after 1990 with a revenue of at least 10 Million dollars.

- First, create the graph projection.
- We can use a Cypher projection to create an in-memory graph with :DIRECTED_ACTOR relationships between two (:Person) nodes. 
- This graph can be traversed to understand the influence across directors and actors.
```
// drop last graph projection
CALL gds.graph.drop('proj', false);

// create Cypher projection for network of people directing actors
// filter to recent high grossing movies
MATCH (source:Person)-[:DIRECTED]->(m:Movie)<-[:ACTED_IN]-(target)
WHERE m.year >= 1990 AND m.revenue >= 10000000
WITH source, target, count(*) as actedWithCount
WITH gds.graph.project(
  'proj',
  source,
  target,
  {
    relationshipType: "DIRECTED_ACTOR"
  }
) as g
RETURN
  g.graphName AS graph, g.nodeCount AS node, g.relationshipCount AS rels
```
- Next stream PageRank to find the top 5 most influential people in director-actor network.
```
CALL gds.pageRank.stream('proj')
YIELD nodeId, score
RETURN
  gds.util.asNode(nodeId).name AS personName,
  score AS influence
ORDER BY influence DESCENDING, personName LIMIT 5
```

- Other Centrality Algorithms
- Other GDS production tier centrality algorithms include:
- **1. Betweenness Centrality:**
- Measures the extent to which a node stands between the other nodes in a graph. 
- It is often **used to find nodes that serve as a bridge from one part of a graph to another.**

- **2. Eigenvector Centrality:**
- Measures the transitive influence of nodes.
- **Similar to PageRank, but works only on the largest eigenvector of the adjacency matrix** so does not converge
- in the same way and tends to more strongly favor high degree nodes. 
- It can be more appropriate in certain use cases, particularly those with undirected relationships.

- **3. Article Rank:**
- A variant of PageRank which assumes that **relationships originating from low-degree nodes have a higher influence than relationships from high-degree nodes**.



- **Check you understanding**
- 1. Algorithm Purpose
- What do centrality algorithms generally measure?
```
The importance of distinct nodes in a graph
```

- 2. Name the Algorithm
- Which centrality algorithm is based solely on a count of relationships for each node
```
degree centrality
```



- **Which actor has directed the most movies?**
- Update and run these 2 GDS calls to find the actor that has directed the most *movies*.
- Create a graph projection using the gds.graph.project() procedure.
- Include Movie and Actor nodes with the DIRECTED relationship type.
```
// Create graph projection
CALL gds.graph.project(
    'actor-directors',
    ['Movie', 'Actor'],
    'DIRECTED'
)
```
- Run the degree centrality algorithm on the projected graph using the gds.degree.stream() procedure.
- Order the results to determine which actor has directed the most movies.
```
CALL gds.degree.stream('actor-directors')
YIELD nodeId, score
RETURN
  gds.util.asNode(nodeId).name AS name,
  score AS movies
ORDER BY movies DESC
```

- Enter the name of the actor (the answer is case sensitive):
```
Woody Allen
```

### 8. Path Finding
- **Path finding algorithms find the shortest path between two or more nodes or evaluate the availability and quality of paths.**
- Common use cases of path finding are:
- **Supply chain analytics:** Identifying the fastest path between an origin and a destination or between a raw material and a finished product
- **Customer Journey:** Analyzing the events that make up a customer’s experience. 
- In healthcare for example, this can be the experience of an in-patient from admission to discharge.

### 9. Dijkstra Source-Target Shortest Path
- A common, industry standard, path finding algorithm is Dijkstra. 
- **It computes the shortest path between a source and a target node.**
- Like many other path finding algorithms in GDS, 
- **Dijkstra supports weighted relationships to account for distance or another cost property when comparing paths.**

- Below is an example of using **Dijkstra source-target shortest path to find the shortest path between the actors "Kevin Bacon" and "Denzel Washington".**
```
CALL gds.graph.project(
	'proj',
    ['Person','Movie'],
    {
        ACTED_IN:{orientation:'UNDIRECTED'},
        DIRECTED:{orientation:'UNDIRECTED'}
    }
);
```

- Below is dijkstras path finding algorithm
```
MATCH (kevin:Actor{name : 'Kevin Bacon'})
MATCH (denzel:Actor{name : 'Denzel Washington'})

CALL gds.shortestPath.dijkstra.stream(
    'proj',
    {
        sourceNode:kevin,
        TargetNode:denzel
    }
)

YIELD sourceNode, targetNode, path
RETURN sourceNode, targetNode, nodes(path) as path;
```

- This should give you a path consisting of four relationships between Kevin Bacon and Denzel Washington.


### 10. Other Path Finding Algorithms
- Other GDS production tier Path Finding algorithms can be split into a few different subcategories that are listed below:


### 11. Shortest path between two nodes:
- **A Shortest Path:** 
- An **extension of Dijkstra** that uses a **heuristic function** to **speed up computation**.

- **Yen’s Algorithm Shortest Path:**
-  An **extension of Dijkstra** that allows you to **find multiple, the top k, shortest paths**.

### 12. Shortest path between a source node and multiple other target nodes:
- **Dijkstra Single-Source Shortest Path:**
- Dijkstra implementation for shortest path **between one source and multiple targets.**

- **Delta-Stepping Single-Source Shortest Path:**
- Parallelized shortest path computation. **Computes faster than Dijkstra single-source shortest Path but uses more memory.**

### 13. General path search between a source node and multiple other target nodes:
- **Breadth First Search:** 
- Searches paths in order of increasing distance from the source node on each iteration.

- **Depth First Search:** 
- Searches as far as possible along a single multi-hop path on each iteration.



- ** Check your understanding**
- What can path finding algorithms accomplish in GDS (select all that apply)?
```
- Find the shortest path between two nodes
- Find the shortest path between a single source node and a set of multiple target nodes
- Leverage breadth first or depth first search to find paths between a single source node and a set of multiple target nodes
```

- Which Path finding algorithm can be used to identify the top 10 shortest path between two nodes?
```
Yen’s Algorithm
```

- Shortest Path
- What is the shortest path between 'Kevin Bacon' and 'Peta Wilson'?
- You can use the same projection and a similar query to the previous lesson to find the answer.
- You will need to:
- Create a projection (or use the same one as the previous lesson) of Actor and Movie nodes and ACTED_IN and DIRECTED relationships.
- Create a query that matches the 2 Actor nodes and uses gds.shortestPath.dijkstra.stream function to find the shortest path
- Count how many relationships there are between the source and target nodes.
- Enter the number of relationship hops between 'Kevin Bacon' and 'Peta Wilson':
```
6
```

```
CALL gds.graph.project('proj',
    ['Person','Movie'],
    {
        ACTED_IN:{orientation:'UNDIRECTED'},
        DIRECTED:{orientation:'UNDIRECTED'}
    }
);
```

```
MATCH (kevin:Actor{name : 'Kevin Bacon'})
MATCH (peta:Actor{name : 'Peta Wilson'})

CALL gds.shortestPath.dijkstra.stream(
    'proj',
    {
        sourceNode:kevin,
        TargetNode:peta
    }
)

YIELD sourceNode, targetNode, path
RETURN sourceNode, targetNode, nodes(path) as path;
```


### 14. Community Detection
- Community detection algorithms are used to evaluate **how groups of nodes** may be **clustered or partitioned in the graph**.
- Much of the **community detection functionality** in GDS is focused on **distinguishing and assigning ids** to these node groups for downstream analytics, visualization, or other processing.
- Common use cases of community detection include:

- **1. Fraud detection**: 
- Finding fraud rings by identifying accounts that have frequent suspicious transactions and/or share identifiers between one another.

- **2. Customer 360**: 
- Disambiguating multiple records and interactions into a single customer profile so an organization has an aggregated source of truth for each customer.

- **3. Market segmentation**:
- dividing a target market into approachable subgroups based on priorities, behaviors, interests, and other criteria.


### 15. Louvain Community Detection
- A common community detection algorithm is Louvain.
- **Louvain maximizes a modularity score for each community**,
- where the **modularity quantifies the quality of an assignment of nodes to communities**. 
- This means **evaluating how much more densely connected the nodes within a community are, compared to how connected they would be in a random network**.
- Louvain optimizes this modularity with a hierarchical clustering approach that recursively merges communities together.
- There are **multiple parameters** that can be used **to tune Louvain** to control its **performance** and **the number** and **size of communities produced**.
- This includes the maximum number of iterations and hierarchical levels to use as well as the tolerance parameter for assessing convergence/stopping conditions.
- Our Louvain documentation covers these parameters and tuning in more detail.

- An additional important consideration is that Louvain is a **stochastic algorithm**.
- **A stochastic algorithm is an algorithm that uses randomness as part of its logic or execution**
- **stochastic algorithms can give different outputs on different runs, even with the same input**.

- As such, the community assignments may change a bit when re-run.
- When the graph does not have a naturally well-defined community structure the changes between runs can become more significant. 
- Louvain includes a **seedProperty** parameter which can be used to **assign initial community ids** and **help with consistency between runs**.
- Also, if consistency is important for your use case, 
- other community detection algorithms, such as **Weakly Connected Components (WCC)**,
- **take a more deterministic partitioning approach to assigning communities and thus will not change between runs**.

- Below is an example of running Louvain to understand communities of actors and directors in our movies recommendations graph.
```
CALL gds.graph.project('proj', ['Movie', 'Person'], {
    ACTED_IN:{orientation:'UNDIRECTED'},
    DIRECTED:{orientation:'UNDIRECTED'}
});
```

- Then we can run Louvain. 
- Here **we will run Louvain in mutate mode to save community Ids and return high level statistics on the community counts**, 
- distribution, modularity score, and information for how Louvain processed the graph.
```
CALL gds.louvain.mutate('proj', {mutateProperty:'communityId'})
```

- We can **verify the communityId node properties in the projection** with a **stream operation**.
```
CALL gds.graph.nodeProperty.stream('proj','communityId', ['Person'])
YIELD nodeId, propertyValue
WITH gds.util.asNode(nodeId) AS n, propertyValue AS communityId
WHERE n:Person
RETURN n.name, communityId LIMIT 10
```

### 16. Other Community Detection Algorithms
- Below are some of the other production tier community detection algorithms. 
- A full list of all community detection algorithms can be found in the Community Detection algorithms documentation.
- **Label Propagation**: **Similar intent as Louvain. Fast algorithm that parallelizes well. Great for large graphs**.
- **Weakly Connected Components (WCC)**: Partitions the graph into sets of connected nodes such that
- Every node is reachable from any other node in the same set
- No path exists between nodes from different sets
- **Triangle Count**: **Counts the number of triangles for each node**. Can be used to **detect the cohesiveness of communities and stability of the graph**.
- **Local Clustering Coefficient**: Computes the local clustering coefficient for each node in the graph 
- which is an indicator for how the node clusters with its neighbors.


- **1. Algorithm Purpose**
- What do community detection algorithms primarily accomplish? Select the best answer.
```
Detect how groups of nodes cluster together to form communities and strength of such clustering in the graph
```

- **2. Name the Algorithm**
- Which of the below community detection algorithms is focused on
- deterministically partitioning the graph into sets of disjoint nodes such that no path exists between sets?
```
Weakly Connected Components
```

### 17. Node Embeddings
- The goal of node embedding is to compute **low-dimensional vector representations of nodes**
- such that **similarity between vectors (eg. dot product) approximates similarity between nodes in the original graph**. 
- These vectors, also called embeddings, can be extremely useful for exploratory data analysis, similarity measurements, and machine learning.

- The below figure illustrates the concept behind node embedding, 
- whereby **nodes that are close together in the graph end up being close together in the 2-dimensional embedding space**. 
- The embedding thus took the structure from the graph, the n-dimensional adjacency matrix, and approximated it in 2-dimensional vectors for each node.
- **The embedding vectors are much more efficient to use for downstream process due to significantly reduced dimensionality**. 
- They could be used for cluster analysis for example, or as features to train a node classification or link prediction model.

- Of course, in real-world problems **node embeddings will usually be larger than 2 dimensions**,
- often ending up in the hundreds or larger, especially when applied to bigger graphs with millions or billions of nodes. 
- Node embedding also doesn’t have to base similarity strictly on node proximity in the graph.
- While similarity based on **distance in relationship hops** and **common neighbors is perhaps most common in application**,
- node embedding can also consider node properties and other "global-view" node attributes when calculating embedding vectors.

- **Use Cases**
- Node embedding has applications across multiple use cases, 
- from recommendations, to anomaly and fraud detection, entity resolution and other forms of knowledge graph completion.
- Node embedding vectors don’t offer insights by themselves, they are created to enable or scale other analytics. Common workflows include:
- **1. Exploratory Data Analysis (EDA)**
- such as visualizing the embeddings in a TSNE plot to better understand the graph structure and potential clusters of nodes

- **2. Similarity Measurements**:
- Node embedding allows you to scale similarity inferences in large graphs using K Nearest Neighbor (KNN) or other techniques. 
- This can be useful for scaling memory based recommendation systems, such as variations of collaborative filtering.
- It can also be used for semi-supervised techniques in areas like fraud detection, where, for example, 
- we may want to generate leads that are similar to a group of known fraudulent entities.

- **3. Features for Machine Learning**:
- Node embedding vectors naturally plug in as features for a variety of machine learning problems.
- For example, in a graph of user purchases for on online retailer, 
- **we could use embeddings to train a machine learning model to predict what products a user may be interested in buying next**.


- **FastRP**
- GDS offers a custom implementation of a node embedding technique called Fast Random Projection, or FastRP for short.
- FastRP leverages probabilistic sampling techniques to generate sparse representations of the graph allowing for extremely fast calculation
- of embedding vectors that are comparative in quality to those produced with traditional random walk and neural net techniques such as Node2vec and GraphSage.
- This makes FastRP a great choice for getting started with exploring embedding on your graph in GDS.

- There are multiple tuning parameters for FastRP and in real-world applications these can be important to take into account. 
- A couple of considerable note below:
- **1. embeddingDimension**:
- Applies to all node embedding algorithms in GDS. **Controls the length of the embedding vectors**. 
- Setting this parameter is a trade-off between **dimensionality reduction** and **accuracy**. 
- **A larger embedding dimension will more accurately capture the graph structure** 
- **but will also take longer to generate and produce embedding vectors that take more memory and computation to handle downstream**.
- Choice of embedding dimension depends, in good part, on the number of nodes in the graph. 
- Since the amount of information the embedding can encode is limited by its dimension, 
- a larger graph will tend to require a larger embedding dimension. 
- A typical value is a power of two in the range 128 - 1024. 
- A value of at least 256 gives good results on graphs in the order of 100K nodes.

- **2. IterationWeights**: 
- This controls **two aspects**: 
- **the number of iterations for intermediate embeddings**.
- **their relative impact on the final node embedding**. 
- The parameter is a list of numbers, indicating one iteration per number where the number is the weight applied to that iteration. 
- the default is [0.0, 1.0, 1.0].
- In general, **the intermediate embedding corresponding to the i:th iteration contains features depending on nodes reachable with paths of length i**.

- There are other parameters that control strength of normalization and node self-influence.
- As always, you can reference these in detail in the docs.
- A last important note on FastRP is that, while we won’t cover it here,
- it has the ability to consider node and relationship property weights when generating embeddings. 
- This can be useful for **generating embedding vectors** that **encapsulate signal from both a weighted graph structure and other properties/attributes** in the data. 
- As always, you can reference the parameters and various configuration in more detail in the docs.


- **FastRP Example**
- Below is an example of generating FastRP embeddings on person nodes in the movies graph based on the movies they acted in and/or directed.
- As always we will start with the graph projection
```
CALL gds.graph.project('proj', ['Movie', 'Person'], {
    ACTED_IN:{orientation:'UNDIRECTED'},
    DIRECTED:{orientation:'UNDIRECTED'}
});
```

- After that we will run FastRP. 
- For demonstration purposes we will just use an embedding dimension of 64.
- We have the option of setting a randomSeed here as well to control consistency between runs
```
CALL gds.fastRP.stream('proj',  {embeddingDimension:64, randomSeed:7474})
YIELD nodeId, embedding
WITH gds.util.asNode(nodeId) AS n, embedding
WHERE n:Person
RETURN id(n), n.name, embedding LIMIT 10
```

- These embeddings, can, in theory, be used for similarity measurements to understand which actors are most similar 
- and can be used in a content recommendation system to recommend movies to users based on the actors and/or directors for movies they recently viewed.

- **Other Node Embedding Algorithms**
- GDS has also implemented Node2Vec, 
- which computes a vector representation of a node based on random walks in the graph, and GraphSage, 
- which is an inductive modeling approach for computing node embeddings using node properties and graph structure.


- **1. Algorithm Purpose**
- What does node embedding accomplish (select all that apply)?
```
- Generates low-dimensional vector representations of nodes
- Provides numeric vectors that can be used for exploratory data analysis and similarity measurements between nodes
- Provides numeric vectors for each node that can be used as features for machine learning models

```
- **2. Algorithm Tuning**
- Which is a valid consideration for setting the embeddingDimension tuning parameter?
```
- The number of nodes in the graph. Generally larger graphs need larger embedding vectors to capture the graph structure
- Processing time and memory considerations. Smaller embeddings take less time to compute and have a smaller memory footprint
- The accuracy/completeness needed in downstream analytics that use the embeddings. Larger embeddings tend to estimate the graph structure more accurately
```


### 18. Similarity
- **Similarity algorithms**, as the name implies, are used to infer similarity between pairs of nodes. 
- In GDS these algorithms run over the graph projection in bulk. 

- **When similar node pairs are identified according to the user specified metric and threshold**, 
- **a relationship with a similarity score property is drawn between the pair.**

- **Depending on which execution mode is used when running the algorithm,** 
- **these similarity relationships can be streamed, mutated to the in-memory graph,or written back to the database.**

- Common use cases for similarity include:
- **Fraud detection**:
- finding potential fraud user accounts by analyzing how similar a set of new user accounts is to flagged accounts
- **Recommendation Systems**: 
- In an online retail store, 
- identifying items that pair to the one currently being viewed by a user to inform impressions and increase rate of purchase
- **Entity Resolution**: 
- Identify nodes that are similar to one another based on activity or identifying information in the graph

### 19. Similarity Algorithms in GDS
- GDS has two primary similarity algorithms:
- **Node Similarity**: 
- Determines similarity between nodes based on the relative proportion of shared neighboring nodes in the graph. 
- Node Similarity is a good choice where explainability is important, and you can narrow down the universe of comparisons to a subset of your data.
- Examples of narrowing down include focusing on just single communities, newly added nodes, or nodes within a specific proximity to a subgraph of interest.

- **K-Nearest Neighbor (KNN)**:
- Determines similarity based off node properties. 
- The GDS KNN implementation can scale well for global inference over large graphs when tuned appropriately. 
- it can be used in conjunction with embeddings and other graph algorithms to determine the similarity between nodes based on proximity in the graph,
- node properties, community structure, importance/centrality, etc.

- **Controlling Scope of Results**
- For similarity comparisons we may also **want to control the number of results returned to only consider the most relevant node pairs**. 
- **Both Node Similarity and KNN have a topK parameter to limit the number similarity comparisons returned per node**. 
- With node similarity there is also the capability to limit the results globally as opposed to just a per node basis.

- **Applied Example with KNN**
- Let’s take the embeddings we calculated from the node embedding lesson and use them to determine similarity
- between the actors and directors based on movies they have been involved with. You can regenerate the projection with the below:
```
CALL gds.graph.project('proj', ['Movie', 'Person'], {
    ACTED_IN:{orientation:'UNDIRECTED'},
    DIRECTED:{orientation:'UNDIRECTED'}
});
```
- We will then run FastRP like in the last lesson except in mutate mode so the embeddings will be saved in the projection.
```
CALL gds.fastRP.mutate('proj',  {
    embeddingDimension:64,
    randomSeed:7474,
    mutateProperty:'embedding'
})
```
- After that we can run similarity. 
- We will use the default cosine metric. For purposes of demonstration, I will limit topK to one so we can see the top pairs for each node
```
CALL gds.knn.stream('proj', {nodeLabels:['Person'], nodeProperties:['embedding'], topK:1})
YIELD  node1, node2, similarity
RETURN gds.util.asNode(node1).name AS actorName1,
    gds.util.asNode(node2).name AS actorName2,
    similarity
LIMIT 10
```

- **Similarity Functions**
- In addition to the node similarity and KNN algorithms, 
- GDS also provides a set of functions that can be used to calculate similarity 
- between two arrays of numbers using various similarity metrics including jaccard, overlap, pearson, cosine similarity and a few others. 
- The full documentation can be found in the Similarity Functions documentation.
- These functions are useful when you are interested in measuring similarity between a single select node pair 
- at a time as opposed to calculating similarity over the entire graph.


- **1. Algorithm Purpose**
- What can similarity algorithms accomplish in GDS? (select all that apply)
```
- Find similarity between pairs of nodes based on the neighboring nodes that they share
- Calculate the similarity between node pairs based solely on the similarity of node properties
- Draw relationships to connect similar node pairs together in the graph
```

- Which Similarity algorithm can be used to find node pairs based off of node properties, 
- including properties that were assigned from other graph algorithms like embeddings or community sizes?
```
K-Nearest Neighbor (KNN)
```


















