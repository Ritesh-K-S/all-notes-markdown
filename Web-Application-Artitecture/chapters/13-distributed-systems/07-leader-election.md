# Leader Election — Who's the Boss?

> **What you'll learn**: How distributed systems safely choose one node to act as the coordinator/leader, what happens when the leader fails, and the algorithms that prevent two nodes from thinking they're both the boss.

---

## Real-Life Analogy: Choosing a Team Captain

Imagine a soccer team where no coach is present. The players need to choose ONE captain who will make decisions during the game. The rules:

1. Only ONE captain at a time (two captains giving conflicting orders = chaos)
2. If the captain gets injured (crashes), a new one must be chosen quickly
3. Players can't always hear each other (network issues) — yet they must still agree
4. A player who was unconscious (crashed) and wakes up must accept the current captain

This is **leader election**: choosing exactly one coordinator from a group of equals, handling failures gracefully, and preventing "split brain" (two leaders).

---

## Core Concept Explained Step-by-Step

### Why Do We Need a Leader?

```
WITHOUT A LEADER:
═══════════════════════════════════════════════════

All nodes are equal. Everyone can accept writes.

  Client A ──write──▶ Node 1: x = "hello"
  Client B ──write──▶ Node 2: x = "world"    ← CONFLICT!
  Client C ──write──▶ Node 3: x = "foo"      ← MORE CONFLICT!

Who's right? Which value wins? Chaos! 💀

WITH A LEADER:
═══════════════════════════════════════════════════

ONE node is the leader. All writes go through it.

  Client A ──write──▶ Leader: x = "hello"  ✅ (accepted)
  Client B ──write──▶ Leader: x = "world"  ✅ (ordered after)
  Client C ──write──▶ Leader: x = "foo"    ✅ (ordered after)

Leader decides order: hello → world → foo
All nodes apply in SAME order. No conflicts! ✅
```

### What Does a Leader Do?

| Responsibility | Example |
|---------------|---------|
| Coordinate writes | Accept all writes and determine their order |
| Manage replication | Ensure followers receive updates |
| Resolve conflicts | Single point of decision eliminates conflicts |
| Act as coordinator | For 2PC, saga orchestration, task assignment |
| Send heartbeats | Tell followers "I'm still alive" |

### The Challenge

```
LEADER ELECTION REQUIREMENTS:
═══════════════════════════════════════════════════

1. SAFETY: At most ONE leader at any time
   (Two leaders = "split brain" = data corruption)

2. LIVENESS: If the leader dies, a new one is elected
   (System can't be leaderless forever)

3. STABILITY: Don't change leaders unnecessarily
   (Frequent elections = downtime + performance hit)

4. CONVERGENCE: All nodes eventually agree on who the leader is
   (Even after network partitions heal)
```

---

## How It Works Internally

### Algorithm 1: Bully Algorithm

The simplest approach — highest-ID node becomes leader:

```
BULLY ALGORITHM:
═══════════════════════════════════════════════════

Nodes: A(id=1), B(id=2), C(id=3), D(id=4), E(id=5)
Current leader: E (highest ID)

E crashes! 💀 Node B notices (missed heartbeat):

Step 1: B starts election
  B → C: "Election!" (C has higher ID)
  B → D: "Election!" (D has higher ID)
  
Step 2: Higher-ID nodes respond "I'll take over"
  C → B: "OK, I'll handle it" (suppresses B)
  D → B: "OK, I'll handle it" (suppresses B)
  
Step 3: C and D start their own elections
  C → D: "Election!" (D has higher ID)
  D → C: "OK, I'll handle it" (suppresses C)
  
  D sends election to E: no response (E is dead)
  
Step 4: D wins! (highest alive node)
  D → all: "I am the new leader!" 👑
  
Step 5: All nodes acknowledge D as leader

Time complexity: O(n²) messages in worst case
```

### Algorithm 2: Ring-Based Election

Nodes arranged in a logical ring:

```
RING ELECTION:
═══════════════════════════════════════════════════

Logical Ring:  A → B → C → D → E → A (circular)

E dies! A notices:

Step 1: A starts election message: [A]
  A → B: election[A]
  
Step 2: B adds itself: [A, B]
  B → C: election[A, B]
  
Step 3: C adds itself: [A, B, C]
  C → D: election[A, B, C]
  
Step 4: D adds itself: [A, B, C, D]
  D → E: no response (dead), skip to A
  D → A: election[A, B, C, D]
  
Step 5: A sees itself in the list → election complete!
  Highest ID in list = D → D is leader!
  A sends coordinator message around ring: "D is leader"

Messages: O(2n) — one round for election, one for announcement
```

### Algorithm 3: Raft Leader Election (Production Standard)

```
RAFT LEADER ELECTION:
═══════════════════════════════════════════════════

Nodes have three states: Follower, Candidate, Leader
Each "term" is like an election period.

STEADY STATE:
  Leader sends heartbeats every 150ms
  Followers reset their election timer on each heartbeat

ELECTION TRIGGER:
  Follower's election timer expires (no heartbeat received)
  Timer is RANDOM: 150-300ms (prevents simultaneous elections)

┌─────────────────────────────────────────────────────────────┐
│ Term 1: Node E is leader                                    │
│                                                             │
│ E: 💓───💓───💓───💀 (crashes at t=1000ms)                   │
│ A: ✓────✓────✓────⏰ timer=250ms                            │
│ B: ✓────✓────✓────⏰ timer=180ms  ← times out FIRST!       │
│ C: ✓────✓────✓────⏰ timer=220ms                            │
│ D: ✓────✓────✓────⏰ timer=290ms                            │
│                                                             │
│ Term 2: Node B becomes candidate                            │
│                                                             │
│ B → B: vote for self (1 vote)                              │
│ B → A: "Vote for me? (term 2)" → A: "Yes!" (2 votes)      │
│ B → C: "Vote for me? (term 2)" → C: "Yes!" (3 votes)      │
│ B → D: "Vote for me? (term 2)" → D: "Yes!" (4 votes)      │
│                                                             │
│ B has 4/4 votes (majority of 5 = 3 needed) → B IS LEADER!  │
│                                                             │
│ B starts sending heartbeats as new leader.                  │
└─────────────────────────────────────────────────────────────┘

SPLIT VOTE SCENARIO:
  If B and C both timeout at nearly the same time:
  B gets votes from A (2 votes)
  C gets votes from D (2 votes)
  Neither has majority!
  
  → Both wait random timeout → try again next term
  → Random timeout makes split votes VERY unlikely twice
```

### Term Numbers Prevent Stale Leaders

```
PREVENTING ZOMBIE LEADERS:
═══════════════════════════════════════════════════

Scenario: Network partition temporarily isolates old leader

  ┌───────────┐       ✂️       ┌───────────┐
  │ E (Leader)│───────X────────│ B, C, D   │
  │ Term = 1  │                │           │
  └───────────┘                └───────────┘
                                     │
                               B elected as new leader
                               Term = 2

E still thinks it's leader (Term 1)!
B is actually leader (Term 2)

When partition heals:
  E sends heartbeat: "I'm leader, Term 1"
  B responds: "I'm leader, Term 2. Your term is STALE."
  E sees Term 2 > Term 1 → E STEPS DOWN to follower

RULE: Any node seeing a higher term → immediately becomes follower.
This prevents two leaders from coexisting long-term.
```

---

## Code Examples

### Python: Leader Election with Heartbeats

```python
import time
import random
import threading
from enum import Enum
from typing import Dict, Optional

class NodeState(Enum):
    FOLLOWER = "follower"
    CANDIDATE = "candidate"
    LEADER = "leader"

class ElectionNode:
    """Simulates Raft-style leader election."""
    
    def __init__(self, node_id: str, cluster_size: int):
        self.node_id = node_id
        self.state = NodeState.FOLLOWER
        self.current_term = 0
        self.voted_for: Optional[str] = None
        self.leader_id: Optional[str] = None
        self.cluster_size = cluster_size
        
        # Timing
        self.election_timeout = self._random_timeout()
        self.last_heartbeat_time = time.time()
        self.heartbeat_interval = 0.05  # 50ms
        
        self.running = True
        self.votes_received = set()
    
    def _random_timeout(self) -> float:
        """Random election timeout between 150-300ms."""
        return random.uniform(0.15, 0.30)
    
    def receive_heartbeat(self, leader_id: str, term: int):
        """Called when receiving heartbeat from leader."""
        if term >= self.current_term:
            self.current_term = term
            self.state = NodeState.FOLLOWER
            self.leader_id = leader_id
            self.last_heartbeat_time = time.time()
            self.voted_for = None
    
    def receive_vote_request(self, candidate_id: str, term: int) -> bool:
        """Called when a candidate requests our vote."""
        if term > self.current_term:
            self.current_term = term
            self.state = NodeState.FOLLOWER
            self.voted_for = None
        
        # Grant vote if we haven't voted this term
        if term >= self.current_term and self.voted_for is None:
            self.voted_for = candidate_id
            self.last_heartbeat_time = time.time()  # Reset timer
            return True
        
        return False
    
    def receive_vote(self, voter_id: str):
        """Called when we receive a vote from another node."""
        self.votes_received.add(voter_id)
        
        majority = self.cluster_size // 2 + 1
        if len(self.votes_received) >= majority and self.state == NodeState.CANDIDATE:
            self._become_leader()
    
    def check_election_timeout(self):
        """Check if election timeout has expired (should be called periodically)."""
        if self.state == NodeState.LEADER:
            return  # Leaders don't timeout
        
        elapsed = time.time() - self.last_heartbeat_time
        if elapsed >= self.election_timeout:
            self._start_election()
    
    def _start_election(self):
        """Start a new election."""
        self.current_term += 1
        self.state = NodeState.CANDIDATE
        self.voted_for = self.node_id
        self.votes_received = {self.node_id}  # Vote for self
        self.election_timeout = self._random_timeout()
        self.last_heartbeat_time = time.time()
        
        print(f"[{self.node_id}] Starting election for term {self.current_term}")
    
    def _become_leader(self):
        """Transition to leader state."""
        self.state = NodeState.LEADER
        self.leader_id = self.node_id
        print(f"[{self.node_id}] 👑 Became LEADER (term {self.current_term}, "
              f"votes: {len(self.votes_received)}/{self.cluster_size})")


# --- Simulate a 5-node cluster ---
def simulate_election():
    nodes = {
        name: ElectionNode(name, 5) 
        for name in ["A", "B", "C", "D", "E"]
    }
    
    # Initially, no leader. Node with shortest timeout starts election.
    # Simulate: Node C has shortest timeout
    nodes["C"]._start_election()
    
    # Other nodes receive vote request and grant votes
    for name, node in nodes.items():
        if name != "C":
            granted = node.receive_vote_request("C", nodes["C"].current_term)
            if granted:
                nodes["C"].receive_vote(name)
                print(f"  [{name}] Voted for C (term {node.current_term})")
    
    print(f"\nLeader: {nodes['C'].leader_id}, State: {nodes['C'].state.value}")
    
    # Simulate leader crash and re-election
    print(f"\n💀 Leader C crashes!")
    del nodes["C"]
    
    # Node A times out first
    nodes["A"].current_term += 1
    nodes["A"]._start_election()
    
    for name, node in nodes.items():
        if name != "A":
            granted = node.receive_vote_request("A", nodes["A"].current_term)
            if granted:
                nodes["A"].receive_vote(name)
                print(f"  [{name}] Voted for A (term {node.current_term})")

simulate_election()
```

### Java: ZooKeeper-Style Leader Election

```java
import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicBoolean;

/**
 * Leader election using an approach similar to ZooKeeper's ephemeral nodes.
 * In production, use Apache Curator's LeaderLatch or LeaderSelector.
 */
public class LeaderElection {
    
    // Simulates a distributed coordination service (like ZooKeeper)
    static class CoordinationService {
        private final TreeMap<String, String> ephemeralNodes = new TreeMap<>();
        private final Object lock = new Object();
        
        /**
         * Create an ephemeral sequential node (like ZooKeeper's EPHEMERAL_SEQUENTIAL).
         * Returns the created path.
         */
        public String createEphemeralSequential(String prefix, String nodeId) {
            synchronized (lock) {
                int seq = ephemeralNodes.size();
                String path = prefix + String.format("%010d", seq);
                ephemeralNodes.put(path, nodeId);
                return path;
            }
        }
        
        /** Get all children (ephemeral nodes) sorted by sequence number. */
        public List<String> getChildren() {
            synchronized (lock) {
                return new ArrayList<>(ephemeralNodes.keySet());
            }
        }
        
        /** Remove a node (simulates node crash/disconnect). */
        public void removeNode(String path) {
            synchronized (lock) {
                ephemeralNodes.remove(path);
            }
        }
        
        /** Get the data (node ID) for a path. */
        public String getData(String path) {
            return ephemeralNodes.get(path);
        }
    }
    
    static class ElectionParticipant {
        private final String nodeId;
        private final CoordinationService zk;
        private String myPath;
        private AtomicBoolean isLeader = new AtomicBoolean(false);
        
        public ElectionParticipant(String nodeId, CoordinationService zk) {
            this.nodeId = nodeId;
            this.zk = zk;
        }
        
        public void joinElection() {
            // Create ephemeral sequential node
            myPath = zk.createEphemeralSequential("/election/node-", nodeId);
            System.out.printf("[%s] Joined election with path: %s%n", nodeId, myPath);
            
            checkLeadership();
        }
        
        public void checkLeadership() {
            List<String> children = zk.getChildren();
            
            if (children.isEmpty()) return;
            
            // Smallest sequence number = leader
            String smallest = children.get(0);
            
            if (smallest.equals(myPath)) {
                isLeader.set(true);
                System.out.printf("[%s] 👑 I am the LEADER!%n", nodeId);
            } else {
                isLeader.set(false);
                // Watch the node JUST before me (herd effect avoidance)
                int myIndex = children.indexOf(myPath);
                String watchNode = children.get(myIndex - 1);
                System.out.printf("[%s] Following. Watching: %s%n", nodeId, watchNode);
            }
        }
        
        public boolean isLeader() { return isLeader.get(); }
        public String getPath() { return myPath; }
    }
    
    public static void main(String[] args) {
        CoordinationService zk = new CoordinationService();
        
        // 5 nodes join the election
        List<ElectionParticipant> participants = new ArrayList<>();
        for (String id : List.of("Node-A", "Node-B", "Node-C", "Node-D", "Node-E")) {
            ElectionParticipant p = new ElectionParticipant(id, zk);
            p.joinElection();
            participants.add(p);
        }
        
        // Simulate leader crash
        System.out.println("\n💀 Leader crashes!");
        ElectionParticipant leader = participants.get(0);
        zk.removeNode(leader.getPath());
        participants.remove(0);
        
        // Next node becomes leader
        System.out.println("\n--- Re-election ---");
        for (ElectionParticipant p : participants) {
            p.checkLeadership();
        }
    }
}
```

---

## Infrastructure Examples

### ZooKeeper Leader Election

```
ZOOKEEPER LEADER ELECTION:
═══════════════════════════════════════════════════

ZooKeeper provides EPHEMERAL + SEQUENTIAL nodes:
- Ephemeral: node disappears when creator disconnects
- Sequential: ZK appends an incrementing number

Algorithm:
1. Each participant creates: /election/node-000000001
2. Smallest sequence number = leader
3. Others watch the node just before them
4. When leader disconnects → ephemeral node vanishes
5. Next-smallest becomes leader

   /election/
   ├── node-0000000001  (Node A) ← LEADER
   ├── node-0000000002  (Node B) ← watches 001
   ├── node-0000000003  (Node C) ← watches 002
   └── node-0000000004  (Node D) ← watches 003

If A dies:
   /election/
   ├── node-0000000002  (Node B) ← NEW LEADER!
   ├── node-0000000003  (Node C) ← watches 002
   └── node-0000000004  (Node D) ← watches 003

Why "watch previous" instead of "watch leader"?
→ Avoids "thundering herd" — only ONE node wakes up!
```

### Kubernetes Leader Election

```
KUBERNETES LEADER ELECTION (via Lease objects):
═══════════════════════════════════════════════════

Multiple replicas of a controller → only ONE should be active.

apiVersion: coordination.k8s.io/v1
kind: Lease
metadata:
  name: my-controller-leader
  namespace: default
spec:
  holderIdentity: "pod-abc-123"    # Current leader
  leaseDurationSeconds: 15          # Lease expires after 15s
  renewTime: "2024-01-15T10:30:00Z" # Last renewal

How it works:
1. All pods try to CREATE/UPDATE the Lease object
2. Only one succeeds (Kubernetes API is atomic)
3. Winner = leader, must RENEW every N seconds
4. If leader fails to renew → lease expires
5. Other pods notice → one of them claims the lease

   Pod-A: "I'm leader!" ─── renew ─── renew ─── 💀
   Pod-B: watching... watching... "Lease expired! I'll claim it!"
   Pod-B: "I'm the new leader!" ─── renew ─── renew ───
```

---

## Real-World Example

### Apache Kafka: Leader Election for Partitions

```
KAFKA PARTITION LEADERSHIP:
═══════════════════════════════════════════════════

Each Kafka partition has ONE leader broker:

Topic: "orders" with 3 partitions

  Partition 0: Leader=Broker1, Followers=[Broker2, Broker3]
  Partition 1: Leader=Broker2, Followers=[Broker1, Broker3]
  Partition 2: Leader=Broker3, Followers=[Broker1, Broker2]

All reads/writes for a partition go to its leader.
Leadership is DISTRIBUTED across brokers for load balancing.

If Broker1 dies:
  Partition 0 needs a new leader!
  
  Controller (elected via ZooKeeper/KRaft):
  → Checks ISR (In-Sync Replicas) for Partition 0
  → ISR = [Broker2, Broker3]  
  → Picks Broker2 as new leader for Partition 0
  
  New state:
  Partition 0: Leader=Broker2, Followers=[Broker3]
  Partition 1: Leader=Broker2, Followers=[Broker3]
  Partition 2: Leader=Broker3, Followers=[Broker2]

KRaft (Kafka 3.3+): Kafka uses Raft internally for metadata consensus,
replacing ZooKeeper dependency!
```

### Redis Sentinel: Leader Election for Redis

```
REDIS SENTINEL FAILOVER:
═══════════════════════════════════════════════════

3 Sentinels monitor 1 Master + 2 Replicas:

  Sentinel 1 ──monitor──▶ Master (writing)
  Sentinel 2 ──monitor──▶ Master
  Sentinel 3 ──monitor──▶ Master

Master goes down:
  Sentinel 1: "Master is unreachable!" (SDOWN)
  Sentinel 2: "I agree!" (ODOWN — objective down)
  Sentinel 3: "I agree!"
  
  Sentinels elect ONE sentinel as failover leader:
  (Raft-like voting among sentinels)
  
  Elected Sentinel:
  → Picks best replica (most data, least lag)
  → Promotes Replica 2 to Master
  → Reconfigures Replica 1 to follow new Master
  → Notifies clients of new Master address
  
  Total failover time: ~1-2 seconds
```

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Wrong | Fix |
|---------|---------------|-----|
| No fencing tokens | Old leader can still issue commands after being deposed | Use fencing tokens/epoch numbers that increase with each leader |
| Leader checks its own liveness | "I can still respond, so I'm still leader" — others may disagree | Leader validity is determined by followers/coordination service |
| Fixed election timeout | All nodes timeout simultaneously → split vote every time | Use random timeouts (Raft approach) |
| Not handling "zombie leader" | Leader isolated by partition still thinks it's leader | Term numbers + fencing tokens invalidate stale leader's actions |
| Leader does too much | Leader becomes bottleneck and single point of failure | Delegate work to followers; leader only coordinates |
| No graceful leadership transfer | Leader crash = brief unavailability | Implement voluntary leadership transfer before maintenance |

---

## When to Use / When NOT to Use

### Use Leader Election When:
- ✅ Exactly one node must coordinate writes (databases, queues)
- ✅ Distributed cron jobs (only one instance should execute)
- ✅ Resource management (one node assigns work to others)
- ✅ Consensus-based systems (Raft/Paxos require a leader)

### Do NOT Use When:
- ❌ All nodes can independently handle requests (AP systems)
- ❌ The leader would be a bottleneck (consider leaderless replication)
- ❌ Geographic distribution makes leader unreachable for many nodes
- ❌ Simple task distribution (use a queue instead)

---

## Key Takeaways

- 🔑 **Leader election** ensures exactly ONE node acts as coordinator, preventing conflicts and enabling ordered operations.
- 🔑 **The key challenge is safety**: never have two leaders simultaneously (split brain).
- 🔑 **Raft's approach** (random timeouts + term numbers) is the most widely used in production today.
- 🔑 **ZooKeeper/etcd** provide leader election as a service — don't implement from scratch.
- 🔑 **Fencing tokens** (epoch numbers) are essential to prevent zombie leaders from corrupting data.
- 🔑 **Leader failure detection** relies on heartbeat timeouts — there's always a brief period of unavailability during re-election.
- 🔑 **In Kubernetes**, leader election is built into the platform via Lease objects — making it easy for application-level coordination.

---

## What's Next?

Leaders often need to ensure exclusive access to shared resources — only one node writing at a time. This is done through **[Distributed Locking (Redlock, ZooKeeper)](./08-distributed-locking.md)** — the next chapter covers how to safely lock resources across multiple machines.
