# Distributed Systems
Note taken from [MIT: Distributed Systems](https://www.youtube.com/channel/UC_7WrbZTCODu1o_kfUMq88g)

### Why choose distributed systems
* Parallelism: parallel computation, handle large requests and respond in a short period of time
* Fault tolerance: availability and recoverability. keep a server available despite of any kind of failure. For example a single computer goes down for whatever reasons (such as outage, flooding), other servers can still respond.
* Security / Isolated: data is not stored in a single server. Independent failure

### Problems
* Concurrency: 
* Partial failure
* Performance: scalability

### Infrastructure
* Storage
* Computation
* Communication

### Replication
#### General methods
* State Transfer: the primary sends a copy of its entire state such as the contents of its RAM to backup and backup stores the latest states. But it requires lots of memory and it is slow

* Replicated State Machine: state machines are deterministic because they just follow some instructions. However, when some external events come, something unexpected would happen, so this method does not send the state to backup but it sends external events instead

#### State Machine
* execute a sequence of commands
* transform its state and may product some outputs. These commands are deterministric and output of the state machine are solely determined by the initial state and by the sequence of commands that it has executed
* independent of time and other activity in a system

#### Replicated State Machine
* replication log ensures state machines execute same commands in same order
* consensus module guarantees agreement on command sequence in the replicated log
* system makes progress as long as any majority of servers are up

#### Where to implement replica coordination
* implement RC at a virtual machine running on the same instruction-set as underlying hardware

#### Steps
* clients send network packet to primary --> interrupt VMM --> VMM simulates a network packet arrival interrupt into the primary operating system --> VMM also sends to the network a copy of that packet to backup
* both of them generate output but the output from backup is ignored by VMM because it knows it is only a backup
* if backup does not receive anything from primary for some time, then it assumes primary is dead and it goes lov. The VMM there would stop ignoring the output from the "new" primary

#### How to keep backup have exactly same replica
* a time on a physical machine that is running in the primary would tick and deliver the interrupt to the primary. At a moment, the primary would stop execution and write down the instruction number and it sends that instruction number to backup

#### Output rule
* send the output to the client only after receiving the acknowledge from the backup. Therefore, if the client sees the reply it means the backup must see it as well or at least buffer it. It causes some delay that is why we don't replicate everything but only on application level replication. 

#### Which one should be primary
* TEST-AND-SET server that decides it when each of them thinks the other one is dead

### Aurora
Modern distributed cloud services tend to decouple compute from storage and have replicas across multiple nodes. One problem of having replicas across different regions or even countries is the delay which due to the network capacity. Therefore, Aurora reduces the size of network packet and it is able to have more replicas. 

#### Goals
* lose an entire AZ and one additional node without losing data
* lose an entire AZ without impacting the ability to write data

#### Keys to succeed
* only send log entry to backup servers
* use quorum-based voting protocol

#### quorum
* each read must be aware of the most recent write, Vr + Vw > V. This ensures that the read and write have an intersects with the set of the nodes. This means at least one of the node would contain the latest version.
* each write must be aware of the most recent write to avoid conflicting writes, Vw > V / 2

In this case, Aurora has 6 replicas, and it has to write to 4 nodes and read from 3 nodes. To reduce the time of Mean Time to Repair, Aurora partitins the database volume into small fixed size segments, currently 10GB in size.

### MapReduce
* run Map function on each of input file
* generate value-Key pairs
* reduce

#### Jobs
Job contains a lot of tasks (map/reduce)
example: word counts
```C++
map(k, v)
	split v in to words
	for each word w:	
		emit(w, “1”)

reduce(k, v) (vector)
	emit(len(v))
```  

Emit would store data in local disk and Reduce emit would write files output to cluster. Inputs and outputs all live in a Network File System such as GFS. Then it read data in parallel and compute at the same time. 

When input 1 comes, MapReduce process has to go off and talk across the network to the correct GFS server or maybe servers that store parts of the inputs and fetch them over the network to the MapReduce. This involves lots of network communication and network throughput becomes the bottleneck. One possible solution: Run GFS server and MapReduce workers on the same machine, GFS server knows which server has the input files and sends the map to that server to compute, then it fetch data from local disks without sending stuff through network. But currently Google does not use it because they assume that networking should be very fast

#### Shuffle
* collect all same keys
* which also means transformation from row stores to column stores
* moving every piece of data across the network
* movement of the keys to Reduce requires network communication again —> expensive

### Threads

#### Why threads
* I/O concurrency: one server waiting for reading from disk, other server can do computation
* Multi-core parallelism 
* Convenience: master checks slave is alive or not, check periodically

#### Why not Event Driven Programming
* get I/O concurrency but does not get parallelism

#### Difference between thread and process
* threads sit in stack, normally the size is serveral KB which is not large
* process can have multiple threads running inside
* each process has its own memory

#### Coordination
* Channels
* Condition variables
* Wait group: just count some numbers, it is waiting for a specific number and then finish the process. (Wait for the children to finish and then close it)

### GFS

#### Challenges in Distributed Systems 
Distributed Systems are designed to shard to different servers in order to get performance, but this also increases faults. The following loop is a challenge in this topic
* Performance —> sharding
* Fault —> tolerance
* Tolerance —> replication
* Replication —> inconsistency
* Consistency —> low performance

#### Features of GFS
* Big, fast
* Global
* Sharding
* Automatic recovery
* Single data centre
* Internal use
* Big sequential access (GB, TB)

#### Master
Master knows which server holds the chunk. Data is divided into small chunks, normally 64 MB

#### Master Data
* File name  —> array of chunk handle (non-volatile nv)
* Handle —> list of chunk servers (the server that holds replicas) (v), including version # (nv), which one is primary (v), lease expiration (v)
* Log, checkpoint stored in disk and remote machines

#### Read operation
* Name, offset —> master
* Master sends Handler, list of Server, cached
* Chunk —> chunk server
* Return data to the client

#### Write operation
Client can read from any chunk server that contains the new data but what about write? It has to have a primary. What if No primary?
* Find up to date replicas (the version number is same as the one that master knows), if no chunk server has the latest version then master would wait maybe forever
* Pick primary, secondary
* Increase version number -> only when a new primary is designated
* Tell primary, secondary, version number —> lease
* Mater writes version number to disk

#### Separate data flow from control flow
To fully utilize network bandwidth, data is pushed linearly aong a chain of chunkservers rather than distributed in some other topology
* Client asks Master about primary and location of all replicas. If no primary, then Master grants one to be primary
* Client caches this information for future use
* Client sends data to all replcias by carefully pick a chain of chunk servers, normally closest first
* Client recieves acknowledge from all replcias, then sends write requests to primary
* Primary forwards write requests to secondary
* Primay replies to Client

Primary picks offset, then all replicas are told to write at offset. If all of them reply "yes", then primary return "success" to clients. Otherwise, primary replies "no" to client. In this case, some clients may or may not append the data successfully so the replicas might have different data now. (Maybe network issue, jut does not get the message back)

--> how to fix this problem: design the system that all backup keeps data in sync

#### Split Brain problem
It is causes by network partition and this is the hardest problem. Master pings Primary periodically and if it does not receive responses it might be because of networking issue with lost packet. Master would designate a new primary which would result in "split brain" problem.

#### How to solve Split Brain problem
When master designates a primary it gives it a lease. Master and primary know how long the lease would last. If lease expires, primary knows it is expired, then it would just stop executing and ignore/reject client requests. Master has to wait for previous primary to expire and then designate to a new one. But why not designate to a new primary immediately after Master receives no reponse from Primary. Because client caches the identity of primary for at least a short period of time. The time when master told a client that A is primary, it designated that B is primary. Then client received wrong information

#### Upgrade GFS to a strong consistent system
* Reduce same requests by duplicate detection
* Secondary has to execute what primary tells it to do. If secondary does have some permanent problems then primary takes it out of the system and replaces it
* Multiple phrases in the write. Only append data until master tell them to do so 
* Primary crashed before share to all clients, but did share to some of the secondaries. New primary must know how to re-synchronizing all the secondaries to keep consistency
* In situation like secondaries have different data or client has a stale indication from the master of which secondary to talk to, the system either sends all client reads through all the primary

### RAFT
One of its features is Majority Vote and elect a leader. If there (2f + 1) servers, then it allows f failures. Any majority  has to be overlapped with previous leader majority, so at least one server has previous term and current term number

#### Process
* Client sends put requests to application layer in Leader
* Leader sends it to RAFT layer and says please commit to application log and tells me when it is done. 
* Raft under Leader server would send Append Entry request to other replicas
* Each RAFT would send request to local server to execute the request too
* RAFT sends acknowledge back to leader and the Leader executes the requests
* Leader sends request back to client
* When leader realize change is committed, it would send message to other replicas saying that it already committed. Then replicas would execute the operation and apply it to their state

No Client is waiting from replicas execution state, so the time does not matter that much. But leader would keep sending Append Entry to all replicas until they all commit. In the long run, RAFT would make sure that the logs in each replicas are identical. If replicas can not handle requests as quickly as leader, then it would build up uncommitted entries and finally lead to out of memory and fail. So replicas also need to communicate to leader about the state that it is in to avoid having a long backlog

```Java
Application layer: 
start(command) (index, term)

RAFT
Return upwards: applyCh, ApplyMsg(command, index)
```

#### Leader Election
why RAFT has a strong leader? To build a more efficient system with a leader. Followers do not need to know about the number of the leader. The term number is enough. Each term, there is at most one leader. For election timer, if it expires, start the election. If no response from leader, then assume it is dead. Then `TERM++` and then request votes by sending RPC to other servers. Usually it takes a few seconds to complete a leader election. 

If a server is voted for a leader, it has to send out append entry to all the servers and let them know because no server is allowed to send append entry unless you are the leader for the term. Then resetting everyone’s election timer. Time is randomly selected in each server. 

Minimum heartbeat: new random number of timer, not the same as previous one
Maximum heartbeat: can decide how much time it takes for the system to recover

#### How the newly selected leader restore the consistent state in the system? How does it figure out the log?
What can the log look like after crashes
```
   1 2 3 4
S1 3
S2 3 3 4
S3 3 3 5 6
```

Slot 1 and 2 we can keep because it is from majority of vote, they are committed. But we have to drop slot 4 and 5 to keep the replicas consistent in content. For slot 2, raft can not disapprove that it is committed, then raft can just keep it.  Because it might happen when the leader sent out the requests and crash, so it does not execute the append entry, neither for other replicas.

S3 is selected as a new leader for term 6. Then prevLogIndex = 3, prevLogTerm = 5. nextIndex[s2] = 3, nexIndex[s1] = 3
`New Leader initializes all nextIndex values to the index just after the last one in its log. If a follower's log is inconsistent with the leader's, the AE consistency check will fail in the next AE RPC. After a rejection, the leader decrements nextIndex and retries the AE RPC. Eventually nextIndex will reach a point where the leader and follower logs match`

Why it is ok to drop log for term 4, because there is no reason to believe that it is executed or even received by any server. It is never executed. 

#### Why not longest log as leader
If you vote a leader, you are supposed to record the term in persistent storage 
```
S1 5 6 7
S2 5 8
S3 5 8
```
Term 8 is likely to be committed, if we choose s1 to be the leader, 8 would be dropped

#### Election restriction
Vote `yes` only if candidate has a higher term in last entry, or same last term, >= log len 

#### Append entry reply
* XTerm: term of conflicting entry
* XIndex: index of the first entry with XTerm
* XLen: length of log

#### Others
Persistent: writing to the disk. Writing any file to the disk takes about 10ms which is very expensive
Log: only record of the application state , so to reconstruct the application state after reboot
CurrentTerm: 
VotedFor

#### Synchronous disk update
* write(fd, —): you don’t know if it writes successfully to the disk or not
* fsync(fd): only by calling this you know that, but that is expensive

#### Log compaction and snapshots
* Application state might be smaller than the log entries
* Snapshot is just a table + log index —> can remove the log entry before that log index then

#### Linearizability 
Execution history is linearizable if exists order of the operations in the history that matches real-time for non-concurrent requests. Each read sees most recent write in the order

### Zookeeper
* Api general purpose coordination service
* Nx —> Nx performance
* Mini-transaction

#### Leader
There is a zab layer inside each server which is very similar to RAFT. However, in RAFT, leader is responsible for handling all the requests from clients and forward them to all other followers. In this case, leader becomes the bottleneck when increasing the number of servers. Therefore more servers means it is slower

A solution is to send requests to leader and read requests to replicas servers. But is data up-to-date? The server handling read requests might not be in the majority, it might not see the write requests. Even though it sees it,  it might miss the commit command. Therefore we can not play this game! linearizability forbids us from doing this

How zookeeper does this? Zookeeper doesn’t guarantee consistency. It does not say I am going to return the up-to-date data to you 

#### Zookpper Guarantees
* linearizable writes
* FIFO client oder
    * writes — client specified order
    * reads — when replicas append entries, leader would send them zxid as well. Client reads the data, replicas would send back the information with zxid and when it switches servers because previous one failed, then it is still able to see exactly the same position with same zxid
    
So, in Zookeeper, if I write the data and read it in order, then I am able to see the latest data but if you write the data and I read, then I might need to wait sometime so I can see the latest data. It is not guaranteed

```
Write order
	delete("ready")
	write f1
	write f2
	create("ready")
	(Read request client sent to server)
				exist("ready") —> if true
				read f1
				read f2
```
If `exist("ready")` returns true, then we know the read sort of monotonically move forward in time to the log, so read f1 can read successfully. In Zookeeper, you can set `exists(ready, watch = true)` which means if there is any change, please send me the notification. It is guaranteed to see this notification before executing the following commands 

#### Motivation (Application)
* Test-and-set server
* config info: publish configuration files for other servers to use
* Master elect
* MapReduce as well, you write your map function and other server takes it

#### Z nodes
* regular
* ephemeral
* sequential

```
# APIs
create(path, data, flags)
 (exclusive)
delete(path, Version)
exists(path, watch)
getdata(path, watch)
setdata(path, new data, version) 
list(d)
```
Even though Zookeeper does not guarantee that client sees the latest data, the APIs provided are able to make sure the data is up-to-date 

```
x = get(key)
set(key, x+1)
```

This is a problem in zookeeper because it is not atomic the get data might be stale data. Instead, we can do following

```
While true:
	x, v = getData("f")
	if setData("f", x+1, v)
		break
```
`setData` makes sure that the version it gets is exactly the same as the version known in Leader. Because write data would only go to leader and set the data only if the version `v` matches the version remember in Leader. While loop is not guaranteed to finish, but in real life since Leader would push other replicas to have same data as it so it is likely that this would finish. However, this is disastrous when there are high consistent load of writes then maybe the first one would succeeds and others fail. This is known as `Herd Effect`, second locking would fix this

#### Using locks
```
Acquire (create lock on file "f")
1. If create("f", ephemeral=T) return
2. If exists("f", watch=T) 
3.     Wait
4. Go to 1
```
What if lock is released between create and exist? That is why there is step 4 it needs to go back and check again. This would also cause Herd Effect, because only the first one would get the lock and all others are waiting for the same lock (non-scalable lock)

```
Lock without herd effect
Acquire
1. Create sequential "f" (it has a file number such as f27, ephemeral file)
2. List f*
3. If no lower number file, return
4. If exists (next lower number, watch =T)
5.    Wait
6. Go to 2
```
Since we guarantee the number of files is ascending `Never` descending, so why do we need to list f* again? Why not just -1. Because file is ephemeral, then it might die and we need to wait for previous file instead. 

Why this does not suffer from Herd Effect? Even though thousands of clients are trying to create the file, each time, only one file is waiting for a lock while in previous one, every client sends RPC requests to zookeeper. (scalable lock)

Good thing is that the next client who gets the locks need to have a way to clean up the previous stale data. For example, if previous MapReduce job didn’t finish executing a command, then the next client can re-execute it. (soft lock) This method can be used to elect a master

In go, you can not tell the order the threads execution with a flag at the end of each thread. It can only tell the order if you have before and then relationship. That means it is very hard to recover when a thread holding a lock crashed. But this scalable lock is possible to do so

### CRAQ
Basic idea is: Chain Replication —> increase read throughput while preserving linearizability

#### head and tail servers
Write requests are sent to Head and then Head passes the request down to the chain and when it reaches the tail, tail sends back response to Head. If head failed, the next node would take over as a head. If head crashes without forwarding write requests ahead and then no commits and client would not get acknowledge. Same for tail. Immediate node failed then need to take it off the chain

#### Difference between RAFT and CRAQ
* Raft Leader would handle more loads with increasing clients since it needs to send requests to all other replicas while in CRAQ, only the Head forward once to the next node
* Raft Leader sees every request from the client while CRAQ splits it by having head and tail

#### Config Manager
Does not have proof for split brain and network partition. Therefore it introduces Config Manager. Whenever there is failure in servers, it would send out new configuration about the chain, who is Head and Tail. Under Head and Tail, they might be Raft, Paxos, zookeeper which is fault tolerance and does not suffer from split brain

But chain replication is not resistant to transient slowdown for example one server is upgrading software or so it does not process the request and forward it to next server quick enough

Raft or Paxos is good when the replication might be very far away from each other, then it only needs to wait for acknowledgement from the majority instead of all of them. So it really depends on what goals are you trying to achieve

### Frangipani
* caching coherence
* distributed transactions
* distributed crash recovery

