# Distributed Systems
Note taken from [MIT: Distributed Systems](https://www.youtube.com/channel/UC_7WrbZTCODu1o_kfUMq88g)

### Why choose distributed systems
* Parallelism: parallel computation, handle large requests and respond in a short period of time
* Fault tolerance: availability and recoverability. keep a server available despite of any kind of failure. For example a single computer goes down for whatever reasons (such as outage, flooding), other servers can still respond.
* Security / Isolated: data is not stored in a single server. Independent failure

#### Problems
* Concurrency: 
* Partial failure
* Performance: scalability

#### Infrastructure
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

```
U1 WS1 (VI, CC —> frangipani)
U2 WS2
U3 WS3
```
System operations happen just locally in the ws which increases scalability. Petal (shared disk drive —> virtual disk) which  comes as pairs

#### Challenges <— caching, decentralized
* Cache coherence
* Atomicity
* Crash recovery

Cache coherence: use of locks as drive, cache the latest data

#### Rules
```
# Lock server table
file | lock owner
X         ws1
Y         ws1
```

```
# Ws frangipani table 
File | lock | content
x      busy      …
Y      idle      …
```

* No cache data without a lock that protects the data
* Acquire a lock and read from petal
* Write data to petal after getting acknowledge from petal then release the lock

#### Coherence protocol
* Request: ws —> lock server
* Grant: lock server —> ws
* Revoke: lock server —> ws (asking ws to give the lock which might be in idle state so other ws can use it). Then it would first write dirty data to Petal and then send back message to lock server saying giving up the lock
* Release: ws —> ls (but if ws is busy with renaming or other operation, then it would finish it first and then transform to idle state)

```
ws1 		ls		ws2
Acquire lock z from ls
ls grant lock z to ws1
Read file z from Petal

ws2 acquires lock z
ls sends revoke message to ws1
ws1 writes z back to Petal
ws1 sends release back to lock server
lock server sends grant to ws2
ws2 reads file z from petal
```

#### Optimization
* Lock is not released immediately after they turn into idle because it is possible that no one read from the file but instead it would write to the data frequently
* Read-only locks: invoke everyone’s lock and then you are allowed to write to the data

#### Atomic multi-step operations
* Transactions —> distributed transaction system
* Acquire all the locks for the operation
	* All updates
	* Write to patal
* Release (Purpose here is to make sure no one can read from this while ws is writing)

#### Crash recovery
Crash with lock
* Write-ahead logging (wal)
 * Per-ws logs
 * logs stored in Petal

* Log entry
 * Log sequence number: when ws crashes, it can scan through the log and if there is no increasing anymore, then it knows the highest LSN as the last log entry (Block number, Version number, Data...)
 
 
* On revoke (z)
 * Write entire log —> Petal, then 
 * Write modified blocks for lock (z), then
 * Send release
 
```
Ws1: delete(d/f) (v3)		crash
Ws2: 		create(d/f)(v4)
Ws3: 					recover
```
When ws3 recovers and replay the log for ws1, the delete might still there, and then ws3 deletes the files which is wrong because it is deleting the file that was created by ws2. Therefore can not simply replay ws1 logs. It would need to check the version number in the stored in the petal and if the log version number is larger than the one stored in petal then it just ignore it

### Distributed transaction
* Concurrency control —> serializability
* Atomic commit —> transaction
```
x=10, y=10
T1:
begin_x
	add(x, 1)
	add(y, -1)
End_x

T2:
Begin_x
	t1 = get(x)
	t3 = get(y)
	print t1, t2
end_x
```
Serializable if exists serial order of execution of transactions that yields same results

#### Concurrency control
* Pessimistic —> two-phrase locking
* Optimistic  —> directly do what you need to do and the later check if there is any conflict? If so, the abort it and retry (OCC) if conflict is not frequent, then you can try this

#### Two-phrase locking
* Acquire lock before using record
* Hold until done —> why waiting all of them to finish first? Bad for performance but good for accuracy —> in this case, it forces the serial order

```
T1
get(x)
get(y)

T2 
get(y)
get(x)
```
Dead lock. Distributed transactions vs failures —> atomicity vs failures (All-or-none)

#### Two-phrase commit
TC-Transaction Coordinator
```
	S1 X
TC
	S2 Y	

Trans ID —> TID
TC		A		B
```
TC sends get or put and other requests first. The it needs to release the lock and let the data become visible to outside world perhaps a client is waiting for reply. Then TC sends PREPARE requests to participants and waiting for yes or no response from them. If all they say yes then it sends out the commit message to each participant. Then participants send the acknowledgement. As long as a single participant says no or there is failure, then it would not commit and send the abort request. So, as long as participant sees commit / abort, it would unlock

B might crash before sending yes message back, TC still waits for it, so B is entitled to unilaterally abort the transaction itself

B crashes after sending yes message but before getting the commit message. Before it sends yes message back it mush write to disk in its log all the info required to commit the transaction. So when it recovers, it restarts and look at its log and complete the transaction and restore that state

TC crashes before sending commit message. Before sending commits it must write the log/outcome to the disk as well. When it crashes, its recovery would see the log and resend the commit messages to all participants, so participants need to be prepared to receive duplicated commit message

#### Message lost in the network
Send out another one but we don’t want to sit there and wait with lock held, then it unilaterally decide to abort the transaction

For participant times out in prepare, then it is allowed to abort and thereby release the lock. If it is waiting for commit message, but it can not abort the transaction after replying yes and just waiting for the commit message. If it times out waiting for commits, it must keep waiting maybe indefinitely. And it can not unilaterally commit the message as well —> BLOCK

After participant commits the message it would forget about it and if TC didn’t receive acknowledge from them TC would resend a new one, but participant already forgot about it, so it sends another message saying there is one unknown commitment and it must be that one

Two-phrase commit is used in sharing databases where they split up data in multiple servers. If the system does not allow multi-record transaction then it does not need this. Evil reputation: multiple rounds of message, lots of disk writes and wait for it to finish (10ms) (100 transaction/second) It is not used in separated organization because of the block issue

Similar to RAFT, but they solve very different problems. RAFT gets high availability by replicating data on multiple peers and can operate even though some of them fail. Two-phrase commit, each participant does different things and they all need to do their part, so it does not help with availability 

#### Combined system?
3 clusters
```
TC: s1, s2, s3 —> RAFT
A: s4, s5, s6 —> RAFT
B: s7, s8, s9 —> RAFT
```

### Google Spanner
* Run two-phrase commit but run it over Paxos replicated participants in order to avoid the problem of crashed coordinator blocking everyone
* Synchronized time have efficient read-only transaction

```
DC1 			DC2 			DC3
shard A			copy A			copy A 		—> this shard forms one Paxos group
shard B			copy B			copy B


ccc			ccc			ccc
```
Each paxos group has a leader. It reads from local data centre, but might read out-of-date data from some shards since Paxos depends on majority voting

#### Read/write transaction
```
BEGIN
	X=X+1
	Y=Y-1
END
```

```
DC1 			DC2 			DC3
X 			X (leader)		X  
Y(leader)		Y 			Y
```
Assume X Y in different shards. 
* Read data first, so send read request to X leader and it sets a lock on it
* Leader of that shard sends response back to client
* Client sends read request to Y and lock it
* Y send back response
* Client is going to do the write action (transaction coordinator let's say leader of Y is the coordinator for this transaction)
* Client sends writes request to X leader
* Leader sends prepare request to all replicas and get the majority message saying yes
* It sends response to coordinator
Client sends write request to Y leader and leader sends prepare message and send response to coordinator saying yes then coordinator can commit and sends out Paxos followers commit message. Then leader release the locks

Coordinator manager itself is a Paxos replicated state machine, so if Y (leader) failed, other Y replicas can take over and become a transaction manager, each of them have the logs. Huge amount of messages in this design, slow 

#### Read-only transaction
* Serializable: fix read-only transaction between read/write transaction
* External consistency

#### Why not read latest value?
```
BEGIN
	PRINT(X, Y)
END
```

```
T1: wx wy c

T2: 			wx wy c

T3		rx			ry	

```
In this case, the read is not correct

#### Snapshot isolation  —> works r/o transaction
Timestamp
	r/w: TS = commit time
	r/o: TS = start time
This is more like a multi-version database which will store the timestamp
```
T1@10: wx wy c

T2@20: 			wx wy c

T3@15:		rx=9			ry=11

```
```
Database
x@10 = 9
y@10 = 11

x@20 = 8
y@20 = 12
```
When execute `ry` we know the timestamp is `@15`, so it looks at any record with timestamp that is lower than 15, T1 —> T3 —> T2: serial order

#### Why is it ok to read stale data for y
Concurrent for T2 and T3, it is ok to have either order. But since we store so much information in the database and it would blow up storage. But it will discard old records
 

#### Time sync
What is the impact on snapshot isolation if clocks are not synced?
* R/o transaction’s TS too large
    * Correct but slow
* R/o transaction’s TS too small
    * Miss recent write, not external consistency

Broadcast time from government lab such as GPS

Clock sync
UTC —> GPS satellites —> GPS receiver in each data centre 

Each data centre there will be GPS receiver, and each server sends request to GPS receiver to get the time. We don’t know how much time it takes to propagate from GPS to data centre. Communication between master and slaves also need to be considered . Which adds uncertainty

True Time Scheme 
* TT interval = [EARLIEST, LATEST]
```
Strat rule
	TS = TT.now().latest
	r/o - assign that time to start
	r/w - assign that time to commit
Commit wait - r/w transaction
	delay TS < TS.now().earliest —> when the commit finish, it is guaranteed to be in the past
```

```
T0@1:	wx1 c

T1@10			 wx2    p 	 c [11, 20]

T2@12						Rx?
```
wx2 and p get a timestamp with Earliest and latest [1, 10], it would not commit but it would keep asking what time is it until it receives a time that is for example [11, 20] which larger and this would make sure that the commit is guaranteed to be in the past

Rx asks for a time [10, 12] which includes 11, then it would choose 12 latest as time stamp
This is how spanner enforces external consistency

### Optimistic concurrency control
FARM, OCC (fast remote memory) \
RDMA (remote direct memory access)

Concerns for Spanner: speed of light and network delays between data centres \
Concerns for FARM: CPU time on the servers because they washed away the speed of light and network delays by putting all the replicas in the same data centre

Zookeeper + configuration manager
```
	Primary1, Backup1
	Primary2, Backup2

        Ccc	   ccc
```
Not a Paxos, so only need 1 replica living not majority. Clients also act as transaction coordinator for two-phrase commit

#### Performance
* sharding
* Data fit into RAM —> read quickly but fail with power failure
* NVRAM (non-volatile RAM)
* RDMA —> read / write from the server without interrupting the server
* Kernel Bypass

Non-volatile RAM —> good for power failure but not other crashes

For battery system when it sees the main power is failed, the battery system while keeps the servers running also alert all the servers with some kind of interrupts or message. Then the software on FARM servers stop all processing for FARM and then each server copies all of its RAM to a SSD attached to that server (a couple minutes). So when the power gets back the machine can read the logs stored in RAM and recover

#### Network - traditional
```
APP					APP			
———			       		———
(Kernel)	
Socket layer		
buffering				buffer
TCP					TCP
NIC driver				driver

———
NIC			—> 		NIC	
```
Network is quite slow and it is hard to increase it to process a few more hundreds of RPC messages per second. The cable runs  like 10GB / second which is hard to write our PC software that can generate small messages of the kind that db often needs to use

#### Kernel bypass - DPDK
Application layer read/write data to NIC directly and not interrupt the kernel

RDMA
```
src					dst
———			       ———
APP					APP			
———			       ———
					one-sided
					RDMA
———
RMDA NIC	—> 		RMDA NIC	
```
Directly read / write from src to app in dst and it does not know the read or write, but NIC is going to process it. So RDMA requires Kernel Bypass \
10 millions small RMDA / second \
5 microseconds for read and write \
All these involve software in the server and help client to get read and write

#### How to lock? —> use optimistic concurrency control
Pessimistic
* Locks —> 2 pl
* Conflicts —> blocks

Optimistic
* Read without lock <— one-sided RDMA super fast because later we will validate
* Buffer writes locally 
* Commit —> validation 
* Conflicts —> abort

#### FARM API
```
 txCreate()
o = txRead(OID) —> call relevant servers
o.f += 1
txWrite(OID, o)
Ok = txCommit()
```
More like a no sql database

OID: region number, address

Server Memory Layout
Region
Object
* lock bit
* versions

### Memcached at Facebook
Hash table
When write to DB, it will delete(k) in memcache —> why not just update it?
```
# Example
C1: x = 1 —> DB

				C2: x = 2 —> DB
				C2: set(x, 2)

C1: set(x, 1)
```
If we just send set RPC then it is likely that we leave stale data in the memcache as shown in this example

#### PERFORMANCE
* Partition
    * + RAM efficient
    * - not good for hot keys
    * - Clients talk to every partition
* Replication
    * + good if the problem is hot keys
    * + Few TCP connections
    *  - less total data

FE reads from the closest memcache server in the same regions. Have clusters inside one region, so limit the expense of network communicating


#### Regional pool
For those data that are not accessed that often, they are moved to regional pool memcache instead

#### Cold start (when a new cluster is added)
No data in memcache, then FE reads data from another clusters and stores it in local memcache in order to reduce the read request to DB

#### Thundering Herd
So many FE are reading for same data from the memcache, but at the same time, there is one FE write that same key to DB and delete it from memcache, then all those FEs are going to send requests to DB 

#### LEASE
The first FE that asks for a data that is missing in the memcache, memcache would send response back saying no data and then install a lease (number) and also send this lease token back to the FE. Then other FE would get response asking them to wait. The first FE would go to DB and fetch data and update the memcache 

#### RACE
```
C1: get(k) —> miss
C1: read k from db —> V1
	C2: writes k = v2 —> DB
	C2: delete(k)
C1: set(k, v1)
```

Solve this problem by using LEASE

``` 
C1: get(k) —> miss + lease
C1: read k from db —> V1
	C2: writes k = v2 —> DB (delete(k))
	C2: delete(k)
C1: set(k, v1, lease)
```

When delete(k), it is going to invalidate the lease as well, so lease is removed. But C1 set still carries the lease and found that there is no lease for that key, so memcache is going to ignore the set request

Even though the delete comes late, set(k, v1, l) gets updated because the lease is still in the memcache, so this request is acceptable. However, when the delete comes, stale data still gets deleted

### COPS, causal consistency
Straw Man One: Client puts write requests to DC1  and DC1 responses, later it is going to send the same request to other DCs, in this case, it is Eventual Consistency, no order guaranteed though —> all DC end up with storing same value

#### How
Wall-clock: assign a timestamp when the write request comes so that all DC know if the coming request has smaller timestamp it is not going to update the value

#### Two little problems
* same timestamp: <1:02, DC1> high bits timestamp

#### LAMPORT Clocks
Tmax = highest version number seen
V# = T = max(Tmax + 1, real time)

Conflicting writes: Last-writer-wins policy

#### Straw Man two - Eventual Consistency = Barriers
sync(k, v#) —> slow
put(k, v) —> return v#

```
C1: v# = put(photo), sync(photo, v#), put(list)

C2:                                 get(list), get(photo)
```

If C2 see the list then C1 must already sync all the data before putting in in the list, so C2 is able to get the photo

Sync is going to block if the DC goes down

(Eventual consistency examples: Dynamo, Cassandra)

### COPS
```
				context
get(X) -> v2			X: V2
get(Y) -> V4			X: V2 Y: V4
put(z, -) -> V3
—> put(z, -, X:V2, Y:V4)
```

Order information

```
Dependencies
Z V3 —> X V2
Z V3 —> Y V4
```

If Z V3 is revealed that means X V2 and Y V4 must be revealed. That means when syncing other DC, they need to check if X has a higher version number than V2 and Y has a higher version number than V4 before it can update Z to V3. Otherwise, it needs to hold the update before receiving request for updating X and Y to corresponding versions
 
—> Cascading dependency waiting problem

### Certificate Transparency Equivocation

SSL, TLS, https

“Gmail.com”

Private key (gmail)

They ask Certificate Authority for a certificate and that includes “gmail.com” and the Public key for gmail

So if a hacker service hack into DNS and make users go to your server instead of gmail, your server is still can not get the certificate that gmail has

Can not have a single database to store those certificates to check if the certificate is bogus

#### Certificate Transparency
```
CT Log                           < —  cert   —  CA
(Append those certificates)			|
     |
  user   <——CERT——>	   	    gmail.com
						    monitor (ask for CT log for recent new logs)
```

* Gmail asks CA and CA returns a Cert
* CA also sends this Cert to CT Log and appends it to the log
* A user sends request to gmail.com and gmail.com sends the certificate back
* The browser asks CT Log if this certificates is valid, if yes, then user can use gmail.com
* There is a monitor attached to the web, and it asks CT Log frequently for the new logs

Append-only
No forks / no equivocation
Untrusted

#### How to do this: Markel Tree
```
(Sign Tree Head)
              h()
      H()             h()
h(c1) h(c2) h(c3) h(c4)
c1       c2      c3      c4
```
It is unique, there is no same output for different inputs in the hash

Proof of Inclusion

Fork Attack-Equivocation

