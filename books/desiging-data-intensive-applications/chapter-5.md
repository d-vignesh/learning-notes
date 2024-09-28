## Distributed data:

- Reasons for which we might need to distrubute a database across multiple machines are: scalability, fault tolerance/high availability, latency. 
- problem with shared-memory architecture(vertical scaling) is that the cost grows faster than linearly and it offers limited fault tolerance. 
- another approach is shared-disk architecture, which uses several machines with independent CPU's and RAM, but stores data on an array of disks that is shared b/w machines. The overhead of locking limits to scalability of the shared-disk architecture.
- in shared-nothing architecture(horizontal scalaing) each machine running the db software is called node. Each node uses its CPUs, RAM and disks independently. 
- there are two common ways data is distributed across multiple nodes: 
    * replication: keeping a copy of the same data on several different nodes, potentially in different locations. 
    * partitioning: splitting a big database into smaller subsets called partitions so that different partitions can be assigned to different nodes. Also known as sharding. 

## Replication:
- replication means keeping a copy of the same data on multiple machines that are connected via a network. 
- reasons to replicate data: 
    * to keep data geographically close to user. 
    * to allow the system to continue working even if some of its parts have failed. 
    * to scale out the number of machines that can serve read queries. 
- all of the difficulty in replication lies in handling changes to replicated data.
- three populor algorithms for replicating changes between nodes are: single-leader, multi-leader, and leaderless replication. 

## Leaders and Followers:
- every node that stores a copy of the database is called a replica. every write to the db needs to be processed by every replica; otherwise, the replica would no longer contain the same data. The most common solution for this is called leader-based replication.
- in leader-based replication, writes are always processed by the leader replica. it writes the data to its local storage and sends this data change to all its follower replicas as part of a replication log or change stream. the followers updates its local storage by applying all write in the same order as they were processed on the leader. clients can query either leader or follower, but writes are only accepted on the leader. 
- this approach is used in Postgresql, mysql, mongodb, rethinkdb, expresso, kafka and rabbitMQ. 

## Synchronous vs asynchronous replication: 
- synchronous replication on db means that one of the followers is synchronous, and the others are asynchronous. if the synchronous follower becomes unavailable or slow, one of the asynchronous followers is made synchronous. this guarantees that we have an up-to-date copy of the data on at least two nodes: the leader and one synchronous follower. this configuration is called semi-synchronous. 
- often leader-based replication is asynchronous. in this case, if the leader fails and is not recoverable, any writes that have not yet been replicated to followers are lost. but it is nevertheless widely used, especially if there are many followers or if they are geographically distributed. 
### setting up new followers: 
* take a consistent snapshot of the leader's db at some point in time - if possible without taking a lock on the entire db.
* copy the snapshot to new follower node. 
* the follower connects to the leader and requests all the data changes that have happened since the snapshot was taken. this requires that the snapshot is associated with an exact position in the leader's replication log. that position has various names: Postgresql calls it log sequence number, mysql calls it binlog coordinates. 
* when the follower has processed the backlog of data changes since the snapshot, we say it has caught up. it can now continue to process data changes from the leader as they happen. 
### handling node outages
### follower failure - catch-up recovery: 
- each follower keeps a log of the data changes. If it crashes and restart, from its log, it knows the last transaction that was processed before the fault occured. thus, the follower can connect to the leader and request all the data changes that occured during the time when the follower was disconnected. 
### leader failure - failover: 
- if leader fails, one of the followers needs to be promoted to be the new leader, clients need to be reconfigured to send their writes to the new leader, and the other followers need to start consuming data changes from the new leader. this process is called failover. 
- an automatic failover process consists of following steps:
    1. determining that the leader has failed - there is no foolproof way of detecting failures, so systems use timeout, if a node doesn't responsd for some period of time, it is assumed dead. 
    2. chossing a new leader - done by election process. best candidate is the replica with the most up-to-date data changes from the old leader. 
    3. reconfiguring the system to use the new leader - clients needs to send write to new leader. if the old leader comes back it should become a follower. 
- things that can go wrong in failover:
    * the new leader might not have received all the writes from old leader. 
    * there are two nodes believing that they are leaders. this situation is called split brain. as a safety catch, some systems shut down one node if two leaders are detected. 
    * right timeout to declare that the leader is dead? 

## implementation of replication logs: 
- different replication methods are, 
### statement-based replication: 
- the leader logs every write request(statement) that it executes and sends the statement log to its followers. 
- some issues with this replication are, 
    * calls to non-deterministic function, such are Now() or rand() might generate a different value on each replica. 
    * statements having autoincrementing column or depend on the existing data in db. 
    * statements having side effects (e.g, triggers, stored_procedures) may result in different side effects occuring on each replica. 
### write ahead log shipping: 
- we can use the log(created in SSTables or B-Tree) to build a replica on another node: beside writing the logs to disk, the leader also sends it accross the network to its followers. when the follower processes this log, it builds a copy of the exact same data structure as found on the leader. 
- the disadvantage WAL is that the log describes the data on a very low level. a WAL contains details of which bytes were changed in which disk blocks. this makes replication closely to the storage engine. if the db changes its storage format from one version to another, it is typically not possible to run different versions of the db software on the leader and the followers. if the replication protocol does not allow different versions of masters and followers, then upgrades require downtime. 
### logical(row-based) log replication: 
- an alternative is to use different log formats for replication and for the storage engine, which allows the replication log to be decoupled from the storage engine internals.
- this replication log is called logical log, 
    * for inserted row, the log contains the values for all columns. 
    * for deleted row, the log contains row identifier. 
    * for updated row, the log contains row identifier and new values. 
- a transaction that modifies several rows generates several such log records, followed by a record indicating that the transaction was committed. 

## Problems with Replication Lag: 
- for read heavy workloads, we can have many followers and distribute the read requests across those followers. this requires asynchronous replication. but a read from an asynchronous follower may see outdated data. this inconsistency is temporary and follower will catch up eventually. this effect is known as eventual consistency. 

### reading your own writes:
- with read-after-write consistency, we can guarantee that if the user reloads the page, they will always see updates they submitted themselves. it makes no promises about other users: other user's updates may not be visible until some later time. 
- read-after-write consistency can be implemented by, 
    * when reading something that the user may have modified, read it from the leader; otherwise, read it from a follower. 
    * if cases where most things are editable by user, we could track the time of the last update and for one minute after the last update, make all reads from the leader. 
    * the client can remember the last update time and ensure that the replica serving the read request reflects the updates. if not the read can be handled by other replica or the read can wait. 
- one issue that might arise is cross-device read-after-write consistency. in this case we may need to, 
    * the timestamp data of the user's last update need to be centralized. 
    * if replica's are distributed accross different datacenters, because if we take the approach of reading from the leader, we may first need to route requests from all of a user's devices to the same datacenter. 

### monotonic reads: 
- another example of an anomaly that can occur when reading from asynchronous followers is that it's possible for a user to see things moving backward in time. 
- monotonic reads means that if one user makes several reads in sequence, they will not read older data after having previously read newer data. 
- can be achieved by making sure each user reads from same replica. (the replica can be chosen by hash of userID). 

### consistent prefix reads: 
- another anamoly with replication lag is violation of causality. consistency prefix reads guarantee says that if a sequence of writes happen in a certain order, then anyone reading those writes will see them appear in the same order. 
- one solution is to make sure that any writes that are causally related to each other are written to the same partition. 

## multi-leader replication: 
- leader-based replication has one major downside: there is only one leader, and all writes must go through it. if we can't connect to it, we can't write data. 
- in multi-leader configuration, we allow more than one node to accept writes. each node that processes a write must forward that data change to all the other nodes. 
- multi-leader configuration is reasonable to use with multi-datacenter setup. 
    * Performance: every write can be processed in a local datacenter and is replicated asynchronously to the other datacenter. 
    * Tolerance of datacenter outage: with single-leader configuration, a failover promotes a follower from other datacenter as leader. in multi-leader configuration, each datacenter can operate independently. 
    * Tolerance of n/w problems.
- some db implement multi-leader by default, but is often implemented with external tools, such as tungsten replicator for mysql, bdr for postgress, goldengate for oracle. 
- a big downside of multi-leader replication is the same data may be concurrently modified in two different datacenters, and those write conflicts must be resolved. autoincrementing keys, triggers and integrity constraints can be problematic. hence multi-leader replication should be avoided if possible. 

### clients with offline operation. 
- consider the calender app in our mobile, laptop and other devices, each device has a local db that acts as a leader and there is an asynchronous multi-leader replication process between the replicas of our calendar on all of our devices. CouchDB is designed for this mode of operation. 

### collaborative editing:
- real-time collaborative editing allows several people to edit a doc simultaneously. like google-docs. for faster collaboration, we may want to make the unit of change very small(a single keystroke) and avoid locking. this approach allows multiple users to edit simultaneously, but it also brings all the challenges of multi-leader replication, including requiring conflict resolution. 

### Handling Write Conflicts: 
- biggest problem with multi-leader replication is write conflict. 
- if we want synchronous conflict resolution, we might as well just use single-leader replication. 
- simple strategy for dealing with conflicts is to avoid them, if the application can ensure that all writes for a particular record go through the same leader, the conflicts cannot occur. 

### multi-leader replication topology:
- a replication topology describes the communication paths along which writes are propagated from one node to another. 
- the most general topology is all-to-all, in which every other leader. 
- mysql supports circular topology, in which each node receives writes from one node and forwards those writes to one other node. 
- in star topology, the root node forwards writes to all of the other nodes. 
- problem with circular and star topologies is that if just one node fails, it can interrupt the flow of replication messages between other nodes, causing them to be unable to communicate until the node is fixed. the topology could be reconfigured to work around the failure node, but in most deployments this has to be done manually. 
- the fault tolerance of all-to-all topology is better as it allows message to travel along different paths, avoiding single point of failure. 
- all-to-all topology has problems too: some network links may be faster than others, which causes some replication messages to overtake others. 


# leader-less replication:
- it became fashionable architecture for databases after amazon used it for its in-house dynamo system. Riak, cassandra and voldemort use leaderless replication model. 
- in leaderless implementation, the client writes directly to several replicas, while in others, a coordinator node does this on behave of the client. 
- in leaderless configuration failover does not exist. client sends the write to all replicas, available replica accepts the write and unavailable replicas miss it. if required number of replicas ack the write, then the write is considered success. 
- when a client reads from the database, it doesn't just send its request to one replica: read requests are sent to several nodes in parallel. the client may get up-to-date response from one node and stale response from other node. version numbers are used to determine which is newer. 

## read-repair and anti-entropy:
- when unavailable node comes up, catchup in done via two mechanisms,
    * Read repair: when client makes a read from several nodes, it detects any stale response and writes the newer value back to that replica. 
    * anti-entropy: some datastores run background process to sync data between replicas. Unlike the replication log in leader-based replication, it does not copy writes in any particular order and may have significant delay before data is copied. 
- without anti-entropy, values that are rarely read may be missing from some replicas. 

## quorums for reading and writing:
- if there are n replicas, every write must be confirmed by w nodes to be considered successful, and we must query at least r nodes for each read. As long as r + w > n, we except to get an up-to-date value when reading, because at least one of the r nodes we're reading from must be up to date. Reads and writes that obey these r and w values are called quorum reads and writes. 
- a common choice is to make n an odd number and to set w = r = (n + 1) / 2. However we can vary for our needs, a workload with fewer writes and many reads may benefit from setting w = n and r = 1. This makes reads faster, but has the disadvantage that just one failed node causes all database writes to fail. 
- reads and writes are always send to all n replicas in parallel. the parameters w and r determine how many nodes we wait for ack before we consider the read or write to be successful. 

## limitations for quorum consistency:
- quorums are not necessarily majorities. it only matters that the set of nodes used by the read and write operations overlap in at lead one node. 
- with smaller w and r we are more likely to read stale values. On the upside, this configuration allows lower latency and higher availability.
- however even with w + r > n, there are some edge cases, 
    * if a sloppy quorum is used, the w writes may end up on different nodes than the r reads, so there is no guaranteed overlap between the r and w nodes. 
    * if two writes occur concurrently, it is not clear which one happened first. 
    * if a write happens concurrently with a read, the write may be reflected on only some of the replicas. 
    * if a write succeeded on fewer than w replicas, it is not rolled back on the replicas where it succeeded. for a failed write, subsequent reads may or may not return the value from that write. 
    * if a node carrying a new value fails, and its data is restored from a replica carrying an old value, the number of replicas storing the new value may fall below w. 
- dynamo-style databases are generally optimized for usecases that can tolerate eventual consistency. 
- the parameter w and r allows you to adjust the probability of stale values being read, but not as absolute guarantee. 

## sloppy quorum and hinted handoff:
- in large cluster with more than n nodes, request can go to nodes different from the n nodes the value exists. In that cause the client can either return error or accept the write and write them to available nodes. Second case is sloppy quorum, write and reads still require w and r successful responses, but those may include nodes that are not among the designated n 'home' nodes for a value. 
- once the network interruption is fixed, any writes that temporarily accepted on one node on behalf of the other node are sent to the appropriate "home" nodes. this is called hinted handoff. 
- sloppy quorums are useful for increasing write availability. 

## multi-datacenter operation:
- the number of replicas n includes nodes in all datacenters, and in the configuration we can specify how many of the n replicas we want to have in each datacenter. each write from a client is sent to all replicas, regardless of datacenter, but the client usually only waits for acknowledgement from a quorum of nodes within its local datacenter so that it is unaffected by delay and interruptions on the cross-datacenter link. the higher-latency writes to other datacenters are often configured to happen asynchronously. 

# detecting concurrent writes: 
## last write wins: 
- even through the writes don't have a natural ordering, we can force an arbitrary order on them. we can attach a timestamp to each write, pick the biggest timestamp as the most recent, and discard any writes with an earlier timestamp. this conflict resolution algorithm is the only supported one in cassandra. 
- lww achieves the goal of eventual convergence, but at the cost of durability: if there are several concurrent writes to the same key, even if they were all reported as successful to the client, only one of the writes will survive and the others will be silently discarded. 
- there are some situations, such as caching, in which lost writes are perhaps acceptable. if losing data is not acceptable, lww is a poor choice for conflict resolution. 

- we can simply say that two operations are concurrent if neither happens before the other(i.e., neither knows about each other).

