# Distributed Consensus (Paxos, Raft)

> **What you'll learn**: How multiple machines agree on a single value even when some machines crash — the fundamental algorithms (Paxos and Raft) that power every strongly consistent distributed system on the planet.

---

## Real-Life Analogy: Electing a Class President During a Storm

Imagine 5 students need to elect a class president, but they're in different buildings during a storm. Phone lines keep dropping. Some students can't reach others. Yet somehow, they need to ALL agree on ONE president.

Rules:
- No central authority (no teacher to decide)
- Students can only communicate by phone (which might drop)
- Some students might fall asleep (crash) and wake up later
- They MUST agree on the same president (can't have two claiming the title)

**This is consensus**: getting multiple unreliable nodes to agree on a single value, even when communication and nodes can fail.

The **Raft** algorithm is like having a clear "campaign manager" process:
1. One student proposes themselves as candidate
2. They call others asking for votes
3. If they get majority votes → they're president (leader)
4. If two students campaign simultaneously → higher term number wins

---

## Core Concept Explained Step-by-Step

### What IS Consensus?

**Consensus** = N nodes must agree on the same value, even if some nodes crash.

```
THE CONSENSUS PROBLEM:
═══════════════════════════════════════════════════

5 Nodes need to agree: "What is the value of X?"

Node 1 proposes: X = "hello"
Node 2 proposes: X = "world"
Node 3: crashed 💀
Node 4: slow (hasn't responded yet)
Node 5 proposes: X = "hello"

REQUIREMENTS:
┌────────────────┬──────────────────────────────────────────┐
│ Agreement      │ All non-crashed nodes decide SAME value  │
│ Validity       │ The decided value was proposed by someone │
│ Termination    │ All non-crashed nodes eventually decide  │
│ Integrity      │ A node decides at most once              │
└────────────────┴──────────────────────────────────────────┘

Result: All agree X = "hello" (majority proposed it)
Node 3 (crashed): will learn the decision when it recovers
```

### Why Is Consensus Hard?

The **FLP Impossibility Theorem** (1985) proves:
> In an asynchronous system with even ONE possible crash, NO protocol can guarantee consensus in bounded time.

This means: consensus algorithms must use **timeouts** or **randomization** to work in practice.

```
THE IMPOSSIBILITY:
═══════════════════════════════════════════════════

Node A: "I propose value 1" ──────▶ Node B: ???
                                     │
                              Is B slow or dead?
                              We CAN'T know for sure!
                              
If we wait forever for B → no termination (stuck)
If we give up on B → B might come back and disagree

Solution: Use timeouts + majority voting
          (might take multiple rounds, but WILL terminate)
```

---

## Paxos: The Original Consensus Algorithm

Invented by **Leslie Lamport** in 1989. Correct but notoriously hard to understand and implement.

### Paxos Roles

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  PROPOSER   │     │  ACCEPTOR   │     │   LEARNER   │
│             │     │             │     │             │
│ Proposes    │     │ Votes on    │     │ Learns the  │
│ values      │     │ proposals   │     │ final value │
└─────────────┘     └─────────────┘     └─────────────┘

In practice, one node often plays all three roles.
```

### Paxos Two Phases

```
PHASE 1: PREPARE (asking permission to propose)
═══════════════════════════════════════════════════

Proposer picks a unique proposal number N.

Proposer ──── Prepare(N) ────▶ Acceptor 1
         ──── Prepare(N) ────▶ Acceptor 2
         ──── Prepare(N) ────▶ Acceptor 3

Each Acceptor:
  IF N > any previously seen proposal number:
    → Promise: "I won't accept anything < N"
    → Reply with any value I already accepted
  ELSE:
    → Reject (already promised to a higher N)

Proposer needs majority of promises to proceed.


PHASE 2: ACCEPT (proposing the actual value)
═══════════════════════════════════════════════════

IF Proposer got majority promises:
  Choose value:
    - If any acceptor already accepted a value → use THAT value
    - If no prior value → use Proposer's own value

Proposer ──── Accept(N, value) ────▶ Acceptor 1
         ──── Accept(N, value) ────▶ Acceptor 2
         ──── Accept(N, value) ────▶ Acceptor 3

Each Acceptor:
  IF N ≥ my promised number:
    → Accept! Store (N, value)
    → Notify Learners
  ELSE:
    → Reject (promised to higher N)

IF majority Accept → CONSENSUS REACHED! 🎉
```

### Paxos Example

```
EXAMPLE: 3 Acceptors, Proposer wants value "X"
═══════════════════════════════════════════════════

Step 1: Prepare(N=1)
  Proposer → Acceptor A: "Prepare(1)"  → A: "Promise! No prior value"
  Proposer → Acceptor B: "Prepare(1)"  → B: "Promise! No prior value"
  Proposer → Acceptor C: (network delay...)
  
  Majority (2/3) promised! ✅

Step 2: Accept(N=1, value="X")
  Proposer → Acceptor A: "Accept(1, X)"  → A: "Accepted!" ✅
  Proposer → Acceptor B: "Accept(1, X)"  → B: "Accepted!" ✅
  Proposer → Acceptor C: (still delayed)
  
  Majority (2/3) accepted! ✅
  
CONSENSUS: value = "X" 🎉
(When C comes back, it learns "X" from others)
```

---

## Raft: Consensus Made Understandable

Invented by **Diego Ongaro and John Ousterhout** in 2014 specifically to be understandable (Paxos is correct but confusing).

### Raft Core Ideas

```
RAFT = Leader-based consensus
═══════════════════════════════════════════════════

Key insight: ONE node is the leader at any time.
All decisions go through the leader.

Node States:
┌────────────┐    timeout    ┌────────────┐   wins election   ┌────────────┐
│  FOLLOWER  │──────────────▶│ CANDIDATE  │──────────────────▶│   LEADER   │
│            │               │            │                    │            │
│ Listens to │               │ Asks for   │                    │ Sends      │
│ leader     │◀──────────────│ votes      │◀───────────────────│ heartbeats │
└────────────┘  loses/timeout└────────────┘  discovers leader  └────────────┘
```

### Raft Leader Election

```
LEADER ELECTION:
═══════════════════════════════════════════════════

Term 1: Node A is leader, sends heartbeats every 150ms

  Node A (Leader): 💓──────💓──────💓──────💀 (crashes!)
  Node B (Follower): ✓────────✓────────✓──────... no heartbeat?
  Node C (Follower): ✓────────✓────────✓──────... no heartbeat?

Election timeout (150-300ms random) expires on Node B first:

Term 2: Node B becomes CANDIDATE
  
  Node B → Node B: "I vote for myself" (1 vote)
  Node B → Node C: "Vote for me? (Term 2)"
  Node C → Node B: "OK, you have my vote" (2 votes)
  
  Node B has majority (2/3) → BECOMES LEADER ✅
  
  Node B (Leader): 💓──────💓──────💓──────💓
  Node C (Follower): ✓────────✓────────✓────────✓
  Node A (recovering): joins as follower, catches up

WHY RANDOM TIMEOUT? 
  If all nodes timeout at same time → all become candidates → split vote
  Random timeout makes it very likely ONE node times out first
```

### Raft Log Replication

```
LOG REPLICATION (how writes are committed):
═══════════════════════════════════════════════════

Client sends write request to Leader:

Step 1: Leader appends to its log (uncommitted)
┌─────────────────────────────────────┐
│ Leader Log: [SET x=1] [SET y=2]     │ ← new entry
└─────────────────────────────────────┘

Step 2: Leader sends AppendEntries to followers
Leader ──── AppendEntries(SET y=2) ────▶ Follower B ✅
       ──── AppendEntries(SET y=2) ────▶ Follower C ✅

Step 3: Majority respond (2/3 including leader) → COMMIT!
┌─────────────────────────────────────┐
│ Leader Log: [SET x=1] [SET y=2] ✓   │ ← committed!
└─────────────────────────────────────┘

Step 4: Leader notifies followers to commit
Step 5: Leader responds to client: "Write successful!"

SAFETY GUARANTEE:
─────────────────────────────────────────────────
Once committed, the entry is PERMANENT.
Even if the leader crashes, the new leader MUST have this entry
(because majority has it, and leader needs majority votes).
```

### Raft vs Paxos Comparison

```
┌──────────────────┬────────────────────────┬────────────────────────┐
│                  │        PAXOS           │         RAFT           │
├──────────────────┼────────────────────────┼────────────────────────┤
│ Invented         │ 1989 (Lamport)         │ 2014 (Ongaro)          │
│ Approach         │ Leaderless (any node)  │ Leader-based           │
│ Understandability│ Notoriously hard       │ Designed to be simple  │
│ Correctness      │ Proven correct         │ Proven correct         │
│ Real-world use   │ Google Chubby          │ etcd, Consul, CockroachDB│
│ Leader election  │ Implicit in protocol   │ Explicit, separate phase│
│ Log ordering     │ Can have gaps          │ No gaps (strict order) │
│ Implementation   │ Very complex           │ More straightforward   │
│ Variants         │ Multi-Paxos, Fast Paxos│ Raft (one standard)    │
└──────────────────┴────────────────────────┴────────────────────────┘
```

---

## How It Works Internally

### Raft State Machine

```
REPLICATED STATE MACHINE:
═══════════════════════════════════════════════════

The goal: all nodes have identical state machines

  Client                  Consensus Module         State Machine
    │                         │                         │
    ├── command ─────────────▶│                         │
    │                         ├── replicate log ──────▶│
    │                         │   (Raft/Paxos)         │
    │                         │                         ├── apply ─▶ Result
    │◀── result ──────────────┤◀────────────────────────┤
    │                         │                         │

On EACH node:
┌─────────────────────────────────────────────────────────┐
│  Log:  [cmd1] [cmd2] [cmd3] [cmd4] [cmd5]              │
│                                     ↑                   │
│                              commitIndex                │
│                                                         │
│  State Machine: applies cmd1→cmd2→cmd3→cmd4             │
│  (all committed entries applied in order)               │
└─────────────────────────────────────────────────────────┘

All nodes apply SAME commands in SAME order → SAME state!
```

### Safety Properties of Raft

```
RAFT SAFETY GUARANTEES:
═══════════════════════════════════════════════════

1. Election Safety:
   At most ONE leader per term
   (nodes only vote once per term)

2. Leader Append-Only:
   Leader never overwrites its own log entries
   (only appends)

3. Log Matching:
   If two logs have entry with same index AND term,
   all preceding entries are identical

4. Leader Completeness:
   If entry is committed in term T,
   ALL leaders in terms > T have that entry

5. State Machine Safety:
   If a node applies entry at index I,
   no other node applies a DIFFERENT entry at I
```

---

## Code Examples

### Python: Simplified Raft Implementation

```python
import time
import random
import threading
from enum import Enum
from dataclasses import dataclass, field
from typing import List, Optional, Dict

class State(Enum):
    FOLLOWER = "follower"
    CANDIDATE = "candidate"
    LEADER = "leader"

@dataclass
class LogEntry:
    term: int
    command: str
    index: int

@dataclass
class RaftNode:
    """Simplified Raft node demonstrating leader election and log replication."""
    
    node_id: str
    state: State = State.FOLLOWER
    current_term: int = 0
    voted_for: Optional[str] = None
    log: List[LogEntry] = field(default_factory=list)
    commit_index: int = 0
    
    # Cluster info
    peers: List[str] = field(default_factory=list)
    votes_received: int = 0
    
    # Timing
    election_timeout: float = 0.0
    last_heartbeat: float = 0.0
    
    def reset_election_timeout(self):
        """Random timeout between 150-300ms (prevents split votes)."""
        self.election_timeout = random.uniform(0.15, 0.30)
        self.last_heartbeat = time.time()
    
    def start_election(self):
        """Transition to candidate and request votes."""
        self.state = State.CANDIDATE
        self.current_term += 1
        self.voted_for = self.node_id
        self.votes_received = 1  # Vote for self
        
        print(f"[{self.node_id}] Starting election for term {self.current_term}")
        
        # Request votes from peers (in real system: RPC calls)
        # Simulated: assume we get votes from majority
        return self._request_votes()
    
    def _request_votes(self) -> bool:
        """Request votes from peers. Returns True if won."""
        # In real implementation: send RequestVote RPCs to all peers
        # Here we simulate: each peer grants vote with some probability
        for peer in self.peers:
            # Simulate vote (in real: check term, log freshness)
            if random.random() > 0.3:  # 70% chance of getting vote
                self.votes_received += 1
        
        majority = (len(self.peers) + 1) // 2 + 1
        if self.votes_received >= majority:
            self.become_leader()
            return True
        else:
            self.state = State.FOLLOWER
            return False
    
    def become_leader(self):
        """Won election — become leader."""
        self.state = State.LEADER
        print(f"[{self.node_id}] 🎉 Became LEADER for term {self.current_term}")
    
    def append_entry(self, command: str) -> bool:
        """Leader: append entry and replicate to followers."""
        if self.state != State.LEADER:
            print(f"[{self.node_id}] Not leader, rejecting write")
            return False
        
        entry = LogEntry(
            term=self.current_term,
            command=command,
            index=len(self.log) + 1
        )
        self.log.append(entry)
        
        # Replicate to peers (simplified)
        acks = 1  # Self
        for peer in self.peers:
            # In real: send AppendEntries RPC
            acks += 1  # Assume success for demo
        
        majority = (len(self.peers) + 1) // 2 + 1
        if acks >= majority:
            self.commit_index = entry.index
            print(f"[{self.node_id}] Committed: {command} (index={entry.index})")
            return True
        
        return False
    
    def receive_heartbeat(self, leader_term: int, leader_id: str):
        """Follower: receive heartbeat from leader."""
        if leader_term >= self.current_term:
            self.state = State.FOLLOWER
            self.current_term = leader_term
            self.reset_election_timeout()
            # print(f"[{self.node_id}] Heartbeat from {leader_id} (term {leader_term})")


# --- Demo ---
print("=== Raft Consensus Demo ===\n")

# Create a 3-node cluster
nodes = [
    RaftNode(node_id="A", peers=["B", "C"]),
    RaftNode(node_id="B", peers=["A", "C"]),
    RaftNode(node_id="C", peers=["A", "B"]),
]

# Simulate: Node A times out first and starts election
nodes[0].start_election()

# Leader replicates entries
if nodes[0].state == State.LEADER:
    nodes[0].append_entry("SET x = 1")
    nodes[0].append_entry("SET y = 2")
    nodes[0].append_entry("DEL z")
    
    print(f"\nLeader log ({len(nodes[0].log)} entries):")
    for entry in nodes[0].log:
        print(f"  [{entry.index}] Term {entry.term}: {entry.command}")
```

### Java: Raft Log Replication

```java
import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;

/**
 * Simplified Raft consensus showing log replication.
 * In production, use etcd or Apache Ratis.
 */
public class RaftConsensus {
    
    enum NodeState { FOLLOWER, CANDIDATE, LEADER }
    
    record LogEntry(int term, String command, int index) {}
    
    static class RaftNode {
        final String id;
        final List<String> peers;
        NodeState state = NodeState.FOLLOWER;
        int currentTerm = 0;
        String votedFor = null;
        List<LogEntry> log = new CopyOnWriteArrayList<>();
        int commitIndex = 0;
        
        RaftNode(String id, List<String> peers) {
            this.id = id;
            this.peers = peers;
        }
        
        // --- Leader Election ---
        boolean startElection(Map<String, RaftNode> cluster) {
            state = NodeState.CANDIDATE;
            currentTerm++;
            votedFor = id;
            AtomicInteger votes = new AtomicInteger(1); // Self-vote
            
            System.out.printf("[%s] Election started (term %d)%n", id, currentTerm);
            
            for (String peerId : peers) {
                RaftNode peer = cluster.get(peerId);
                if (peer != null && requestVote(peer)) {
                    votes.incrementAndGet();
                }
            }
            
            int majority = (peers.size() + 1) / 2 + 1;
            if (votes.get() >= majority) {
                state = NodeState.LEADER;
                System.out.printf("[%s] 🎉 Won election! (votes: %d/%d)%n", 
                        id, votes.get(), peers.size() + 1);
                return true;
            }
            
            state = NodeState.FOLLOWER;
            return false;
        }
        
        private boolean requestVote(RaftNode peer) {
            // Simplified: peer grants vote if our term >= theirs
            // and they haven't voted for someone else this term
            if (currentTerm >= peer.currentTerm && 
                    (peer.votedFor == null || peer.votedFor.equals(id))) {
                peer.votedFor = id;
                peer.currentTerm = currentTerm;
                return true;
            }
            return false;
        }
        
        // --- Log Replication ---
        boolean replicateEntry(String command, Map<String, RaftNode> cluster) {
            if (state != NodeState.LEADER) {
                System.out.printf("[%s] Not leader! Redirecting...%n", id);
                return false;
            }
            
            LogEntry entry = new LogEntry(currentTerm, command, log.size() + 1);
            log.add(entry);
            
            int acks = 1; // Leader counts as one
            for (String peerId : peers) {
                RaftNode peer = cluster.get(peerId);
                if (peer != null && appendEntries(peer, entry)) {
                    acks++;
                }
            }
            
            int majority = (peers.size() + 1) / 2 + 1;
            if (acks >= majority) {
                commitIndex = entry.index();
                System.out.printf("[%s] COMMITTED: '%s' (index=%d, acks=%d)%n",
                        id, command, entry.index(), acks);
                return true;
            }
            
            System.out.printf("[%s] FAILED: '%s' (acks=%d, need=%d)%n",
                    id, command, acks, majority);
            return false;
        }
        
        private boolean appendEntries(RaftNode follower, LogEntry entry) {
            // Simplified: follower accepts if term is valid
            if (currentTerm >= follower.currentTerm) {
                follower.log.add(entry);
                follower.currentTerm = currentTerm;
                follower.commitIndex = entry.index();
                return true;
            }
            return false;
        }
    }
    
    public static void main(String[] args) {
        // Create 5-node cluster
        Map<String, RaftNode> cluster = new LinkedHashMap<>();
        cluster.put("A", new RaftNode("A", List.of("B", "C", "D", "E")));
        cluster.put("B", new RaftNode("B", List.of("A", "C", "D", "E")));
        cluster.put("C", new RaftNode("C", List.of("A", "B", "D", "E")));
        cluster.put("D", new RaftNode("D", List.of("A", "B", "C", "E")));
        cluster.put("E", new RaftNode("E", List.of("A", "B", "C", "D")));
        
        // Node A starts election
        RaftNode leader = cluster.get("A");
        leader.startElection(cluster);
        
        // Replicate some commands
        leader.replicateEntry("SET user:1 = Alice", cluster);
        leader.replicateEntry("SET user:2 = Bob", cluster);
        leader.replicateEntry("INCR counter", cluster);
        
        // Simulate leader crash: Node A dies, Node B starts election
        System.out.println("\n💀 Leader A crashes!");
        cluster.remove("A");
        
        RaftNode newLeader = cluster.get("B");
        newLeader.peers.remove("A");
        for (var node : cluster.values()) {
            node.votedFor = null; // Reset votes for new term
        }
        newLeader.startElection(cluster);
        newLeader.replicateEntry("SET user:3 = Carol", cluster);
    }
}
```

---

## Infrastructure Examples

### etcd: Raft in Production

```
etcd — The backbone of Kubernetes
═══════════════════════════════════════════════════

Kubernetes stores ALL cluster state in etcd:
- Pod definitions
- Service configs
- Secrets
- ConfigMaps

etcd cluster (typically 3 or 5 nodes):
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐         │
│  │  etcd-1  │◄──▶│  etcd-2  │◄──▶│  etcd-3  │         │
│  │ (Leader) │    │(Follower)│    │(Follower)│         │
│  └────┬─────┘    └──────────┘    └──────────┘         │
│       │                                                 │
│       ▼                                                 │
│  All writes go through leader                          │
│  Committed only when majority (2/3) acknowledge        │
│                                                         │
│  Can tolerate 1 node failure (majority = 2)            │
│  With 5 nodes: can tolerate 2 failures                 │
└─────────────────────────────────────────────────────────┘

Performance:
- Write latency: ~10ms (consensus round)
- Read latency: ~1ms (from leader)
- Throughput: ~10,000 writes/sec
```

### CockroachDB: Multi-Raft

```
CockroachDB uses MULTIPLE Raft groups for scalability:
═══════════════════════════════════════════════════

Problem: Single Raft group → all writes through one leader
         = bottleneck at scale

Solution: Partition data into "Ranges" (64MB each)
          Each Range has its OWN Raft group

┌─────────────────────────────────────────────────────┐
│                    CockroachDB                       │
│                                                     │
│  Range 1 (keys a-f):  Raft Group 1                 │
│    Node A* ←→ Node B ←→ Node C                    │
│                                                     │
│  Range 2 (keys g-m):  Raft Group 2                 │
│    Node B* ←→ Node C ←→ Node D                    │
│                                                     │
│  Range 3 (keys n-z):  Raft Group 3                 │
│    Node C* ←→ Node D ←→ Node A                    │
│                                                     │
│  * = Leader for that range                         │
│                                                     │
│  Leadership is DISTRIBUTED across nodes!            │
│  Writes to different ranges hit different leaders   │
│  → Parallelism! → Scale!                           │
└─────────────────────────────────────────────────────┘
```

---

## Real-World Example

### Google Chubby (Paxos)

Google's distributed lock service, used by virtually every Google service:

```
Chubby: Paxos-based lock service
═══════════════════════════════════════════════════

Used by:
- Google File System (GFS) — master election
- Bigtable — tablet server assignment
- MapReduce — task coordination

Architecture:
  5 Chubby replicas (1 master, 4 replicas)
  Paxos ensures master election is safe
  All writes go through master
  
  Client → Chubby Master → Paxos → Replicas
  
Google learned: Paxos is correct but INCREDIBLY hard
to implement correctly. Took years of debugging.
```

### HashiCorp Consul (Raft)

```
Consul: Raft-based service discovery
═══════════════════════════════════════════════════

3-5 Consul server nodes form a Raft cluster:

  ┌─────────────────────────────────────────┐
  │         Consul Raft Cluster             │
  │                                         │
  │  Server 1* ←→ Server 2 ←→ Server 3    │
  │  (Leader)                               │
  └───────────────┬─────────────────────────┘
                  │
    ┌─────────────┼─────────────┐
    ▼             ▼             ▼
┌────────┐  ┌────────┐  ┌────────┐
│Agent A │  │Agent B │  │Agent C │
│(client)│  │(client)│  │(client)│
└────────┘  └────────┘  └────────┘

Service registration → goes through Raft leader
Service discovery → can read from any server
Health checks → clients report to local agent
```

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Wrong | Fix |
|---------|---------------|-----|
| "Let's implement Paxos from scratch" | It takes experts years to get right. Google's Chubby team found bugs after months of testing. | Use battle-tested implementations (etcd, Consul, ZooKeeper). |
| Using even number of nodes | 4 nodes tolerates 1 failure (same as 3!) but has more overhead. | Always use odd numbers: 3, 5, or 7 nodes. |
| Too many consensus nodes | More nodes = slower consensus (more messages). | 5 is typical. 7 max. Never more. |
| Consensus for everything | Consensus is expensive (~10ms+ per decision). | Only use for coordination. NOT for application data writes. |
| Ignoring disk fsync | If log isn't fsynced before responding, crash can lose committed entries. | Always fsync log entries before acknowledging. |
| Not handling split-brain | Two leaders in same term = data corruption. | Raft's term numbers prevent this (but verify your implementation). |

---

## When to Use / When NOT to Use

### Use Consensus (Raft/Paxos) When:
- ✅ Leader election (who's the primary?)
- ✅ Distributed locking (only one writer at a time)
- ✅ Configuration management (all nodes must agree on config)
- ✅ Service discovery (consistent view of cluster membership)
- ✅ Metadata management (database master assignment)

### Do NOT Use Consensus When:
- ❌ High-throughput data writes (too slow — use sharding + local writes)
- ❌ User-facing request path (adds 10ms+ latency)
- ❌ Systems that need availability over consistency (use AP databases instead)
- ❌ Large clusters (consensus doesn't scale beyond ~7 nodes for one group)
- ❌ Cross-region coordination (latency makes consensus very slow)

### Scaling Consensus:
```
Problem: Single Raft group = ~10K writes/sec max

Solutions:
1. Multi-Raft (CockroachDB): Multiple groups, each handles a data range
2. Parallel Paxos (Spanner): Multiple Paxos groups for different data
3. Batching: Group multiple writes into one consensus round
```

---

## Key Takeaways

- 🔑 **Consensus** = getting N nodes to agree on a value, even when some crash. It's the foundation of strong consistency.
- 🔑 **Paxos** is the original algorithm (correct but complex). **Raft** is the understandable alternative (same guarantees, easier to implement).
- 🔑 **Raft uses a leader** — all writes go through one node, which replicates to followers. Leader elected by majority vote.
- 🔑 **Majority quorum** (N/2 + 1) is the key: a 5-node cluster tolerates 2 failures, a 3-node cluster tolerates 1.
- 🔑 **Always use odd numbers** of nodes (3, 5, 7). Even numbers waste resources without improving fault tolerance.
- 🔑 **Don't implement consensus yourself** — use etcd, Consul, or ZooKeeper. Even Google spent years debugging Paxos.
- 🔑 **Consensus is expensive** (~10ms per decision). Only use it for coordination, not every data write.

---

## What's Next?

Consensus helps nodes agree, but what about transactions that span multiple services? The next chapter covers **[Distributed Transactions (2PC, Saga Pattern)](./06-distributed-transactions.md)** — how to maintain data consistency across different databases and microservices.
