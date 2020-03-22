# Distributed Systems

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

#### Replicated State Machine
* replication log ensures state machines execute same commands in same order
* consensus module guarantees agreement on command sequence in the replicated log
* system makes progress as long as any majority of servers are up














