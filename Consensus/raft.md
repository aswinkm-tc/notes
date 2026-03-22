# Raft
Raft is a consensus algorithm designed to be easy to understand. It ensures that a cluster of machines can agree on a series of values (a log) even if some of the machines fail.

It operates on a "Leader-Follower" model where one node takes the reins, and the others replicate its state.

## High-Level Architecture
The system is built on a Replicated State Machine (RSM) architecture. The goal is to keep the State Machine (your database or service) identical across all nodes.
```mermaid
graph TD
    Client((Client)) -->|Command| Leader[Leader Node]
    Leader -->|1. Append Entries| F1[Follower 1]
    Leader -->|1. Append Entries| F2[Follower 2]
    F1 -->|2. Acknowledge| Leader
    F2 -->|2. Acknowledge| Leader
    Leader -->|3. Commit & Apply| SM[State Machine]
    Leader -->|4. Response| Client
```

## The Node State Machine
Every node in the cluster exists in one of three states. Most of the time, there is one Leader and everyone else is a Follower.
```mermaid
stateDiagram-v2
    [*] --> Follower
    Follower --> Candidate : Times out, starts election
    Candidate --> Leader : Receives majority of votes
    Candidate --> Follower : Discovers newer term or leader
    Leader --> Follower : Discovers newer term
```
**Follower**: Passive; only responds to requests from Leaders/Candidates.

**Candidate**: Active; used to elect a new leader.

**Leader**: Handles all client requests and manages log replication.

## Leader Election (The Heartbeat)
Raft uses Terms (logical clocks) to detect stale information.
* Timeout: If a Follower doesn't hear from a Leader for a random period (e.g., 150ms–300ms), it becomes a Candidate.
* Request Vote: The Candidate increments its Term and asks other nodes for votes.
* Majority: If it gets votes from a majority ($N/2 + 1$), it becomes the Leader.
* Heartbeats: The new Leader immediately sends "empty" AppendEntries calls to prevent others from timing out.

## Log Replication (The Workhorse)
Once a leader is elected, it manages the log. This is how the "Consensus" actually happens.

* **Client Request**: A client sends a command (e.g., SET X=10) to the Leader.

* **Local Append**: The Leader adds the command to its own log.

* **Replication**: The Leader sends AppendEntries RPCs to all Followers.

* **Commitment**: Once the Leader receives a "success" from a majority of Followers, it considers the entry committed.

* **Application**: The Leader applies the entry to its local State Machine and notifies Followers to do the same in the next heartbeat.

## Safety & Consistency
Raft ensures that if any node has applied a log entry to its state machine, no other node can ever apply a different command for that same index.

**Election Restriction**: A Candidate cannot win an election unless its log is "at least as up-to-date" as a majority of the cluster. This prevents a node that was offline from becoming leader and wiping out valid history.

**Log Matching Property**: If two logs contain an entry with the same index and term, then the logs are identical through that index.

## Summary
| Phase | Action |
|---|---|
| Normal | OpLeader receives client request $\rightarrow$ replicates to followers $\rightarrow$ commits $\rightarrow$ responds. |
| Failure | Leader dies $\rightarrow$ Followers time out $\rightarrow$ Election starts $\rightarrow$ New Leader emerges. |
| Recovery |Old Leader comes back $\rightarrow$ Sees higher Term $\rightarrow$ Steps down to Follower $\rightarrow$ Syncs log. |

# Sequence Diagram
```mermaid
sequenceDiagram
    autonumber
    actor Client
    participant Leader
    participant Follower 1
    participant Follower 2

    Note over Leader, Follower 2: Cluster is stable. Current Term: 2.

    %% 1. Client Request
    Client->>+Leader: Send Command: "SET X=10"
    
    %% 2. Leader Appends Locally
    Note right of Leader: Uncommitted entry (Term 2, Index 101)<br/>appended to local log.
    
    %% 3. Leader Replicates (First Majority)
    par Replicate to Follower 1
        Leader->>+Follower 1: AppendEntries RPC (Index 101, Cmd: "X=10", LeaderCommit=100)
        Note right of Follower 1: Appends entry to local log.
        Follower 1-->>-Leader: AppendEntries Response: SUCCESS
    and Replicate to Follower 2 (e.g., this message is delayed)
        Leader->>+Follower 2: AppendEntries RPC (Index 101, Cmd: "X=10", LeaderCommit=100)
        Note right of Follower 2: Message delayed by network...
    end

    %% 4. Leader Commits
    Note over Leader: RECEIVED SUCCESS FROM<br/>MAJORITY (Self + F1).<br/>ENTRY 101 COMMITTED.
    Note right of Leader: Apply "X=10" to State Machine (DB).

    %% 5. Respond to Client
    Leader-->>-Client: Success (Result: 10)

    %% 6. Update Followers (The next Heartbeat or AppendEntries)
    Note over Leader: Background: Send heartbeat to finalize commitment on Followers.
    par Update F1 Commitment
        Leader->>+Follower 1: AppendEntries RPC (Empty/Heartbeat, LeaderCommit=101)
        Note right of Follower 1: Entry 101 < LeaderCommit.<br/>Apply "X=10" to local State Machine.
        Follower 1-->>-Leader: AppendEntries Response: SUCCESS
    and Deliver delayed message to F2
        Note right of Follower 2: ...Delayed message arrives.
        Note right of Follower 2: Appends entry 101.
        Follower 2-->>-Leader: AppendEntries Response: SUCCESS
    end
```
Steps 2 & 3: Parallelism. The leader is responsible for notifying all followers in parallel. It does not wait for all of them; it only waits for a majority.

Step 4: Commitment Point. This is the moment consensus is achieved. The "Commit Index" advances on the Leader. Once committed, this entry is durable and cannot be lost.

Step 5: Rapid Response. The leader responds to the client immediately after the majority responds, before the remaining followers even know the entry is committed (as seen in Step 6, F2 is still catching up).

Step 6: Delayed Application. The next AppendEntries (which might be an empty heartbeat) is used to inform the Followers of the new Commit Index, allowing them to finally apply the change to their own state machines.