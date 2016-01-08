# DataStax CASOPT notes
### [Operations and performance tuning](https://academy.datastax.com/courses/ds210-operations-and-performance-tuning)

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

* Too much data - data has outgrown the node's hardware capacity. Currently best recommendation of how much data to store depends on a type of media that is used to store data. If it is based on rotational drives it is recommended to go up to **1 TB** per server. If it is SSD backed storage it can be much higer - **3-5TB** per node.
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

#### [Bootstrap](https://academy.datastax.com/courses/ds210-operations-and-performance-tuning/managing-cassandra-bootstrap)
Node bootstrapping consist of a following steps

1. Joining node contact a seed node
2. Seed node sees incoming communication from the joining node
  * Shares cluster information with the joining node and token information that is assigned
  * Seed node and the joining node handshake
3. Joining node calculates token(s) it is responsible for
  * Shares token with a seed node
  * Seed node and the joining node handshake again
4. Existing nodes prepare to stream data to the joining node
  * 30 second pause occurs to do following tasks for durability
    * Flush data in Memtables to disk
    * New system keyspace information flushes to disk for joining node
5. Existing nodes locate appropriate keys from SSTables for data to stream
6. Existing nodes stream only the SSTables that hold keys for the new node
  * No data is removed from the new node
7. Writes during this streaming period continue to be written to the existing node which owns it
  * This writes get forwarded to the new node
8. Joining node switching from JOINING to NORMAL state
  * Write and read read requests will go now to the new node
9. New node starts listener service for CQL calls (port 9042) and Thrift (port 9160)

Sometimes we don't want to auto bootstrap the node. The most common case is bringing up a new datacenter (we want to wait for all the nodes in a new DC to be up before streaming the data over). In that case we might want to set property in *cassandra.yaml*
```
auto_bootstrap: false
```

#### [Cleanup](https://academy.datastax.com/courses/ds210-operations-and-performance-tuning/managing-cassandra-cleanup)
Cleanup operation reclaims a disk space. There is no need to run it as data is reclaimed when it goes through compaction process however cleanup operation speeds things up. We usually run it in following cases
* Added new node to a cluster - each node shifts part if its partition range to the new node
* Decreased replication factor
Cleanup operation does "single table compaction" underneath - one SSTable at the time.
It has to run on a node that we want to clean a data and looks like
```
nodetool cleanup -- <keyspace> (<table>)
```
If node keyspace is specified it will clean all keyspaces

#### [Remove Node](https://academy.datastax.com/courses/ds210-operations-and-performance-tuning/managing-cassandra-remove-node)
#### [Decommission](https://academy.datastax.com/courses/ds210-operations-and-performance-tuning/managing-cassandra-decommission)
#### [Replace Node](https://academy.datastax.com/courses/ds210-operations-and-performance-tuning/managing-cassandra-replace-node)
Benefits of replacing a downed node (in comparsion to adding and removing a node)
* There is no need to shuffle data twice
* Backup for a node will work for replaced node because same tokens are used to bring replaced node into a cluster
To replace a downed node, bootstrap a new node using the IP address of the dead node as the **replace_address** value in JVM options in *cassandra-env.sh*
```
JVM_OPTS="$JVM_OPTS -Dcassandra.replace_address=<ip_address_of_a_dead_node>"
```
This means that this node advertises to rest of a nodes in a cluster that it is replacing IP address of a dead node and will own takens that were own by the dead node. Rest of a cluster will stream data for those tokens to a new machine.
When replacing a seed node there are some other considerations
* Go to every other node and change IP address in a seed list
* A new node will not bootstrap automatically if it has its IP in a seed nodes

### Maintaing Cassandra
#### [Hinted Handoff](https://academy.datastax.com/courses/ds210-operations-and-performance-tuning/maintaining-cassandra-maintaining-cassandra-and)
* Recovery mechanism for writes targeting offline nodes
* Coordinator can store a hinted handoff if target node is downed
  * is know to be downed
  * or failed to acknowledge
* Coordinator stores hints in its *system.hints* table
* Write is replayed when node comes back online
* If node is decommissioned or removed or table is dropped the hints are automatically removed

Hinted handoff is configured with following settings
```
hinted_handoff_enable: true
max_hint_window_in_ms: 10800000 # 3 hours
```
* hinted_handoff_enable - enable/disable hinted handoff
* max_hint_window_in_ms - how long hints are going to be stored from. We don't want to keeps hints for too long because we buffer them locally on a disk. This takes up storage and IO
  * Nodes offline longer are made consistent using repairs or other operations

How does it affect replication
* Within the hint window, if gossip repots a node is back up, the node with the hinted handoff information sends the data for each hint to the target
  * checks every 10 minutes (worst case) for any hints for writes that timed out during outage
* Hinted write doesn't count towards consistency level (CL) requirements
  * If there is no enough replica to satisfy CL an exception is thrown
  * Consistency level **ANY** guarantees that write is durable and will be readabl after replica comes back online and can receive hint reply

#### [Changing replication factor](https://academy.datastax.com/courses/ds210-operations-and-performance-tuning/maintaining-cassandra-changing-replication-factor)
First we have to understand Consistency Levels
![CL](https://github.com/netf/datastax-notes/blob/master/cl.png)
* Meeting the consistency level set for a read and a write depends on
  * replication factor
  * replicas available at the time of a read/write operation
* replication factor directly affect calculation of **QUORUM**
* **LOCAL_** and **EACH_** modify some consistency Levels
  * LOCAL_ONE, LOCAL_QUORUM, LOCAL_SERIAL restrict the validation to data center
  * EACH_QUORUM requires validation from each data center in the cluster

Consequences of changing the replication factor
* changing RF in production environment can have dire consequences (and isnt recommended)
* if RF is lowered to *RF = 1* then *QUORUM* becomes *ALL*
* if replication factor is raised, read and write failures may occur until repair operation completes
* consistency level of two or higher are more likely to cause blocking read repair

#### [Repair](https://academy.datastax.com/courses/ds210-operations-and-performance-tuning/maintaining-cassandra-repair)
Repair synchronsies replicas and there are few types of repairs
* read repair - a *digest* query is sent to replica nodes, and nodes with stale data are updated in a background. If data discrepency is found the data is repaired based on a timestamp (latest timestamp wins) and result is returned to the client
  * digest query returns a hash to check current data state, rather than return a complete query
  * **read_repair_chance**, set as a table property value between 0 and 1, to set probability with which read repair should be invoked on non-quorum reads
    * defautls to 0
    * can be configured per table
  * **dclocal_read_repair_chance**, set per data center. defaults to 0.1 (10% of our queries will perform LOCAL_ALL and check data across all replicas)
* repair (an action to cope with with cluster entropy)
  * entropy can arise from nodes that were down for longer than hint window, dropped mutations or other causes
  * a repair operates on all of a nodes in a replica set by default
  * ensures that all replicas have identical copies of a given partition
  * consist of following phases
    * build Merkle tree (tree of hashes) of data per partition
    * replicas then compare the difference between their trees and stream the data differences to each other as needed
    ![Merkle Tree](https://github.com/netf/datastax-notes/blob/master/merkle.png)
  * in some cases it is faster to bring a new node online
  * best practices
    * run this once a week (within **gc_grace_seconds** period - default 10 days)
    * run repair on a small subset of nodes at a time (preferably one)
    * schedule for a low hour usage
      * it is very IO heavy operation (validation compaction of Merkle trees)
      * can be mitigated with compaction throttling
    * run repair on a partition or subrange of partition

  We can run repair operation with following
  * repair a range of data
  ```
  nodetool repair --partitioner_range|-pr
  ```
    * repairs only primary range
    * otherwise repair will take 2 / 3 times longer depending on a number of replicas
  * repair a subrange
  ```
  nodetool repair --start-token <token> --end-token <token>
  ```
    * repairs only portion of data belonging to a node
    * steps to use subrange repair
      * use java *describe_splits* call to ask for a split containing 32K partition
      * iterate through the entire range incrementally or in parallel
      * pass the token received for the split to the nodetool repair -st and -et options
      * pass the --local option to repair only within the local data center, reducing cross data center transfer load

#### [Backup and recovery](https://academy.datastax.com/courses/ds210-operations-and-performance-tuning/maintaining-cassandra-backup-and-recovery)
Reasons to perform backup with Cassandra
* programmatic accidental deletion or overwriting of data
  * changes could be propagated to all replicas before error is noticed
  * corrupted SSTables can be restored from snapshots and incremental snapshots
* to recover from catastrophic single data center failure
  * offline snapshots can be used

Ways to take backup and snapshots
* snapshots
  * represents state of date in a particular point in time
  * consists of a single table, single keyspace, or multiple keyspaces
  * flushes all in-memory writes to SSTables on disk
  * makes hard links to SSTables for each keyspace in a snapshot directory
* incremental backups
  * data which is changed since last full snapshot
  * configured in *cassandra.yaml* with
  ```
  incremental_backup: true
  ```
  * snapshot information is stored under *snapshots* directory while incremental backups are stored in *backups* under a keyspace data directory
  * both snapshot and incremental backup are **needed** to restore data
  * incremental backups are not automatically cleared
* opscenter backup and restore service (DSE only)

Snapshots/Backups do not backup schema information so order to backup it is important to backup schema information as well

Default **auto_snapshot** ensures that snapshot will be taken when dropping a table (this is default and protects against human errors)

The **nodetool snapshot** command takes a snapshot of
* one or more keyspaces
* a table specified to backup data
```
nodetool snapshot [options] snapshot (-cf <table> | -t <tag> | -- keyspace)
```

When we want to remove snapshot we use **nodetool clearsnapshot**

To restore snapshot we can
* delete current data directory and copy the snapshot and incremental files to the data directories
  * if using incremental backups, copy the content of the backups directory to each table directory
  * table schema must already be present
  * restart (or run *nodetool refresh*) and repair the node after the file copying is done
* using *sstableloader*
  * needs to be careful - can add significant load to cluster while loading
  * if there is not an exactly alighment (for example from 4 node cluster to 6 node cluster) this just about the only way to restore

#### [Security](https://academy.datastax.com/courses/ds210-operations-and-performance-tuning/maintaining-cassandra-security)
Follwing security mechanisms are supported
* Client-to-node encryption
  * SSL is used to ensure data is not compromised while transferring between client and a server
* Node-to-node encryption
  * Protects data transfer between nodes in a cluster
* Transparent data encryption (**DSE only**)
  * Protects data stored on disk (data at rest)
* Authentication
  * Based on internally controlled accounts and passwords
  * Passwords internally are encrypted using **bcrypt**
  * LDAP/AD integration (**DSE only**)
* Object Permission Management
  * Authorization can be granted or revoked per user to access database objects
  ![Grant and revoke](https://github.com/netf/datastax-notes/blob/master/grant_revoke.png)


It is recommended to increase *system_auth* keyspace replication factor to the number of nodes in a cluster

#### [Rebuild Indexes](https://academy.datastax.com/courses/ds210-operations-and-performance-tuning/maintaining-cassandra-rebuild-index)
Why would you rebuild secondary indexes?
* Occasionally secondary indexes get out-of-sync
* if you suspect that this happened it is easier to rebuild the indexes than to verify that there is a problem

Rebuilding indexes:
* doesn't affect consistency
* doesn't drop the indexes
* adds the data again to the index
* consumes CPU and IO while operation happens

Cassandra indexes:
* under a hood are just cassnadra tables
* index the data that is on the node

Command to do that
```
nodetool rebuild_index keyspace table.index
```

#### [Multiple datacenters](https://academy.datastax.com/courses/ds210-operations-and-performance-tuning/multiple-data-centers-multiple-data-centers)
How are nodes organised as racks and data centers?
* Cluster of nodes can be logically grouped as *racks* and *data centers*
  * nodes - the virtual or physical host of a single Cassandra node
  * racks - the logical grouping of physical related nodes
  * data centers - logical group of a set of racks
* Enables geographically aware read nad write request routing
  * cluster topology is communicated as the **Snitch** and **Gossip**
* Each node belongs to one rack in one data center
* The identity of each node's rack and data center may be configured in its **conf/cassandra-rackdc.properties** file (when using *GossipingPropertyFileSnitch*)

Reasons for second DC
* Ensures continous availability of application
  * If one DC is affected by disaster, Cassandra keeps on working in second DC
* Live backup
  * If data centers are in the same physical location, but are configured as separate virtual data centers then a fallback cluster can be quickly switched on when needed  
* Improved performance
  * Latency is reduced when data is accessed from a local DC
* Search (*solr*) and Analytics (*spark*) integration (**DSE only**)
  * Workload isolation

How does Cassandra cluster operate between data centers?
* A data center is a grouping of nodes configured together for replication purposes
  * different replication factors and consistency levels can be configured
* Data replicates across data centers automatically and transparently
  * Data replicates in each data center depending on the replication factor (using remote coordinator)
* Consistency level *LOCAL* restricts actions to the local data center
* Consitency level *EACH* all data centers to be read/write to receive acknowledge

What happens when DC goes down?
* Failure of a data center will likely go unnoticed
* if node(s) fail, they will stop communicating via gossip
  * will be marked as DOWN but it takes around 10 seconds to figure that out
* recovery can be accomplished in following ways
  * if outage is brief - under *max_hint_window_in_ms* - hints will be sent to the DC after it comes back online
  * if outage is longer than *max_hint_window_in_ms* but shorter than *gc_grace_seconds* rolling repair can be used
  * if it's longer than *gc_grace_seconds* it is better to bootstrap a new DC

How to use multiple DC
* Use **NetworkTopologyStrategy** instead of SimpleStrategy
  * Allows for awareness for racks and datacenters
  * Can specify number of replicas per DC
* Use **LOCAL_** consistency level for read/write operations to limit latency
* If possible define one rack for entire DC (basically replication factor should be divisible by the number of racks - one rack makes things simple)
* Specify the Snitch
  * informs Cassandra about network topology
  * ensures requests are routed efficently
  * allows cassandra to distribute replicas
  * all nodes must have exactly same snitch configuration
  ![Snitches](https://github.com/netf/datastax-notes/blob/master/snitches.png)

Some considerations when using multiple DC
* WAN bandwidth - how quickly can we push data to a new DC
* Consistency level - how much data must be pushed to establish correct amount of repication needed to meet consitency level
* OpsCenter is a usuful tool for watching the progress of a new DC coming up

### Performance Tuning
#### [Goals](https://academy.datastax.com/courses/ds210-operations-and-performance-tuning/performance-tuning-goals)
It is important to clearly define a goal for performance tuning. Following are the attributes of a goal
* Type of operation or query
  * Read or Write
  * SELECT, INSERT, UPDATE, DELETE
* Latency
  * Expressed as a percentile rank - ie: "95th" percentile read latency is 2 ms
* Throughput
  * Expressed as operations per second
* Size
  * Expressed in averages of bytes
* Duration
  * Expressed in minutes or hours
* Scope
  * Keyspace, table

Example of a well defined goal
  ![Goal](https://github.com/netf/datastax-notes/blob/master/perf_goal.png)

Goal verification
* Timing hooks in the application
* Query tracing
* jmeter test plan
* customizable cassandra-stress

Latency diagram
![Latency](https://github.com/netf/datastax-notes/blob/master/perf_latency.png)
Common latency timings in Cassandra (single servers)
* Reads from main memory should take between **36 - 130** microseconds
* Reads from SSDs should take take between **100** microseconds and **12** miliseconds
* Reads from Serial Attached SCSI rotational drives should take between **8** miliseconds and **40** miliseconds
* Reads from SATA rotational drives take more than **15** miliseconds

#### [Workload characterization](https://academy.datastax.com/courses/ds210-operations-and-performance-tuning/performance-tuning-workload-characterization)
Here are some important questions to ask about a load
* what is the load being places on your cluster
  * Calling application or API
  * remote IP address
* Who is causing the load
  * code path or stack trace
* Why is the load being called
* What are the load characteristics
  * Throughput
  * Direction (read/write)
  * Include variance (standard deviation)
  * Keyspace and table
* How is the load changing over time and is there a pattern
* Is your workload read heavy or write heavy
* How big is your data (per cluster)
* How much data on each node (data density per node)
* Does active data fits into buffer cache

#### [Performance Impact of Data Model](https://academy.datastax.com/courses/ds210-operations-and-performance-tuning/performance-tuning-performance-impact-data-model)

How does data model affect performance. In Cassandra data model is tightly coupled to performance - has more impact than anything else.
Impact of data model on performance
* Poorly shaped rows (too narrow or too wide)
  * Too narrow - partitions are too small so to get data that we need we have to query multiple partitions
* Hotspots (particular areas with loads of read/writes)
* Poor primary or secondary indexes
* Too many tombstones
  * Workload that has lots of deletes and then reading data around the partitions where data was deleted

Data model considerations
* Understand how primary key affects performance
* Take a look at query patterns and adjust how tables are modeled
* See how replication factor and/or consistency level impact performance
* Change in compaction strategy can have a possitive (or negative) impact
* Parallelize reads/writes if necessary
* Look at how moving infrenquently accessed data can improved performance
* See how per column family cache is having an impact

What is the relationship between the data model and Cassandra's read path optimizations (key/row cache, bloom filters, index)?
Relationship between data model and Cassandra's read path optimization
* Nesting data
  * Nesting data allows for greater degree of flexibilty in the column family structure
* Model to keep the most active data sets in a cache
  * Frequently accessed data which are in cache can improve performance  

#### [Methodologies](https://academy.datastax.com/courses/ds210-operations-and-performance-tuning/performance-tuning-methodologies)
Performence tuning methodologies
* Active performence tuning
  * Isolate problems using provided tools
  * Determine if problem is with Cassandra, environment or both
  * Verify problems and test reproducibility
  * Fix problems using tuning strategies provided
  * Test, test and test again
* Passive performance tuning
  * Regulary monitor key health areas in Cassandra/Environment using tools provided
  * Identify and grow for future growth/scalability
  * Apply tuning strategies as needed
* The USE method (utilisation, saturation and errors)
  * Real goal is of this method is to collect methodolically whole spectrum of data - so we can have a complete view of the system

### Cassandra Tuning
#### [Apache Cassandra](https://academy.datastax.com/courses/ds210-operations-and-performance-tuning/cassandra-tuning-cassandra-tuning)
##### Cassandra Cluster level activities
* coordinator
* gossip
* replication
* repair
* read repair
* bootstrapping
* node removal
* node decommissioning

##### What does single Cassandra node do?
* Read
* Write
* Maintain consistency
* Monitor
* Participate in a cluster

##### How does Cassandra organize all of that work?
* Staged-Event-Driven-Architecture (SEDA)
  * Separates different tasks into stages that are connected by message passing
  * Each like task is grouped into a stage having a queue and a thread pool

##### What is a thread pool?
  ![Thread pool](https://github.com/netf/datastax-notes/blob/master/threadpool.png)

##### What are Cassandra's thread pools
  ![Cassandra's Thread pools](https://github.com/netf/datastax-notes/blob/master/cassandrathreadpool.png)
In Cassandra we check C* Thread Pool statistics using **nodetool tpstats**. It has following headers:
* Active - number of messages pulled off the queue, currently being processed by a thread
* Pending - number of messages in a queue waiting for a thread
* Completed - number of messages completed
* Blocked - when pool reaches its max thread count it will begin queuing until the max size is reached. When this is reached it will block until there is a room in the queue
* Total Blocked/All Time Blocked - total number of messages that have been blocked

Output of **nodetool tpstats**
```
Pool Name                    Active   Pending      Completed   Blocked  All time blocked
MutationStage                     0         0          11416         0                 0
ReadStage                         0         0        3423564         0                 0
RequestResponseStage              0         0             37         0                 0
ReadRepairStage                   0         0              0         0                 0
CounterMutationStage              0         0              0         0                 0
MiscStage                         0         0              0         0                 0
HintedHandoff                     0         0              0         0                 0
GossipStage                       0         0              0         0                 0
CacheCleanupExecutor              0         0              0         0                 0
InternalResponseStage             0         0              0         0                 0
CommitLogArchiver                 0         0              0         0                 0
CompactionExecutor                0         0         178199         0                 0
ValidationExecutor                0         0              0         0                 0
MigrationStage                    0         0             13         0                 0
AntiEntropyStage                  0         0              0         0                 0
PendingRangeCalculator            0         0              1         0                 0
Sampler                           0         0              0         0                 0
MemtableFlushWriter               0         0            493         0                 0
MemtablePostFlush                 0         0           9877         0                 0
MemtableReclaimMemory             0         0            493         0                 0
Native-Transport-Requests         0         0        6350258         0                 0

Message type           Dropped
READ                         0
RANGE_SLICE                  0
_TRACE                       0
MUTATION                     0
COUNTER_MUTATION             0
BINARY                       0
REQUEST_RESPONSE             0
PAGED_RANGE                  0
READ_REPAIR                  0
```
How Thread Pools relate to system components (single threaded are in red)
![Thread pool to system component](https://github.com/netf/datastax-notes/blob/master/threadrelate.png)
##### Multi-Threaded stages
* ReadStage - (affected by main memory, disk) - perform local reads
* MutationStage - (affected by CPU, main memory, disk) - perform local insert/update, schema merge, commit log replay, hints in progress
* RequestResponseStage - (affected by network, other nodes) - when a response to a request is received this is the stage used to execute any callbacks that were created with the original requests
* FlushWriter - (affected by CPU, disk) - sort and flush memtables to disk
* HintedHandoff - (one thread per host) - sends missing mutations to other nodes
* MemoryMeter - (several separate threads) - measure memory usage
* ReadRepairStage - (affected by network, other nodes) - perform read repair
* CounterMutation - perform counter writes on non-coordinator nodes and replicates after a local write
* InternalResponseStage - Responds to non-client initiated messaging, including bootstrapping and schema changes

##### Single-Threaded stages
* GossipStage - (affected by network) - Gossip communication
* AntiEntropyStage - (affected by network, other nodes) - Build Merkle trees and repair consistency
* MigrationStage - (affected by network, other nodes) - Make schema changes
* MiscStage - (affected by disk, network, other nodes) - Snapshotting; replicating data after node remove
* MemtablePostFlusher - (affected by disk) - Operations after flushing a memtable. Discard commit log files that have all data in them persisted to sstables. Flush non-column family backed seconadry indexes
* Tracing - query tracing
* CommitLogArchiver - back up or restore commit log

##### Message Types
* Handled by ReadStage thread pool
  * READ: read data from cache or disk
  * RANGE_SLICE: read a range of data
  * PAGE_RANGE: read part of a range of data
* Handled by MutationStage thread pool
  * MUTATION: write (insert or update or delete) data
  * COUNTER_MUTATION: changes counter column
  * READ_REPAIR: update out-of-sync data discovery during a read
* Handled by ReadResponseStage
  * REQUEST_RESPONSE: respond to coordinator
  * \_TRACE: used to trace a query

##### Why are some messages types droppable?
* Some messages can be dropped to prioritize activites in Cassandra when high resource contention occurs
* Dropped messages should be investigated

### Cassandra stats observability tools
* nodetool

* OpsCenter



Links:
* http://www.pythian.com/blog/guide-to-cassandra-thread-pools/
