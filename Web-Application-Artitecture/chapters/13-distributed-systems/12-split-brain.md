# Split Brain Problem & How to Handle It

> **What you'll learn**: What happens when a distributed cluster gets partitioned into isolated groups — why both sides might think they're the "real" cluster, and the battle-tested techniques to prevent catastrophic data divergence.

---

## Real-Life Analogy: Two Generals Running One Army

Imagine a general commanding 1,000 soldiers. The general is connected to all troops by radio. One day, the radio tower in the middle breaks — now there are two groups: 600 soldiers on the north side and 400 on the south side.

Both groups can't reach the general (or each other). So both groups elect their own commander:
- North group: "We're the real army! Start attacking!"
- South group: "WE'RE the real army! Start defending!"

**Result**: One army is now doing two conflicting things simultaneously. When communication is restored, the mess is unimaginable — conflicting orders, friendly fire, chaos.

This is **split brain**: a network partition causes a single cluster to become two independent clusters, each believing it's authoritative.

---

## Core Concept Explained Step-by-Step

### What is Split Brain?

```
SPLIT BRAIN:
═══════════════════════════════════════════════════

NORMAL CLUSTER (5 nodes, 1 leader):
┌─────────────────────────────────────────────┐
│                                             │
│   [F1] ←──→ [F2] ←──→ [LEADER] ←──→ [F3] ←──→ [F4]   │
│                                             │
│   All nodes can communicate. Leader handles  │
│   writes. Followers replicate.              │
└─────────────────────────────────────────────┘

NETWORK PARTITION HAPPENS:
┌──────────────────────┐   ╳╳╳   ┌──────────────────────┐
│  PARTITION A         │  NETWORK │  PARTITION B         │
│                      │   CUT    │                      │
│  [LEADER] [F1] [F2] │         │  [F3] [F4]           │
│                      │         │                      │
│  "We're the cluster! │         │  "Leader is dead!    │
│   We have the leader"│         │   Elect new leader!" │
└──────────────────────┘         └──────────────────────┘

NOW TWO LEADERS EXIST (SPLIT BRAIN!):
┌──────────────────────┐         ┌──────────────────────┐
│  [LEADER-1] [F1] [F2]│         │  [LEADER-2] [F3] [F4]│
│                      │         │                      │
│  Accepting writes!   │         │  Also accepting      │
│  "UPDATE user SET    │         │  writes!             │
│   balance=100"       │         │  "UPDATE user SET    │
│                      │         │   balance=200"       │
└──────────────────────┘         └──────────────────────┘

WHEN NETWORK HEALS:
  User balance = 100? or 200? CONFLICT! 💀
  Two different states. Which one is "correct"?
```

### Why Split Brain is Dangerous

```
CONSEQUENCES OF SPLIT BRAIN:
═══════════════════════════════════════════════════

1. DATA DIVERGENCE:
   Both sides accept writes independently
   After healing: conflicting records everywhere
   
2. DUPLICATE PROCESSING:
   A job scheduler on both sides → same job runs TWICE
   Billing system: customer charged TWICE
   
3. RESOURCE CONFLICTS:
   Both sides create VM with same IP address
   Both sides claim same S3 bucket
   
4. CORRUPT STATE:
   Database A: INSERT order #1001 (from partition A)
   Database B: INSERT order #1001 (from partition B)
   Different data, same primary key → constraint violation on merge!

5. UNSAFE OPERATIONS:
   Fencing token #5 issued by Leader A
   Fencing token #5 ALSO issued by Leader B
   Distributed lock is now NOT exclusive!
```

### Causes of Split Brain

```
COMMON CAUSES:
═══════════════════════════════════════════════════

1. NETWORK PARTITION:
   Physical network link fails between racks/datacenters
   
   [Rack A] ════╳════ [Rack B]
   
2. SWITCH/ROUTER FAILURE:
   Top-of-rack switch dies → all nodes in that rack isolated
   
3. ASYMMETRIC PARTITION:
   A can talk to B, B can talk to C, but A CANNOT talk to C
   
   A ──→ B ──→ C
   A ────╳────→ C
   
4. GC PAUSE / PROCESS PAUSE:
   Leader has 30-second GC pause → followers think it's dead
   Elect new leader → old leader wakes up → TWO leaders!
   
5. FULL NETWORK CONGESTION:
   Network so congested that heartbeats don't arrive in time
   Falsely triggers failure detection
```

---

## How It Works Internally

### Solution 1: Quorum (Majority Wins)

```
QUORUM-BASED APPROACH:
═══════════════════════════════════════════════════

Rule: A partition can only elect a leader if it has 
      MAJORITY of nodes (> N/2).

5-node cluster, partition splits 3|2:

  Partition A: [Leader, F1, F2]  → 3 nodes = MAJORITY ✅
    → Can elect/keep a leader
    → Can accept writes
    → CONTINUES OPERATING
    
  Partition B: [F3, F4]          → 2 nodes = MINORITY ❌
    → Cannot form a quorum
    → REFUSES to elect a leader
    → Goes READ-ONLY or stops entirely
    
RESULT: Only ONE side can operate as leader!
        Split brain PREVENTED! ✅

Why N/2 + 1?
  With N=5: quorum = 3
  Any two groups of 3 share at least 1 node
  → IMPOSSIBLE for two groups to both have a majority
  
  Partition [3 | 2] → only [3] has quorum
  Partition [2 | 3] → only [3] has quorum  
  Partition [2 | 2 | 1] → NO group has quorum → cluster STOPS
```

### Solution 2: Fencing (STONITH)

```
FENCING / STONITH (Shoot The Other Node In The Head):
═══════════════════════════════════════════════════

When a new leader is elected, KILL the old leader to be safe.

Mechanism:
1. Partition detected → new leader elected in majority
2. New leader sends STONITH command to old leader:
   - Power off via IPMI/BMC (hardware fencing)
   - Kill VM via hypervisor API
   - Revoke network access via SDN
3. Old leader is DEAD — can't cause split brain

Before:
  [OLD-LEADER] ──╳── [NEW-LEADER elected by majority]
  
After STONITH:
  [💀 DEAD] ──╳── [NEW-LEADER]  ← Only one leader! Safe!

Used by:
  - Pacemaker/Corosync (Linux HA)
  - VMware HA (restart VMs on different host)
  - AWS: terminate old leader's EC2 instance
```

### Solution 3: Fencing Tokens

```
FENCING TOKENS (software-based protection):
═══════════════════════════════════════════════════

Every time a leader is elected, it gets an INCREMENTING token.

Leader A gets token #5
  → Uses token #5 for all writes to storage

Network partition → Leader B elected with token #6
  → Uses token #6 for all writes to storage

Storage server enforces:
  "I will only accept writes with token >= my last seen token"

Scenario:
  Leader A (token #5) sends write → Storage sees token 5 → Accepts ✅
  Leader B (token #6) sends write → Storage sees token 6 → Accepts ✅
  Leader A (token #5) sends another write → Storage checks: 
    "Last token I saw was 6. Token 5 < 6. REJECT!" ❌

  ┌──────────┐         ┌───────────────┐
  │ Leader A │──token 5──▶│   Storage    │
  │ (stale!) │         │               │
  └──────────┘         │  Last seen: 6 │
                       │  5 < 6 → DENY │
  ┌──────────┐         │               │
  │ Leader B │──token 6──▶│   ACCEPTED   │
  │  (new)   │         │               │
  └──────────┘         └───────────────┘

No hardware fencing needed! Storage layer enforces safety.
```

### Solution 4: Witness / Tie-Breaker Node

```
WITNESS NODE (odd number strategy):
═══════════════════════════════════════════════════

Problem: 2-node cluster. Partition = 1|1. No majority!

Solution: Add a WITNESS (lightweight node that only votes):

  [Node A] ←──→ [WITNESS] ←──→ [Node B]
  
  If A-B partition:
    [Node A] ──→ [WITNESS] ──╳── [Node B]
    
    A + Witness = majority (2/3) → A continues
    B alone = minority (1/3) → B stops
    
  The witness doesn't store data — just participates in quorum!

Cloud equivalents:
  - AWS: "Quorum disk" in S3 (both nodes try to write a lock file)
  - Azure: Cloud Witness for Windows Failover Clustering
  - ZooKeeper/etcd: External arbiter service
```

### Solution 5: Lease-Based Leadership

```
LEASE-BASED LEADERSHIP:
═══════════════════════════════════════════════════

Leader holds a TIME-LIMITED LEASE (like a lock with expiry):

  Leader renews lease every 5 seconds
  Lease expires after 15 seconds without renewal
  
Normal operation:
  t=0:  Leader renews lease (valid until t=15)
  t=5:  Leader renews lease (valid until t=20)
  t=10: Leader renews lease (valid until t=25)
  
Network partition at t=12:
  t=12: Leader can't reach lease store!
  t=15: Leader knows: "My lease is about to expire"
  t=15: Leader STEPS DOWN voluntarily!
        (Stops accepting writes before lease expires)
  
  t=20: Lease expires. Followers detect: "No valid lease"
  t=20: Followers can now safely elect new leader
  
  ┌──────────────────────────────────────────────┐
  │ TIMELINE:                                    │
  │                                              │
  │  ──renew──renew──PARTITION──step-down──      │
  │  0     5      10    12       15              │
  │                                  │           │
  │                         ──────expire──elect──│
  │                              20       22     │
  │                                              │
  │ GAP: System unavailable 15-22 (7 seconds)    │
  │ But NO split brain! Safety preserved!        │
  └──────────────────────────────────────────────┘

Key insight: Leader must stop BEFORE lease expires.
             There's a safety gap = lease_duration - renewal_interval.
```

---

## Code Examples

### Python: Split Brain Detection and Prevention

```python
import time
import threading
from enum import Enum
from dataclasses import dataclass, field
from typing import Optional, Set, Dict

class NodeRole(Enum):
    LEADER = "LEADER"
    FOLLOWER = "FOLLOWER"
    CANDIDATE = "CANDIDATE"
    STOPPED = "STOPPED"  # Stepped down due to split brain

@dataclass
class LeaseInfo:
    holder: Optional[str] = None
    token: int = 0
    expires_at: float = 0.0
    
    def is_valid(self) -> bool:
        return time.time() < self.expires_at

class QuorumBasedNode:
    """
    Node that uses quorum-based split brain prevention.
    Only operates as leader if it can reach a majority.
    """
    
    def __init__(self, node_id: str, cluster_size: int):
        self.node_id = node_id
        self.cluster_size = cluster_size
        self.quorum_size = (cluster_size // 2) + 1
        self.role = NodeRole.FOLLOWER
        self.current_leader: Optional[str] = None
        self.reachable_nodes: Set[str] = {node_id}
        self.fencing_token: int = 0
        
        # Lease management
        self.lease = LeaseInfo()
        self.lease_duration = 15.0  # seconds
        self.renewal_interval = 5.0
        
        print(f"[{node_id}] Started. Cluster size={cluster_size}, "
              f"quorum={self.quorum_size}")
    
    def update_reachable_nodes(self, reachable: Set[str]):
        """Update which nodes we can communicate with."""
        old_reachable = self.reachable_nodes
        self.reachable_nodes = reachable | {self.node_id}
        
        lost = old_reachable - self.reachable_nodes
        gained = self.reachable_nodes - old_reachable
        
        if lost:
            print(f"[{self.node_id}] Lost contact with: {lost}")
        if gained:
            print(f"[{self.node_id}] Gained contact with: {gained}")
        
        # Check if we still have quorum
        self._check_quorum()
    
    def _check_quorum(self):
        """If we're leader but lost quorum, step down immediately."""
        reachable_count = len(self.reachable_nodes)
        has_quorum = reachable_count >= self.quorum_size
        
        if self.role == NodeRole.LEADER and not has_quorum:
            print(f"[{self.node_id}] ⚠️ LOST QUORUM! "
                  f"Reachable: {reachable_count}/{self.cluster_size}. "
                  f"Need: {self.quorum_size}. STEPPING DOWN!")
            self._step_down()
            
        elif self.role == NodeRole.FOLLOWER and has_quorum:
            if self.current_leader is None:
                print(f"[{self.node_id}] Have quorum and no leader. "
                      f"Starting election...")
    
    def try_become_leader(self) -> bool:
        """Attempt to become leader (only if quorum is met)."""
        if len(self.reachable_nodes) < self.quorum_size:
            print(f"[{self.node_id}] Cannot become leader: "
                  f"no quorum ({len(self.reachable_nodes)}/{self.quorum_size})")
            return False
        
        # Acquire lease with incremented fencing token
        self.fencing_token += 1
        self.lease = LeaseInfo(
            holder=self.node_id,
            token=self.fencing_token,
            expires_at=time.time() + self.lease_duration
        )
        self.role = NodeRole.LEADER
        self.current_leader = self.node_id
        
        print(f"[{self.node_id}] ✅ Became LEADER with fencing token #{self.fencing_token}")
        return True
    
    def _step_down(self):
        """Voluntarily stop being leader (split brain prevention)."""
        self.role = NodeRole.STOPPED
        self.current_leader = None
        print(f"[{self.node_id}] 🛑 Stepped down. Role=STOPPED. "
              f"Will not accept writes.")
    
    def write(self, key: str, value: str) -> bool:
        """Accept a write (only if we're a valid leader)."""
        if self.role != NodeRole.LEADER:
            print(f"[{self.node_id}] ❌ REJECTED write: not leader "
                  f"(role={self.role.value})")
            return False
        
        if not self.lease.is_valid():
            print(f"[{self.node_id}] ❌ REJECTED write: lease expired! "
                  f"Stepping down.")
            self._step_down()
            return False
        
        if len(self.reachable_nodes) < self.quorum_size:
            print(f"[{self.node_id}] ❌ REJECTED write: lost quorum!")
            self._step_down()
            return False
        
        print(f"[{self.node_id}] ✅ ACCEPTED write: {key}={value} "
              f"(token #{self.fencing_token})")
        return True
    
    def status(self) -> str:
        return (f"[{self.node_id}] Role={self.role.value}, "
                f"Reachable={len(self.reachable_nodes)}/{self.cluster_size}, "
                f"Quorum={'YES' if len(self.reachable_nodes) >= self.quorum_size else 'NO'}, "
                f"Leader={self.current_leader}")


class FencingTokenStorage:
    """
    Storage that enforces fencing tokens.
    Rejects writes from stale leaders.
    """
    
    def __init__(self):
        self.data: Dict[str, str] = {}
        self.last_token: int = 0
    
    def write(self, key: str, value: str, fencing_token: int) -> bool:
        if fencing_token < self.last_token:
            print(f"  [Storage] ❌ REJECTED write from stale leader "
                  f"(token {fencing_token} < last seen {self.last_token})")
            return False
        
        self.last_token = fencing_token
        self.data[key] = value
        print(f"  [Storage] ✅ Accepted write: {key}={value} "
              f"(token #{fencing_token})")
        return True


# --- Simulation: Split Brain Scenario ---
print("=" * 60)
print("SPLIT BRAIN SIMULATION")
print("=" * 60)

# 5-node cluster
nodes = {}
all_ids = {"N1", "N2", "N3", "N4", "N5"}
for nid in all_ids:
    nodes[nid] = QuorumBasedNode(nid, cluster_size=5)

# N1 becomes leader (all nodes reachable)
print("\n--- Phase 1: Normal operation ---")
for nid in all_ids:
    nodes[nid].update_reachable_nodes(all_ids)
nodes["N1"].try_become_leader()
nodes["N1"].write("balance", "1000")

# Simulate network partition: [N1, N2] | [N3, N4, N5]
print("\n--- Phase 2: Network partition! ---")
print("Partition: [N1, N2] | [N3, N4, N5]")

# Partition A: N1 and N2 can only reach each other
nodes["N1"].update_reachable_nodes({"N1", "N2"})
nodes["N2"].update_reachable_nodes({"N1", "N2"})

# Partition B: N3, N4, N5 can reach each other
nodes["N3"].update_reachable_nodes({"N3", "N4", "N5"})
nodes["N4"].update_reachable_nodes({"N3", "N4", "N5"})
nodes["N5"].update_reachable_nodes({"N3", "N4", "N5"})

# N1 tries to write (should be rejected — lost quorum!)
print("\n--- Phase 3: Old leader tries to write ---")
nodes["N1"].write("balance", "2000")

# N3 tries to become leader in the majority partition
print("\n--- Phase 4: Majority partition elects new leader ---")
nodes["N3"].try_become_leader()
nodes["N3"].write("balance", "1500")

# Show final status
print("\n--- Final Status ---")
for nid in sorted(all_ids):
    print(nodes[nid].status())

# Fencing token demo
print("\n\n--- Fencing Token Demo ---")
storage = FencingTokenStorage()
storage.write("balance", "1000", fencing_token=5)  # Old leader
storage.write("balance", "1500", fencing_token=6)  # New leader  
storage.write("balance", "2000", fencing_token=5)  # Old leader (stale!)
```

### Java: Split Brain Prevention with Fencing

```java
import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.atomic.*;

/**
 * Split brain prevention using quorum + fencing tokens.
 * Demonstrates how systems like ZooKeeper and etcd prevent dual-leader scenarios.
 */
public class SplitBrainPrevention {
    
    enum Role { LEADER, FOLLOWER, STOPPED }
    
    static class ClusterNode {
        private final String nodeId;
        private final int clusterSize;
        private final int quorumSize;
        private volatile Role role = Role.FOLLOWER;
        private volatile int fencingToken = 0;
        private volatile Set<String> reachableNodes;
        private volatile long leaseExpiresAt = 0;
        private static final long LEASE_DURATION_MS = 15_000;
        
        public ClusterNode(String nodeId, int clusterSize) {
            this.nodeId = nodeId;
            this.clusterSize = clusterSize;
            this.quorumSize = (clusterSize / 2) + 1;
            this.reachableNodes = Set.of(nodeId);
        }
        
        public synchronized void updateReachableNodes(Set<String> reachable) {
            Set<String> newReachable = new HashSet<>(reachable);
            newReachable.add(nodeId);
            this.reachableNodes = newReachable;
            
            System.out.printf("[%s] Reachable nodes: %s (%d/%d, quorum=%s)%n",
                    nodeId, newReachable, newReachable.size(), clusterSize,
                    hasQuorum() ? "YES" : "NO");
            
            if (role == Role.LEADER && !hasQuorum()) {
                stepDown("Lost quorum");
            }
        }
        
        public boolean hasQuorum() {
            return reachableNodes.size() >= quorumSize;
        }
        
        public synchronized boolean tryBecomeLeader() {
            if (!hasQuorum()) {
                System.out.printf("[%s] Cannot become leader: no quorum%n", nodeId);
                return false;
            }
            
            fencingToken++;
            role = Role.LEADER;
            leaseExpiresAt = System.currentTimeMillis() + LEASE_DURATION_MS;
            
            System.out.printf("[%s] ✅ ELECTED as leader (token=#%d)%n", 
                    nodeId, fencingToken);
            return true;
        }
        
        public synchronized boolean write(String key, String value) {
            if (role != Role.LEADER) {
                System.out.printf("[%s] ❌ Write REJECTED: role=%s%n", nodeId, role);
                return false;
            }
            
            if (System.currentTimeMillis() > leaseExpiresAt) {
                stepDown("Lease expired");
                System.out.printf("[%s] ❌ Write REJECTED: lease expired%n", nodeId);
                return false;
            }
            
            if (!hasQuorum()) {
                stepDown("Lost quorum during write");
                return false;
            }
            
            System.out.printf("[%s] ✅ Write ACCEPTED: %s=%s (token=#%d)%n",
                    nodeId, key, value, fencingToken);
            return true;
        }
        
        private void stepDown(String reason) {
            System.out.printf("[%s] 🛑 STEPPING DOWN (%s)%n", nodeId, reason);
            role = Role.STOPPED;
        }
        
        public int getFencingToken() { return fencingToken; }
        public Role getRole() { return role; }
    }
    
    /**
     * Storage with fencing token enforcement.
     * Rejects writes from stale leaders.
     */
    static class FencedStorage {
        private final Map<String, String> data = new ConcurrentHashMap<>();
        private final AtomicInteger lastSeenToken = new AtomicInteger(0);
        
        public boolean write(String key, String value, int fencingToken) {
            int lastToken = lastSeenToken.get();
            
            if (fencingToken < lastToken) {
                System.out.printf("  [Storage] ❌ REJECTED: stale token %d < %d%n",
                        fencingToken, lastToken);
                return false;
            }
            
            lastSeenToken.set(fencingToken);
            data.put(key, value);
            System.out.printf("  [Storage] ✅ ACCEPTED: %s=%s (token=#%d)%n",
                    key, value, fencingToken);
            return true;
        }
        
        public String read(String key) { return data.get(key); }
    }
    
    public static void main(String[] args) {
        System.out.println("═══════════════════════════════════════");
        System.out.println(" SPLIT BRAIN PREVENTION DEMO");
        System.out.println("═══════════════════════════════════════");
        
        // Create 5-node cluster
        var allNodes = Set.of("N1", "N2", "N3", "N4", "N5");
        var nodes = new HashMap<String, ClusterNode>();
        for (String id : allNodes) {
            nodes.put(id, new ClusterNode(id, 5));
        }
        
        var storage = new FencedStorage();
        
        // Phase 1: Normal operation
        System.out.println("\n--- Phase 1: All nodes connected ---");
        nodes.values().forEach(n -> n.updateReachableNodes(allNodes));
        nodes.get("N1").tryBecomeLeader();
        
        int token1 = nodes.get("N1").getFencingToken();
        storage.write("user:balance", "1000", token1);
        
        // Phase 2: Network partition [N1,N2] | [N3,N4,N5]
        System.out.println("\n--- Phase 2: PARTITION [N1,N2] | [N3,N4,N5] ---");
        nodes.get("N1").updateReachableNodes(Set.of("N1", "N2"));
        nodes.get("N2").updateReachableNodes(Set.of("N1", "N2"));
        nodes.get("N3").updateReachableNodes(Set.of("N3", "N4", "N5"));
        nodes.get("N4").updateReachableNodes(Set.of("N3", "N4", "N5"));
        nodes.get("N5").updateReachableNodes(Set.of("N3", "N4", "N5"));
        
        // Phase 3: Old leader can't write
        System.out.println("\n--- Phase 3: Old leader tries to write ---");
        nodes.get("N1").write("user:balance", "WRONG!");
        
        // Phase 4: New leader in majority partition
        System.out.println("\n--- Phase 4: Majority elects new leader ---");
        nodes.get("N3").tryBecomeLeader();
        
        int token2 = nodes.get("N3").getFencingToken();
        storage.write("user:balance", "1500", token2);
        
        // Even if old leader bypassed quorum check (bug!),
        // storage rejects stale token:
        System.out.println("\n--- Phase 5: Fencing token protects storage ---");
        storage.write("user:balance", "HACKED!", token1);  // Stale!
        
        System.out.printf("%nFinal value: user:balance = %s%n", 
                storage.read("user:balance"));
    }
}
```

---

## Infrastructure Examples

### ZooKeeper: Preventing Split Brain

```
ZOOKEEPER QUORUM:
═══════════════════════════════════════════════════

ZooKeeper ensemble (5 nodes):
  - Requires majority (3) to serve ANY write
  - If partition splits 3|2:
    - Group of 3: continues operating
    - Group of 2: stops accepting writes (read-only mode)

ZooKeeper session guarantees:
  - Client sessions have timeouts
  - If client can't reach quorum, session expires
  - Ephemeral nodes are deleted when session expires
  - Leader lock (ephemeral node) is auto-released!

Configuration:
  server.1=zk1:2888:3888
  server.2=zk2:2888:3888
  server.3=zk3:2888:3888
  server.4=zk4:2888:3888
  server.5=zk5:2888:3888

  # Port 2888: follower-to-leader
  # Port 3888: leader election
```

### Kubernetes: Split Brain in etcd

```
KUBERNETES etcd CLUSTER:
═══════════════════════════════════════════════════

Kubernetes control plane uses etcd (Raft consensus):
  - 3 or 5 etcd nodes
  - Raft guarantees: only 1 leader at a time
  - Partition with minority → minority goes read-only

If etcd loses quorum:
  - kubectl commands HANG (can't write to etcd)
  - No new pods can be scheduled
  - Running pods CONTINUE (kubelet works independently)
  - But no new deployments, no scaling, no healing

Recovery:
  # Check etcd health
  etcdctl endpoint health --cluster
  
  # Check which node is leader
  etcdctl endpoint status --cluster -w table
  
  # If permanently lost quorum (disaster):
  # Restore from snapshot (LAST RESORT)
  etcdctl snapshot restore backup.db
```

### Redis Sentinel: Split Brain Protection

```
REDIS SENTINEL SPLIT BRAIN:
═══════════════════════════════════════════════════

Redis Sentinel uses quorum for failover decisions:

  sentinel.conf:
    sentinel monitor mymaster 10.0.1.1 6379 2
    #                                        ^ quorum=2
    # Need at least 2 sentinels to agree on failover

Split brain scenario:
  [Sentinel-1] [Redis-Master]  |PARTITION|  [Sentinel-2] [Sentinel-3] [Redis-Replica]
  
  Sentinel-2 + Sentinel-3 agree: "Master is down" → promote Replica
  Sentinel-1 alone: can't reach quorum → doesn't promote anything
  
Protection: min-replicas-to-write
  redis.conf:
    min-replicas-to-write 1
    min-replicas-max-lag 10
    
  If master can't reach ANY replica for 10 seconds:
    Master STOPS accepting writes!
    Prevents stale master from diverging during partition.
```

### PostgreSQL: Split Brain with Patroni

```
PATRONI (PostgreSQL HA):
═══════════════════════════════════════════════════

Patroni uses DCS (Distributed Configuration Store):
  - etcd, ZooKeeper, or Consul as arbiter
  - Leader must keep renewing a LOCK in DCS
  - If lock expires → leader MUST stop accepting writes

Flow:
  1. Leader holds DCS lock (TTL = 30s, renew every 10s)
  2. Network partition → leader can't renew lock
  3. Lock expires after 30s
  4. Leader detects: "My lock expired" → demotes to read-only
  5. Remaining nodes elect new leader via DCS
  
  patroni.yml:
    ttl: 30          # Lock time-to-live
    loop_wait: 10    # Check interval
    retry_timeout: 10
    
  This guarantees:
    - Old leader stops within 30 seconds
    - New leader can't start until old lock expires
    - No overlap! No split brain!
```

---

## Real-World Example

### Elasticsearch Split Brain (Historical)

```
ELASTICSEARCH SPLIT BRAIN (pre-7.x):
═══════════════════════════════════════════════════

Old config (DANGEROUS):
  discovery.zen.minimum_master_nodes: 1  ← TOO LOW!
  
  With 3 master-eligible nodes and minimum_master_nodes=1:
  Partition [1] | [1] | [1] → ALL three elect themselves master!
  
Fix: Set minimum_master_nodes = (N/2) + 1
  discovery.zen.minimum_master_nodes: 2
  
  Now partition [1] | [2]:
    Group of 2 → elects master ✅
    Group of 1 → can't form quorum ❌

Elasticsearch 7.0+ (FIXED):
  cluster.initial_master_nodes: ["node1", "node2", "node3"]
  
  Automatic quorum calculation!
  No more manual minimum_master_nodes configuration.
  Split brain is prevented by default.
```

### MongoDB: Avoiding Split Brain in Replica Sets

```
MONGODB REPLICA SET ELECTION:
═══════════════════════════════════════════════════

MongoDB requires majority vote for election:
  - 3-member set: needs 2 votes
  - 5-member set: needs 3 votes

Priority-based voting + arbiter node:

  rs.initiate({
    _id: "myReplicaSet",
    members: [
      { _id: 0, host: "mongo1:27017", priority: 2 },  // Preferred primary
      { _id: 1, host: "mongo2:27017", priority: 1 },
      { _id: 2, host: "mongo3:27017", arbiterOnly: true }  // Tie-breaker
    ]
  });

Write concern for safety:
  db.collection.insertOne(
    { balance: 1500 },
    { writeConcern: { w: "majority", j: true, wtimeout: 5000 } }
  );
  
  // Write only acknowledged after majority of nodes confirm!
  // If partition isolates the primary, write TIMES OUT (safe!)
```

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Wrong | Fix |
|---------|---------------|-----|
| Even number of nodes | Partition can split exactly 50/50 → no majority possible | Always use ODD numbers (3, 5, 7) |
| Ignoring GC pauses | JVM GC pause mimics a partition → false failover | Tune GC, set lease > max GC pause |
| No fencing on shared storage | Old leader's in-flight writes arrive AFTER new leader starts | Implement fencing tokens on all storage |
| Setting quorum to 1 | Any single node can declare itself master | Set quorum = (N/2) + 1 minimum |
| Not testing partitions | "It won't happen to us" → it does | Use chaos engineering (Jepsen, Chaos Monkey) |
| Same rack for all nodes | Physical failure takes out everything → total outage | Spread across racks/AZs |
| Aggressive failure timeouts | Normal network jitter triggers false failovers → cascading split brain | Use adaptive detection (φ accrual) |

---

## When to Use / When NOT to Use

### Apply Split Brain Prevention When:
- ✅ Any leader/follower database setup (PostgreSQL, MySQL, MongoDB)
- ✅ Distributed lock services (ZooKeeper, etcd, Consul)
- ✅ Clustered systems with shared state (Elasticsearch, Kafka controllers)
- ✅ Job schedulers (only one scheduler should be active)
- ✅ Multi-datacenter deployments with async replication

### Less Critical When:
- ❌ Stateless services (no writes to diverge; load balancer just retries)
- ❌ Pure AP systems designed for eventual consistency (Cassandra, DynamoDB)
- ❌ Read-only replicas (split brain only matters for WRITES)
- ❌ Single-node setups (no partition possible)

### Summary of Prevention Strategies:

```
┌────────────────────────┬────────────────────────────────────┐
│ Strategy               │ Used By                            │
├────────────────────────┼────────────────────────────────────┤
│ Quorum (majority vote) │ ZooKeeper, etcd, Raft, MongoDB     │
│ Fencing tokens         │ Chubby, lock services, Redlock     │
│ STONITH (kill old)     │ Pacemaker, VMware HA, cloud HA     │
│ Lease-based            │ Chubby, Patroni, DynamoDB          │
│ Witness/Arbiter        │ Azure, MongoDB arbiter, SQL Server │
│ min-replicas-to-write  │ Redis, Kafka (min.insync.replicas) │
└────────────────────────┴────────────────────────────────────┘
```

---

## Key Takeaways

- 🔑 **Split brain** occurs when a network partition causes two groups to independently elect leaders — leading to data divergence and corruption.
- 🔑 **Quorum (majority vote)** is the primary defense: only a partition with > N/2 nodes can elect a leader.
- 🔑 **Always use odd numbers** of nodes (3, 5, 7) — even numbers can deadlock with 50/50 splits.
- 🔑 **Fencing tokens** protect shared storage: monotonically increasing tokens let storage reject writes from stale leaders.
- 🔑 **Lease-based leadership** forces the old leader to step down BEFORE the lease expires — creating a safe gap.
- 🔑 **STONITH** (Shoot The Other Node In The Head) is the nuclear option: physically power-off the old leader.
- 🔑 **Test for split brain** with tools like Jepsen and Chaos Monkey — it's not "if" a partition happens, but "when."

---

## What's Next?

You've completed **PART 13: Distributed Systems — The Hard Stuff**! You now understand the fundamental challenges of building systems that span multiple machines. Next, we move to **PART 14: Security** — starting with **[Authentication vs Authorization](../14-security/01-authentication-vs-authorization.md)** — because a distributed system that isn't secure is just a distributed vulnerability.
