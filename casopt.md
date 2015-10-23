# DataStax CASOPT notes
* [Operations and performance tuning](https://academy.datastax.com/courses/ds210-operations-and-performance-tuning)

### Managing Cassandra
#### [Adding nodes](https://academy.datastax.com/courses/ds210-operations-and-performance-tuning/managing-cassandra-managing-cassandra-and-adding)
When you startup a new node there are four settings that must be changed in *cassandra.yaml*
```
cluster_name
```
All nodes in a cluster must agree on a name of a cluster they are part of
```
rpc_address
listen_address
```
IP addresses that the nodes/clients will be communicating over
```
-seeds
```
List of nodes to initially connect to gather cluster topology. For operations simplicity it is prefered to use the same seed list for all machines

We add new nodes in following scenarios

* Too much data - data has outgrown the node's hardware capacity. Currently best recommendation of how much data to store depends on a type of media that is used to store data. If it is based on rotational drives it is recommended to go up to **1 GB** per server. If it is SSD backed storage it can be much higer - **3-5GB** per node.
* Too much traffic - if servers are not able to handle high troughput or latency is too high
* Operational headroom - compactions, repairs

#### [Adding nodes best practices] (https://academy.datastax.com/courses/ds210-operations-and-performance-tuning/managing-cassandra-best-practices-adding-nodes)
When adding new nodes to a **single-token cluster** there are some best practices to follow
* It is best if we can double the size of a cluster because adding single node can cause cluster to be unbalanced
  * can minimalize latency impact on a production load, where token recalculation and data movement can affect performance
  * hot spots are minimalized during data movement to a new nodes
With **vnodes cluster** we can add nodes when needed because
* token ranges are distributed and assigned automatically. New node coming online advertises number of tokens it owns and rest of a nodes in a cluster will stream data (for token ranges) back to that node
Adding multiple nodes at the same time
* **vnodes cluster** - start up all nodes at once, otherwise you will end up pushing the same data to new nodes more than once as data reshuffles its position
* **single-token cluster** there is no much difference - recommendation is one at the time to minimalize streaming effort
#### [Add, Remove, Decommission, Move Nodes](https://academy.datastax.com/courses/ds210-datastax-enterprise-operations-and-performance-tuning/managing-cassandra-add-remove)
