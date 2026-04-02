# Distributed Systems - SKILL.md

## Overview
Graduate-level distributed systems covering fundamental theory, consensus protocols, replication, messaging, storage, microservices architecture, and observability.

---

## Fundamental Theory

### Concept
Theoretical foundations that define the limits and possibilities of distributed computing.

### Principles

**CAP Theorem (Brewer's Theorem):**
- **Consistency:** Every read returns the most recent write or an error (linearizability)
- **Availability:** Every request receives a non-error response (no guarantee of freshness)
- **Partition Tolerance:** System continues to operate despite network partitions
- **Theorem:** During a network partition, you must choose between C and A
- **Practical implication:** All real systems must handle partitions → choose CP or AP

| System Type | C | A | Example |
|-------------|---|---|---------|
| CP | ✓ | ✗ | ZooKeeper, HBase, MongoDB (default) |
| AP | ✗ | ✓ | Cassandra, DynamoDB, CouchDB |
| CA | ✓ | ✓ | Impossible in distributed systems (single-node RDBMS) |

**FLP Impossibility Theorem:**
- In an asynchronous distributed system, consensus is impossible if even one process can crash
- No deterministic algorithm can guarantee consensus in bounded time
- **Practical workaround:** Use randomization (Paxos) or timeouts (Raft) to make progress

**Consistency Models (from strongest to weakest):**

| Model | Guarantee | Cost |
|-------|-----------|------|
| Linearizable | All operations appear atomic, globally ordered | Highest latency |
| Sequential | Operations of each process in order, total order | High latency |
| Causal | Causally related operations seen in order | Moderate overhead |
| Eventual | All replicas converge given no new updates | Low latency |

**Consistent Hashing:**
- Map both nodes and keys onto a hash ring (0 to 2^160-1)
- Each key is owned by the first node clockwise from its hash position
- **Virtual Nodes:** Each physical node maps to multiple positions on the ring
- **Advantage:** Adding/removing a node only affects K/N keys (K = total keys, N = nodes)
- **Used by:** DynamoDB, Cassandra, Riak, Akka

### Algorithms & Code

**Causal Consistency (Vector Clocks):**

```python
class VectorClock:
    def __init__(self, node_ids):
        self.clock = {nid: 0 for nid in node_ids}
    
    def increment(self, node_id):
        self.clock[node_id] += 1
    
    def merge(self, other):
        """Merge with another vector clock (take component-wise max)."""
        all_keys = set(self.clock) | set(other.clock)
        merged = {}
        for k in all_keys:
            merged[k] = max(self.clock.get(k, 0), other.clock.get(k, 0))
        self.clock = merged
        return self
    
    def happens_before(self, other):
        """Check if self → other (causal ordering)."""
        at_least_one_less = False
        for k in set(self.clock) | set(other.clock):
            if self.clock.get(k, 0) > other.clock.get(k, 0):
                return False
            if self.clock.get(k, 0) < other.clock.get(k, 0):
                at_least_one_less = True
        return at_least_one_less
    
    def concurrent(self, other):
        """Check if self || other (concurrent events)."""
        return not self.happens_before(other) and not other.happens_before(self)
    
    def __repr__(self):
        return f"VC({self.clock})"


class CausalStore:
    """Key-value store with causal consistency."""
    def __init__(self, node_id, node_ids):
        self.node_id = node_id
        self.data = {}  # key → (value, vector_clock)
        self.vc = VectorClock(node_ids)
    
    def write(self, key, value):
        self.vc.increment(self.node_id)
        self.data[key] = (value, VectorClock(dict(self.vc.clock)))
    
    def read(self, key):
        if key in self.data:
            return self.data[key]
        return None
    
    def sync(self, other_store, key):
        """Synchronize with another node for a key (resolve conflicts)."""
        if key not in other_store.data:
            return
        
        my_vc = self.data.get(key, (None, VectorClock([])))[1]
        their_vc = other_store.data[key][1]
        
        if my_vc.concurrent(their_vc):
            # Conflict! Apply last-writer-wins or merge
            self._resolve_conflict(key, self.data.get(key), other_store.data[key])
        elif their_vc.happens_before(my_vc):
            pass  # Our version is newer
        else:
            # Their version is newer
            self.data[key] = other_store.data[key]
            self.vc.merge(their_vc)
    
    def _resolve_conflict(self, key, my_entry, their_entry):
        """Last-writer-wins conflict resolution."""
        # In practice, use application-specific merge (CRDT)
        self.data[key] = their_entry  # Simple LWW
        self.vc.merge(their_entry[1])
```

**Consistent Hashing Ring:**

```python
import hashlib
import bisect

class ConsistentHashRing:
    def __init__(self, virtual_nodes=150):
        self.vnodes = virtual_nodes
        self.ring = []       # Sorted list of hash positions
        self.mapping = {}    # hash → node_id
        self.nodes = set()
    
    def _hash(self, key):
        return int(hashlib.sha256(key.encode()).hexdigest(), 16) % (2**128)
    
    def add_node(self, node_id):
        self.nodes.add(node_id)
        for i in range(self.vnodes):
            h = self._hash(f"{node_id}#vn{i}")
            bisect.insort(self.ring, h)
            self.mapping[h] = node_id
    
    def remove_node(self, node_id):
        self.nodes.discard(node_id)
        for i in range(self.vnodes):
            h = self._hash(f"{node_id}#vn{i}")
            idx = bisect.bisect_left(self.ring, h)
            if idx < len(self.ring) and self.ring[idx] == h:
                self.ring.pop(idx)
                del self.mapping[h]
    
    def get_node(self, key):
        if not self.ring:
            return None
        h = self._hash(key)
        idx = bisect.bisect_right(self.ring, h)
        if idx == len(self.ring):
            idx = 0
        return self.mapping[self.ring[idx]]
    
    def get_replicas(self, key, n=3):
        """Get n distinct nodes for replication."""
        if not self.ring or n <= 0:
            return []
        h = self._hash(key)
        idx = bisect.bisect_right(self.ring, h)
        
        result = []
        seen = set()
        for i in range(len(self.ring)):
            pos = (idx + i) % len(self.ring)
            node = self.mapping[self.ring[pos]]
            if node not in seen:
                seen.add(node)
                result.append(node)
                if len(result) == n:
                    break
        return result
```

### Trade-offs

**Consistency Models:**
- **Strong (Linearizable):** Correct but slow (needs coordination)
- **Causal:** Good balance, vector clock overhead
- **Eventual:** Fast reads, stale data possible, conflict resolution needed

**Consistent Hashing:**
- **More virtual nodes:** Better distribution, more memory
- **Fewer virtual nodes:** Less overhead, potentially uneven distribution

### Applications

- **Distributed Caching:** Memcached consistent hashing
- **Database Sharding:** Cassandra token ring
- **CDN:** Routing requests to nearest edge node
- **Load Balancing:** Distributing requests across backends

---

## Consensus Protocols

### Concept
Algorithms that enable distributed nodes to agree on a single value or state, despite failures.

### Principles

**Paxos (Basic Paxos):**
- **Roles:** Proposer, Acceptor, Learner
- **Phases:**
  1. **Prepare:** Proposer sends numbered proposal to acceptors
  2. **Promise:** Acceptors promise not to accept lower-numbered proposals
  3. **Accept:** Proposer sends value, acceptors accept if number is highest seen
  4. **Learn:** Accepted value is communicated to learners

**Multi-Paxos:**
- Skip prepare phase when a stable leader exists
- Leader can directly send Accept requests
- Much more efficient for consecutive proposals

**Raft:**
- **Leader Election:** Term-based, randomized timeouts
  - Follower → Candidate (election timeout)
  - Candidate requests votes from all nodes
  - Majority vote → Leader
- **Log Replication:**
  - Leader appends to log, replicates to followers
  - Commit when majority have the entry
  - Followers apply committed entries to state machine
- **Safety:**
  - Election restriction: Candidate's log must be at least as up-to-date
  - Leader completeness: Committed entries are never lost
- **Membership Changes:** Joint consensus (old + new configuration)

**ZAB (ZooKeeper Atomic Broadcast):**
- Used by ZooKeeper for metadata management
- Similar to Raft: leader-based, term-based
- **Phases:** Discovery → Synchronization → Broadcast
- Primary use: Configuration management, leader election service

**PBFT (Practical Byzantine Fault Tolerance):**
- Tolerates f Byzantine (arbitrary) failures with 3f+1 nodes
- **Phases:** Pre-prepare → Prepare → Commit
- Each replica maintains a view number
- View change protocol for leader failure
- **Cost:** O(n²) message complexity per consensus round

### Algorithms & Code

**Raft Implementation (Core):**

```python
import random
import time
from enum import Enum

class NodeState(Enum):
    FOLLOWER = 'follower'
    CANDIDATE = 'candidate'
    LEADER = 'leader'

class LogEntry:
    def __init__(self, term, command):
        self.term = term
        self.command = command

class RaftNode:
    def __init__(self, node_id, peers):
        # Persistent state
        self.node_id = node_id
        self.current_term = 0
        self.voted_for = None
        self.log = [LogEntry(0, None)]  # Index 0 is sentinel
        
        # Volatile state
        self.commit_index = 0
        self.last_applied = 0
        self.state = NodeState.FOLLOWER
        self.leader_id = None
        
        # Leader state
        self.next_index = {}  # peer → next log index to send
        self.match_index = {}  # peer → highest replicated index
        
        # Election
        self.peers = peers
        self.votes_received = set()
        self.election_timeout = self._random_timeout()
        self.last_heartbeat = time.time()
    
    def _random_timeout(self):
        return random.uniform(150, 300)  # ms
    
    def tick(self):
        """Called periodically."""
        if self.state == NodeState.LEADER:
            self._send_heartbeats()
        else:
            if time.time() - self.last_heartbeat > self.election_timeout / 1000:
                self._start_election()
    
    def _start_election(self):
        self.current_term += 1
        self.state = NodeState.CANDIDATE
        self.voted_for = self.node_id
        self.votes_received = {self.node_id}
        self.last_heartbeat = time.time()
        
        last_log_idx = len(self.log) - 1
        last_log_term = self.log[-1].term
        
        for peer in self.peers:
            self._request_vote(peer, self.current_term, self.node_id,
                              last_log_idx, last_log_term)
    
    def handle_request_vote(self, term, candidate_id, last_log_idx, last_log_term):
        """Handle incoming RequestVote RPC."""
        if term < self.current_term:
            return {'term': self.current_term, 'vote_granted': False}
        
        if term > self.current_term:
            self._step_down(term)
        
        # Voting rules
        if self.voted_for is None or self.voted_for == candidate_id:
            # Log completeness check
            my_last_idx = len(self.log) - 1
            my_last_term = self.log[-1].term
            
            if last_log_term > my_last_term or \
               (last_log_term == my_last_term and last_log_idx >= my_last_idx):
                self.voted_for = candidate_id
                self.last_heartbeat = time.time()
                return {'term': self.current_term, 'vote_granted': True}
        
        return {'term': self.current_term, 'vote_granted': False}
    
    def handle_vote_response(self, term, vote_granted):
        """Handle response to our vote request."""
        if term > self.current_term:
            self._step_down(term)
            return
        
        if vote_granted and self.state == NodeState.CANDIDATE:
            self.votes_received.add(term)  # Simplified
            if len(self.votes_received) > (len(self.peers) + 1) // 2:
                self._become_leader()
    
    def _become_leader(self):
        self.state = NodeState.LEADER
        self.leader_id = self.node_id
        next_idx = len(self.log)
        for peer in self.peers:
            self.next_index[peer] = next_idx
            self.match_index[peer] = 0
    
    def handle_append_entries(self, term, leader_id, prev_log_idx, 
                               prev_log_term, entries, leader_commit):
        """Handle AppendEntries RPC."""
        if term < self.current_term:
            return {'term': self.current_term, 'success': False}
        
        self.last_heartbeat = time.time()
        
        if term > self.current_term:
            self._step_down(term)
        
        self.leader_id = leader_id
        
        # Log consistency check
        if prev_log_idx >= len(self.log):
            return {'term': self.current_term, 'success': False}
        
        if self.log[prev_log_idx].term != prev_log_term:
            # Log mismatch, delete conflicting entry and all following
            self.log = self.log[:prev_log_idx]
            return {'term': self.current_term, 'success': False}
        
        # Append new entries
        for i, entry in enumerate(entries):
            idx = prev_log_idx + 1 + i
            if idx < len(self.log):
                if self.log[idx].term != entry.term:
                    self.log = self.log[:idx]
                    self.log.append(entry)
            else:
                self.log.append(entry)
        
        # Update commit index
        if leader_commit > self.commit_index:
            self.commit_index = min(leader_commit, len(self.log) - 1)
            self._apply_committed()
        
        return {'term': self.current_term, 'success': True}
    
    def _send_heartbeats(self):
        """Send AppendEntries to all peers."""
        for peer in self.peers:
            prev_idx = self.next_index[peer] - 1
            prev_term = self.log[prev_idx].term if prev_idx < len(self.log) else 0
            entries = self.log[self.next_index[peer]:]
            
            self._append_entries(peer, self.current_term, self.node_id,
                                prev_idx, prev_term, entries, self.commit_index)
    
    def _apply_committed(self):
        """Apply committed entries to state machine."""
        while self.last_applied < self.commit_index:
            self.last_applied += 1
            entry = self.log[self.last_applied]
            # Apply entry.command to state machine
    
    def _step_down(self, term):
        self.current_term = term
        self.state = NodeState.FOLLOWER
        self.voted_for = None
    
    def propose(self, command):
        """Propose a new command (leader only)."""
        if self.state != NodeState.LEADER:
            return False
        
        entry = LogEntry(self.current_term, command)
        self.log.append(entry)
        
        # Replicate to followers
        self._send_heartbeats()
        
        # Check if committed (majority replicated)
        # ... (handle in append_entries responses)
        return True
```

**PBFT (Simplified):**

```python
class PBFTNode:
    def __init__(self, node_id, all_nodes, f=1):
        self.node_id = node_id
        self.all_nodes = all_nodes  # 3f + 1 nodes
        self.f = f
        self.view = 0
        self.sequence = 0
        self.log = []  # Messages seen
        self.prepared = {}   # (view, seq) → message
        self.committed = {}  # (view, seq) → message
    
    def request(self, operation):
        """Client sends request to primary."""
        msg = {'type': 'request', 'op': operation, 
               'timestamp': time.time(), 'client': self.node_id}
        primary = self.all_nodes[self.view % len(self.all_nodes)]
        primary.handle_request(msg)
    
    def handle_request(self, msg):
        """Primary handles client request."""
        if self.node_id != self.all_nodes[self.view % len(self.all_nodes)].node_id:
            return  # Not primary
        
        self.sequence += 1
        pre_prepare = {
            'type': 'pre-prepare', 'view': self.view,
            'sequence': self.sequence, 'digest': hash(str(msg)),
            'message': msg
        }
        self.log.append(pre_prepare)
        
        # Broadcast to all replicas
        for node in self.all_nodes:
            node.handle_pre_prepare(pre_prepare)
    
    def handle_pre_prepare(self, msg):
        """Replica handles pre-prepare from primary."""
        self.log.append(msg)
        
        prepare = {
            'type': 'prepare', 'view': msg['view'],
            'sequence': msg['sequence'], 'digest': msg['digest'],
            'node_id': self.node_id
        }
        
        # Send prepare to all nodes
        for node in self.all_nodes:
            node.handle_prepare(prepare)
    
    def handle_prepare(self, msg):
        """Handle prepare message from another replica."""
        key = (msg['view'], msg['sequence'])
        if key not in self.prepared:
            self.prepared[key] = []
        self.prepared[key].append(msg)
        
        # Check if we have 2f prepares (including our own)
        if len(self.prepared[key]) >= 2 * self.f:
            # Prepared! Send commit
            commit = {
                'type': 'commit', 'view': msg['view'],
                'sequence': msg['sequence'], 'digest': msg['digest'],
                'node_id': self.node_id
            }
            for node in self.all_nodes:
                node.handle_commit(commit)
    
    def handle_commit(self, msg):
        """Handle commit message."""
        key = (msg['view'], msg['sequence'])
        if key not in self.committed:
            self.committed[key] = []
        self.committed[key].append(msg)
        
        # Check if we have 2f+1 commits
        if len(self.committed[key]) >= 2 * self.f + 1:
            # Committed! Execute operation
            self._execute(msg)
    
    def _execute(self, msg):
        """Execute the committed operation."""
        # Apply to state machine
        pass
```

### Trade-offs

**Consensus Protocol Selection:**

| Protocol | Fault Tolerance | Message Complexity | Latency | Use Case |
|----------|-----------------|-------------------|---------|----------|
| Paxos | Crash (f < N/2) | O(N) (Multi-Paxos) | 2 RTT | Chubby, Spanner |
| Raft | Crash (f < N/2) | O(N) | 2 RTT | etcd, Consul, TiKV |
| ZAB | Crash (f < N/2) | O(N) | 2 RTT | ZooKeeper |
| PBFT | Byzantine (f < N/3) | O(N²) | 3 RTT | Blockchain, Hyperledger |

**Raft vs Paxos:**
- Raft: Easier to understand, leader-based, strong consistency
- Paxos: More flexible, harder to implement, proven correct

### Applications

- **Configuration Management:** etcd (Raft), ZooKeeper (ZAB)
- **Distributed Locking:** Consul, ZooKeeper
- **Database Replication:** TiKV (Raft), CockroachDB (Raft)
- **Blockchain:** PBFT variants for permissioned chains

---

## Distributed Transactions

### Concept
Protocols for executing transactions that span multiple nodes or services.

### Principles

**2PC (Two-Phase Commit):**
1. **Prepare Phase:** Coordinator asks all participants to prepare
2. **Commit/Abort Phase:** If all prepared → commit; if any abort → abort all
- **Blocking Problem:** If coordinator crashes after prepare, participants are locked
- **Solution:** 3PC adds pre-commit phase (requires failure detector)

**3PC (Three-Phase Commit):**
1. **CanCommit:** Coordinator checks if participants can commit
2. **PreCommit:** Participants prepare and acknowledge
3. **DoCommit:** Coordinator sends final commit
- **Advantage:** Non-blocking (if network doesn't partition)
- **Disadvantage:** More message rounds, still vulnerable to partitions

**TCC (Try-Confirm-Cancel):**
- **Try:** Reserve resources (check inventory, hold funds)
- **Confirm:** Commit the operation (deduct inventory, charge card)
- **Cancel:** Release reserved resources (release hold)
- Each service must implement try, confirm, cancel operations
- Used in: Alibaba Seata, service mesh transaction frameworks

**Saga Pattern:**
- **Choreography:** Each service emits events, next service reacts
- **Orchestration:** Central coordinator invokes services in sequence
- **Compensation:** Each step has a compensating action for rollback
- **Guarantee:** Eventually consistent (not ACID)

**Distributed Snapshot (Chandy-Lamport):**
- Record consistent global state without stopping the system
- Uses marker messages on each channel
- Algorithm:
  1. Initiator records its state, sends markers on all outgoing channels
  2. On receiving first marker: record state, send markers, start recording incoming channel
  3. On receiving subsequent markers: stop recording that channel
  4. Collected states form a consistent cut

### Algorithms & Code

**2PC Coordinator:**

```python
class Coordinator2PC:
    def __init__(self, participants, timeout=5.0):
        self.participants = participants
        self.timeout = timeout
        self.transaction_log = {}
    
    def execute(self, transaction_id, operations):
        """Execute distributed transaction using 2PC."""
        self.transaction_log[transaction_id] = {'status': 'preparing'}
        
        # Phase 1: Prepare
        votes = {}
        for participant, op in zip(self.participants, operations):
            try:
                vote = participant.prepare(transaction_id, op)
                votes[participant] = vote
            except TimeoutError:
                votes[participant] = 'abort'
            except Exception:
                votes[participant] = 'abort'
        
        # Phase 2: Commit or Abort
        if all(v == 'prepared' for v in votes.values()):
            decision = 'commit'
        else:
            decision = 'abort'
        
        self.transaction_log[transaction_id]['status'] = decision
        
        for participant in self.participants:
            try:
                if decision == 'commit':
                    participant.commit(transaction_id)
                else:
                    participant.abort(transaction_id)
            except Exception:
                # Retry until successful (participants must be able to commit eventually)
                self._retry(participant, transaction_id, decision)
        
        return decision
    
    def _retry(self, participant, txid, decision):
        """Retry commit/abort until successful."""
        for _ in range(3):
            try:
                if decision == 'commit':
                    participant.commit(txid)
                else:
                    participant.abort(txid)
                return
            except Exception:
                time.sleep(1)
        # Log for manual intervention
        print(f"CRITICAL: Failed to {decision} tx {txid} on {participant}")


class Participant2PC:
    def __init__(self, resource_manager):
        self.rm = resource_manager
        self.prepared = {}  # txid → undo_info
    
    def prepare(self, txid, operation):
        """Prepare to commit. Write undo log."""
        undo_info = self.rm.apply_temporarily(txid, operation)
        self.prepared[txid] = undo_info
        # Write prepare record to durable log
        return 'prepared'
    
    def commit(self, txid):
        """Commit the prepared transaction."""
        self.rm.make_permanent(txid)
        del self.prepared[txid]
    
    def abort(self, txid):
        """Abort and undo the transaction."""
        if txid in self.prepared:
            self.rm.undo(txid, self.prepared[txid])
            del self.prepared[txid]
```

**Saga Orchestrator with Compensation:**

```python
class SagaDefinition:
    def __init__(self):
        self.steps = []
    
    def add_step(self, name, action_fn, compensate_fn):
        self.steps.append({
            'name': name,
            'action': action_fn,
            'compensate': compensate_fn
        })
        return self

class SagaExecution:
    def __init__(self, definition):
        self.definition = definition
        self.completed_steps = []  # (step_index, result)
    
    def execute(self):
        """Execute saga with automatic compensation on failure."""
        for i, step in enumerate(self.definition.steps):
            try:
                result = step['action']()
                self.completed_steps.append((i, result))
            except Exception as e:
                print(f"Saga failed at step '{step['name']}': {e}")
                self._compensate()
                raise
        
        return {'status': 'completed', 'steps': len(self.definition.steps)}
    
    def _compensate(self):
        """Compensate completed steps in reverse order."""
        for i, result in reversed(self.completed_steps):
            step = self.definition.steps[i]
            try:
                step['compensate'](result)
            except Exception as e:
                print(f"Compensation failed for step '{step['name']}': {e}")
                # Log for manual intervention (dead letter queue)

# Example: Order processing saga
saga = SagaDefinition()
saga.add_step(
    'reserve_stock',
    action_fn=lambda: {'reservation_id': 'res-123'},
    compensate_fn=lambda res: release_stock(res['reservation_id'])
)
saga.add_step(
    'process_payment',
    action_fn=lambda: {'payment_id': 'pay-456'},
    compensate_fn=lambda res: refund_payment(res['payment_id'])
)
saga.add_step(
    'create_order',
    action_fn=lambda: {'order_id': 'ord-789'},
    compensate_fn=lambda res: cancel_order(res['order_id'])
)
saga.add_step(
    'schedule_delivery',
    action_fn=lambda: {'tracking_id': 'trk-012'},
    compensate_fn=lambda res: cancel_delivery(res['tracking_id'])
)

execution = SagaExecution(saga)
execution.execute()
```

**Chandy-Lamport Distributed Snapshot:**

```python
class DistributedSnapshot:
    def __init__(self, node_id, channels):
        self.node_id = node_id
        self.channels = channels  # Incoming channels from other nodes
        self.state = None
        self.channel_states = {ch: [] for ch in channels}
        self.markers_received = set()
        self.snapshot_complete = False
    
    def initiate_snapshot(self):
        """Start a snapshot (called by initiator)."""
        # Step 1: Record own state
        self.state = self._get_current_state()
        
        # Step 2: Send markers on all outgoing channels
        for channel in self.channels:
            self._send_marker(channel)
        
        # Step 3: Start recording incoming channels
        self.recording = True
    
    def handle_marker(self, channel, marker):
        """Handle incoming marker message."""
        if self.state is None:
            # First marker: record state, send markers, start recording
            self.state = self._get_current_state()
            self.markers_received.add(channel)
            
            for ch in self.channels:
                self._send_marker(ch)
            
            self.recording = True
        else:
            # Subsequent marker: stop recording this channel
            self.markers_received.add(channel)
        
        # Check if snapshot is complete
        if self.markers_received == set(self.channels):
            self.snapshot_complete = True
    
    def handle_message(self, channel, message):
        """Handle regular (non-marker) message."""
        if self.recording and channel not in self.markers_received:
            # Record message as part of channel state
            self.channel_states[channel].append(message)
        
        # Process message normally
        self._process(message)
    
    def _get_current_state(self):
        """Capture current application state."""
        return {'node': self.node_id, 'data': dict(self._application_state)}
    
    def _send_marker(self, channel):
        """Send marker on a channel."""
        channel.send({'type': 'marker', 'source': self.node_id})
    
    def _process(self, message):
        """Process application message."""
        pass
```

### Trade-offs

**Transaction Models:**

| Model | Consistency | Performance | Complexity | Failure Handling |
|-------|------------|-------------|------------|-----------------|
| 2PC | Strong | Low (blocking) | Medium | Coordinator failure blocks |
| TCC | Strong | Medium | High | Requires compensation logic |
| Saga | Eventual | High | Medium | Compensation chain |
| 1PC (per service) | Weak | Highest | Low | Manual reconciliation |

**Saga Choreography vs Orchestration:**
- **Choreography:** Decentralized, event-driven, harder to debug
- **Orchestration:** Centralized, easier to monitor, single point of failure

### Applications

- **E-Commerce:** Order processing across inventory, payment, shipping services
- **Banking:** Fund transfers across different banking systems
- **Travel Booking:** Flight + hotel + car reservation
- **Microservices:** Cross-service business transactions

---

## Replication

### Concept
Maintaining copies of data across multiple nodes for availability, durability, and read scaling.

### Principles

**Replication Topologies:**

| Type | Write Path | Read Path | Conflict Risk |
|------|-----------|-----------|--------------|
| Primary-Secondary | Write to primary, replicate | Read from any | None (single writer) |
| Multi-Master | Write to any | Read from any | High (concurrent writes) |
| Consensus (Raft) | Write to leader | Read from leader | None (consensus) |

**Read-Write Quorums (Dynamo-style):**
- N = replication factor
- W = write quorum (min writes acknowledged)
- R = read quorum (min reads to contact)
- **Consistency guarantee:** W + R > N → strong read-after-write consistency
- **Common configurations:**
  - Strong: R=2, W=2, N=3 (any 2 overlap)
  - Eventual: R=1, W=1, N=3 (fast but stale reads possible)

**Anti-Entropy:**
- **Read Repair:** Fix inconsistencies detected during reads
- **Hinted Handoff:** Store writes for temporarily down nodes
- **Merkle Tree:** Efficiently detect differences between replicas
  - Hash tree of data partitions
  - Compare root hashes; if different, compare children recursively
  - O(log n) comparisons to find divergent data

**Conflict Resolution Strategies:**
- **Last Writer Wins (LWW):** Use timestamp, simple but lossy
- **Application-level merge:** Custom logic (e.g., CRDTs)
- **CRDT (Conflict-free Replicated Data Types):**
  - **G-Counter:** Grow-only counter (max of per-node counts)
  - **PN-Counter:** Increment/decrement counter (two G-counters)
  - **G-Set:** Grow-only set (union of all elements)
  - **OR-Set:** Observed-remove set (add wins with unique tags)
  - **LWW-Register:** Last-writer-wins register

### Algorithms & Code

**Merkle Tree for Anti-Entropy:**

```python
import hashlib

class MerkleTree:
    def __init__(self, data_blocks, leaf_size=1024):
        self.data = data_blocks
        self.leaf_size = leaf_size
        self.tree = self._build_tree()
    
    def _hash(self, data):
        return hashlib.sha256(str(data).encode()).hexdigest()
    
    def _build_tree(self):
        """Build Merkle tree bottom-up."""
        # Create leaf nodes
        leaves = []
        for i in range(0, len(self.data), self.leaf_size):
            block = self.data[i:i+self.leaf_size]
            leaves.append(self._hash(block))
        
        # Build internal nodes
        tree = [leaves]
        while len(tree[-1]) > 1:
            level = tree[-1]
            parent_level = []
            for i in range(0, len(level), 2):
                left = level[i]
                right = level[i+1] if i+1 < len(level) else left
                parent_level.append(self._hash(left + right))
            tree.append(parent_level)
        
        return tree
    
    @property
    def root_hash(self):
        return self.tree[-1][0] if self.tree else None
    
    def diff(self, other):
        """Find differences with another Merkle tree."""
        if self.root_hash == other.root_hash:
            return []  # No differences
        
        return self._find_diffs(other, len(self.tree)-1, 0)
    
    def _find_diffs(self, other, level, index):
        """Recursively find differing leaf indices."""
        if level == 0:
            return [index]  # Leaf level, this block differs
        
        if self.tree[level][index] == other.tree[level][index]:
            return []  # Same hash, no differences below
        
        # Different hash, check children
        left_child = index * 2
        right_child = index * 2 + 1
        
        diffs = []
        if left_child < len(self.tree[level-1]):
            diffs.extend(self._find_diffs(other, level-1, left_child))
        if right_child < len(self.tree[level-1]):
            diffs.extend(self._find_diffs(other, level-1, right_child))
        
        return diffs


class AntiEntropySync:
    """Synchronize data between replicas using Merkle trees."""
    def __init__(self, node_id, data):
        self.node_id = node_id
        self.data = data  # Dict of key-value pairs
        self.merkle = MerkleTree(list(data.items()))
    
    def sync_with(self, other_node):
        """Synchronize with another node."""
        # Compare Merkle tree roots
        diffs = self.merkle.diff(other_node.merkle)
        
        if not diffs:
            return  # Already in sync
        
        # Exchange differing data
        for diff_idx in diffs:
            my_version = self._get_block(diff_idx)
            their_version = other_node._get_block(diff_idx)
            
            # Resolve conflict (LWW)
            for key in set(list(my_version.keys()) + list(their_version.keys())):
                my_ts = my_version.get(key, {}).get('timestamp', 0)
                their_ts = their_version.get(key, {}).get('timestamp', 0)
                
                if my_ts > their_ts:
                    other_node.data[key] = self.data[key]
                elif their_ts > my_ts:
                    self.data[key] = other_node.data[key]
        
        # Rebuild Merkle trees
        self.merkle = MerkleTree(list(self.data.items()))
        other_node.merkle = MerkleTree(list(other_node.data.items()))
```

**CRDT (G-Counter & PN-Counter):**

```python
class GCounter:
    """Grow-only counter CRDT."""
    def __init__(self, node_id, all_nodes):
        self.node_id = node_id
        self.counts = {node: 0 for node in all_nodes}
    
    def increment(self, amount=1):
        self.counts[self.node_id] += amount
    
    def value(self):
        return sum(self.counts.values())
    
    def merge(self, other):
        """Merge with another G-Counter (take max per node)."""
        for node in self.counts:
            self.counts[node] = max(self.counts[node], other.counts[node])


class PNCounter:
    """Positive-Negative counter CRDT."""
    def __init__(self, node_id, all_nodes):
        self.p = GCounter(node_id, all_nodes)  # Positive
        self.n = GCounter(node_id, all_nodes)  # Negative
        self.node_id = node_id
    
    def increment(self, amount=1):
        self.p.increment(amount)
    
    def decrement(self, amount=1):
        self.n.increment(amount)
    
    def value(self):
        return self.p.value() - self.n.value()
    
    def merge(self, other):
        self.p.merge(other.p)
        self.n.merge(other.n)


class ORSet:
    """Observed-Remove Set CRDT."""
    def __init__(self, node_id):
        self.node_id = node_id
        self.elements = {}  # element → set of unique tags
        self.tombstones = set()  # Removed tags
    
    def add(self, element):
        tag = (self.node_id, time.time())
        if element not in self.elements:
            self.elements[element] = set()
        self.elements[element].add(tag)
    
    def remove(self, element):
        if element in self.elements:
            self.tombstones |= self.elements[element]
            del self.elements[element]
    
    def contains(self, element):
        return element in self.elements and \
               bool(self.elements[element] - self.tombstones)
    
    def get_elements(self):
        return {e for e, tags in self.elements.items() 
                if tags - self.tombstones}
    
    def merge(self, other):
        """Merge with another OR-Set."""
        # Add: keep all tags
        for element, tags in other.elements.items():
            if element not in self.elements:
                self.elements[element] = set()
            self.elements[element] |= tags
        
        # Remove: keep all tombstones
        self.tombstones |= other.tombstones
        
        # Clean up: remove tombstoned elements
        for element in list(self.elements):
            remaining = self.elements[element] - self.tombstones
            if not remaining:
                del self.elements[element]
```

### Trade-offs

**Replication Factor (N):**
- Higher N: Better durability and availability, more storage and write cost
- Lower N: Less overhead, lower durability

**Quorum Configuration (R/W):**
- R+W > N: Strong consistency, higher latency
- R+W ≤ N: Eventual consistency, lower latency

**Synchronous vs Asynchronous Replication:**
- **Synchronous:** Zero data loss, higher write latency
- **Asynchronous:** Lower latency, potential data loss on failover

### Applications

- **Database Replication:** PostgreSQL streaming, MySQL GTID, MongoDB replica sets
- **Caching:** Redis replication, Memcached consistent hashing
- **File Systems:** HDFS replication (default 3x)
- **CDN:** Content replication to edge nodes

---

## Message Queues

### Concept
Asynchronous communication infrastructure enabling decoupled, reliable message delivery between services.

### Principles

**Apache Kafka:**
- **Topics:** Categories/feeds where messages are published
- **Partitions:** Ordered, immutable sequence of messages within a topic
- **Offsets:** Unique sequence ID per partition (consumer tracks position)
- **Consumer Groups:** Set of consumers cooperating to consume a topic
  - Each partition consumed by exactly one consumer in a group
  - Enables parallel consumption and replay
- **Replication:** Each partition replicated across brokers (leader + followers)
- **Exactly-Once Semantics:**
  - Idempotent producers (PID + sequence number)
  - Transactional producers (atomic writes across partitions)
  - Consumer read-process-write with transactions

**Kafka Architecture:**
```
Producer → [Broker 1: P0(L), P1(F)] → Consumer Group A
         → [Broker 2: P0(F), P1(L)] → Consumer Group B
         → [Broker 3: P2(L)]
```

**RabbitMQ:**
- **Exchange Types:**
  - **Direct:** Route by exact routing key
  - **Fanout:** Broadcast to all bound queues
  - **Topic:** Route by pattern (stock.*.price)
  - **Headers:** Route by message headers
- **Queue:** Buffer storing messages until consumed
- **Binding:** Link between exchange and queue with routing rules
- **Acknowledgments:** Consumer acks message after processing
- **Dead Letter Exchange:** Route failed/expired messages

**Apache Pulsar:**
- **Layered Architecture:** Separate compute (brokers) and storage (bookies)
- **Topics → Bundles → Brokers:** Automatic load balancing
- **Persistent vs Non-persistent:** Durable (BookKeeper) vs in-memory
- **Multi-tenancy:** Built-in tenant/namespace isolation
- **Geo-replication:** Cross-region replication

### Algorithms & Code

**Kafka Producer (Python):**

```python
from kafka import KafkaProducer, KafkaConsumer
import json

class ReliableKafkaProducer:
    def __init__(self, bootstrap_servers, transactional_id=None):
        config = {
            'bootstrap_servers': bootstrap_servers,
            'value_serializer': lambda v: json.dumps(v).encode(),
            'key_serializer': lambda k: k.encode() if k else None,
            'acks': 'all',  # Wait for all ISR acks
            'enable_idempotence': True,  # Exactly-once for producer
            'retries': 3,
            'max_in_flight_requests_per_connection': 5,
            'linger_ms': 10,  # Batch small sends
            'batch_size': 16384,
            'compression_type': 'lz4',
        }
        if transactional_id:
            config['transactional_id'] = transactional_id
        
        self.producer = KafkaProducer(**config)
        if transactional_id:
            self.producer.init_transactions()
    
    def send(self, topic, value, key=None, partition=None):
        future = self.producer.send(topic, value=value, key=key, partition=partition)
        return future
    
    def send_transactional(self, topic, value, key=None):
        """Send message within a transaction."""
        self.producer.begin_transaction()
        try:
            self.producer.send(topic, value=value, key=key)
            self.producer.commit_transaction()
        except Exception:
            self.producer.abort_transaction()
            raise
    
    def close(self):
        self.producer.flush()
        self.producer.close()


class KafkaStreamProcessor:
    """Consume-Process-Produce pattern with exactly-once."""
    def __init__(self, bootstrap_servers, group_id, 
                 input_topic, output_topic):
        self.consumer = KafkaConsumer(
            input_topic,
            bootstrap_servers=bootstrap_servers,
            group_id=group_id,
            enable_auto_commit=False,
            isolation_level='read_committed',
            value_deserializer=lambda m: json.loads(m.decode())
        )
        self.producer = ReliableKafkaProducer(
            bootstrap_servers,
            transactional_id=f"{group_id}-tx"
        )
        self.output_topic = output_topic
    
    def process_loop(self, transform_fn):
        for record in self.consumer:
            self.producer.producer.begin_transaction()
            try:
                output = transform_fn(record.value)
                self.producer.send(self.output_topic, value=output)
                
                # Commit consumer offset as part of transaction
                self.producer.producer.send_offsets_to_transaction(
                    {record.topic: {record.partition: record.offset + 1}},
                    self.consumer.consumer_group_metadata()
                )
                self.producer.producer.commit_transaction()
            except Exception:
                self.producer.producer.abort_transaction()
```

**RabbitMQ Exchange Routing:**

```python
import pika

class RabbitMQRouter:
    def __init__(self, host='localhost'):
        self.connection = pika.BlockingConnection(
            pika.ConnectionParameters(host)
        )
        self.channel = self.connection.channel()
    
    def setup_exchange(self, exchange_name, exchange_type='topic'):
        self.channel.exchange_declare(
            exchange=exchange_name,
            exchange_type=exchange_type,
            durable=True
        )
    
    def setup_queue(self, queue_name, exchange_name, routing_keys,
                     dlq=True):
        # Main queue
        args = {}
        if dlq:
            dlq_name = f"{queue_name}.dlq"
            self.channel.queue_declare(queue=dlq_name, durable=True)
            args = {
                'x-dead-letter-exchange': '',
                'x-dead-letter-routing-key': dlq_name
            }
        
        self.channel.queue_declare(queue=queue_name, durable=True,
                                    arguments=args)
        
        for key in routing_keys:
            self.channel.queue_bind(
                exchange=exchange_name,
                queue=queue_name,
                routing_key=key
            )
    
    def publish(self, exchange, routing_key, message):
        self.channel.basic_publish(
            exchange=exchange,
            routing_key=routing_key,
            body=json.dumps(message),
            properties=pika.BasicProperties(
                delivery_mode=2,  # Persistent
                content_type='application/json'
            )
        )
    
    def consume(self, queue, handler, prefetch=10):
        self.channel.basic_qos(prefetch_count=prefetch)
        
        def callback(ch, method, properties, body):
            try:
                message = json.loads(body)
                handler(message)
                ch.basic_ack(delivery_tag=method.delivery_tag)
            except Exception as e:
                # Nack and requeue (or go to DLQ)
                ch.basic_nack(delivery_tag=method.delivery_tag,
                             requeue=False)
        
        self.channel.basic_consume(queue=queue, on_message_callback=callback)
        self.channel.start_consuming()
```

### Trade-offs

**Kafka vs RabbitMQ vs Pulsar:**

| Feature | Kafka | RabbitMQ | Pulsar |
|---------|-------|----------|--------|
| Model | Log (pull) | Queue (push) | Log + Queue |
| Ordering | Per-partition | Per-queue | Per-partition |
| Replay | Yes (offset) | Limited | Yes (cursor) |
| Throughput | Very high | Moderate | High |
| Latency | Moderate | Low | Low |
| Retention | Time/size based | Ack-based | Time/size based |
| Backpressure | Consumer controls | Flow control | Consumer controls |
| Exactly-once | Yes (transactions) | No | Yes |

### Applications

- **Event Streaming:** Clickstream processing, IoT data ingestion
- **Microservices:** Async communication, event-driven architecture
- **Log Aggregation:** Centralized logging pipeline
- **Change Data Capture (CDC):** Database change events

---

## Distributed Storage

### Concept
Storage systems designed to scale across multiple nodes, handling petabytes of data with fault tolerance.

### Principles

**GFS/HDFS (Distributed File System):**
- **Architecture:** Single NameNode + multiple DataNodes
- **Chunk Size:** 64MB (GFS) / 128MB (HDFS) → reduces metadata overhead
- **Replication:** Default 3x, rack-aware placement
- **Write Path:** Client → NameNode (get locations) → Pipeline write to DataNodes
- **Read Path:** Client → NameNode (get locations) → Read from nearest DataNode
- **Weakness:** NameNode is SPOF (HDFS Federation mitigates)

**BigTable (Wide-Column Store):**
- **Data Model:** (row, column, timestamp) → value
- **Storage:** SSTables (sorted string tables) on GFS
- **Architecture:** Master + many TabletServers
- **MemTable + SSTable:** Writes go to MemTable (in-memory), flushed to SSTable
- **Compaction:** Merge SSTables to reduce read amplification
- **Used by:** HBase, Cassandra (variant)

**Cassandra:**
- **Ring topology:** Consistent hashing (token ring)
- **Data Model:** Keyspace → Column Family → Row → Column
- **Write Path:** MemTable + Commit Log → SSTable
- **Read Path:** MemTable → Row Cache → Key Cache → SSTable (bloom filter first)
- **Tunable Consistency:** ANY, ONE, QUORUM, ALL
- **Anti-Entropy:** Merkle tree based repair

**DynamoDB:**
- **Architecture:** Managed, multi-AZ, auto-scaling
- **Data Model:** Table → Item (JSON-like) → Attributes
- **Partition Key:** Hash-based distribution
- **Sort Key:** Range queries within partition
- **LSI/GSI:** Local/Global Secondary Indexes
- **Capacity:** On-demand or provisioned (RCU/WCU)

### Algorithms & Code

**LSM Tree (Log-Structured Merge Tree):**

```python
import bisect
import os

class SSTable:
    """Immutable sorted file on disk."""
    def __init__(self, filename):
        self.filename = filename
        self.index = {}  # key → file_offset
        self.min_key = None
        self.max_key = None
        self._load_index()
    
    def _load_index(self):
        """Load sparse index from file."""
        # In practice, store index block at end of file
        pass
    
    def get(self, key):
        """Binary search for key in SSTable."""
        if self.min_key and key < self.min_key:
            return None
        if self.max_key and key > self.max_key:
            return None
        
        # Find nearest index entry
        # Then scan block to find exact key
        # Implementation simplified
        return None

class MemTable:
    """In-memory sorted key-value store."""
    def __init__(self, max_size=4*1024*1024):  # 4MB default
        self.data = {}  # Sorted dict (use sortedcontainers in production)
        self.size = 0
        self.max_size = max_size
    
    def put(self, key, value):
        self.data[key] = value
        self.size += len(str(key)) + len(str(value))
    
    def get(self, key):
        return self.data.get(key)
    
    def is_full(self):
        return self.size >= self.max_size
    
    def flush(self, filename):
        """Flush to SSTable on disk."""
        with open(filename, 'w') as f:
            for key in sorted(self.data):
                f.write(f"{key}\t{self.data[key]}\n")

class BloomFilter:
    """Probabilistic set membership test."""
    def __init__(self, expected_items, false_positive_rate=0.01):
        import math
        self.size = int(-expected_items * math.log(false_positive_rate) / 
                       (math.log(2) ** 2))
        self.num_hashes = int(self.size / expected_items * math.log(2))
        self.bits = [False] * self.size
    
    def add(self, item):
        for seed in range(self.num_hashes):
            idx = self._hash(item, seed) % self.size
            self.bits[idx] = True
    
    def might_contain(self, item):
        for seed in range(self.num_hashes):
            idx = self._hash(item, seed) % self.size
            if not self.bits[idx]:
                return False
        return True
    
    def _hash(self, item, seed):
        import hashlib
        h = hashlib.sha256(f"{item}:{seed}".encode()).hexdigest()
        return int(h, 16)

class LSMTree:
    """Log-Structured Merge Tree."""
    def __init__(self, data_dir):
        self.data_dir = data_dir
        self.memtable = MemTable()
        self.sstables = []  # Level 0, 1, 2, ...
        self.bloom = BloomFilter(1000000)
        self.wal = open(os.path.join(data_dir, 'wal.log'), 'a')
    
    def put(self, key, value):
        # Write to WAL first (durability)
        self.wal.write(f"PUT\t{key}\t{value}\n")
        self.wal.flush()
        
        # Write to MemTable
        self.memtable.put(key, value)
        self.bloom.add(key)
        
        # Flush if full
        if self.memtable.is_full():
            self._flush_memtable()
    
    def get(self, key):
        # Check bloom filter first
        if not self.bloom.might_contain(key):
            return None
        
        # Check MemTable
        result = self.memtable.get(key)
        if result is not None:
            return result
        
        # Check SSTables (newest first)
        for sstable in reversed(self.sstables):
            result = sstable.get(key)
            if result is not None:
                return result
        
        return None
    
    def _flush_memtable(self):
        """Flush MemTable to SSTable and trigger compaction."""
        filename = os.path.join(self.data_dir, f"sstable_{len(self.sstables)}.sst")
        self.memtable.flush(filename)
        self.sstables.append(SSTable(filename))
        self.memtable = MemTable()
        
        # Trigger compaction if too many SSTables at level 0
        if len(self.sstables) > 4:
            self._compact()
    
    def _compact(self):
        """Merge SSTables (size-tiered compaction)."""
        # Merge all SSTables into one (simplified)
        merged = {}
        for sstable in self.sstables:
            # Read and merge
            pass
        
        # Write new SSTable
        # Delete old SSTables
```

### Trade-offs

**GFS/HDFS vs Object Storage:**
- **HDFS:** Better for batch processing, co-locate compute
- **S3:** More durable, cheaper, higher latency

**BigTable vs Cassandra:**
- **BigTable:** Strongly consistent (single master), Google managed
- **Cassandra:** Tunable consistency (no master), open source

**LSM Tree vs B-Tree:**
- **LSM:** Faster writes (sequential), slower reads (multiple SSTables)
- **B-Tree:** Faster reads (single lookup), slower writes (random I/O)

### Applications

- **Data Lakes:** HDFS/S3 for big data analytics
- **Time-Series:** Cassandra for high-write sensor data
- **Content Storage:** S3 for images, videos, backups
- **Session Store:** DynamoDB for web session data

---

## Microservices

### Concept
Architectural pattern where applications are composed of loosely coupled, independently deployable services.

### Principles

**Service Discovery:**
- **Client-Side Discovery:** Client queries registry, selects instance (Netflix Eureka)
- **Server-Side Discovery:** Load balancer/router queries registry (AWS ALB, Nginx)
- **Service Registry:** Consul, Eureka, etcd, ZooKeeper
- **Health Checking:** Active (HTTP/TCP probes) and passive (circuit breaker)

**Load Balancing:**

| Algorithm | Description | Best For |
|-----------|-------------|----------|
| Round Robin | Sequential rotation | Uniform backends |
| Weighted Round Robin | Weight-based rotation | Different capacities |
| Least Connections | Route to fewest active | Variable request duration |
| Consistent Hash | Hash-based sticky sessions | Cache-friendly routing |
| Random | Random selection | Simple, uniform load |
| P2C (Power of 2) | Pick 2 random, choose less loaded | Load-aware, low overhead |

**API Gateway:**
- **Functions:** Routing, authentication, rate limiting, request transformation
- **Implementations:** Kong, Nginx, Envoy, AWS API Gateway
- **Pattern:** Backend For Frontend (BFF) — dedicated gateway per client type

**Resilience Patterns:**
- **Circuit Breaker:** Three states (Closed → Open → Half-Open)
  - Closed: Normal operation, track failures
  - Open: Block requests, return fallback
  - Half-Open: Allow limited requests to test recovery
- **Rate Limiter:**
  - Token Bucket: Refill tokens at fixed rate, burst allowed
  - Leaky Bucket: Fixed output rate, smooth traffic
  - Sliding Window: Count requests in time window
  - Fixed Window: Count in fixed time intervals
- **Bulkhead:** Isolate resources per service/dependency
- **Timeout:** Connect timeout, read timeout, circuit timeout
- **Retry with Exponential Backoff:** Fixed → Linear → Exponential → Jitter

**Service Mesh (Istio/Envoy):**
- **Sidecar Proxy:** Envoy injected alongside each service
- **Control Plane:** Pilot (discovery), Citadel (mTLS), Galley (config)
- **Features:** Traffic management, observability, security, policy enforcement
- **mTLS:** Automatic encryption between services

**gRPC:**
- **Protocol Buffers:** Binary serialization, schema-first
- **HTTP/2:** Multiplexing, server push, header compression
- **Streaming:** Unary, Server, Client, Bidirectional
- **Code Generation:** Stubs for 11+ languages

### Algorithms & Code

**Circuit Breaker:**

```python
import time
from enum import Enum

class CircuitState(Enum):
    CLOSED = 'closed'
    OPEN = 'open'
    HALF_OPEN = 'half_open'

class CircuitBreaker:
    def __init__(self, failure_threshold=5, recovery_timeout=30,
                 half_open_max_calls=3):
        self.state = CircuitState.CLOSED
        self.failure_count = 0
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.half_open_max_calls = half_open_max_calls
        self.half_open_calls = 0
        self.last_failure_time = None
        self.success_count = 0
    
    def call(self, func, *args, **kwargs):
        if self.state == CircuitState.OPEN:
            if time.time() - self.last_failure_time > self.recovery_timeout:
                self.state = CircuitState.HALF_OPEN
                self.half_open_calls = 0
            else:
                raise CircuitBreakerOpen("Circuit is OPEN")
        
        if self.state == CircuitState.HALF_OPEN:
            if self.half_open_calls >= self.half_open_max_calls:
                raise CircuitBreakerOpen("Half-open limit reached")
            self.half_open_calls += 1
        
        try:
            result = func(*args, **kwargs)
            self._on_success()
            return result
        except Exception as e:
            self._on_failure()
            raise
    
    def _on_success(self):
        if self.state == CircuitState.HALF_OPEN:
            self.success_count += 1
            if self.success_count >= self.half_open_max_calls:
                self.state = CircuitState.CLOSED
                self.failure_count = 0
                self.success_count = 0
        self.failure_count = 0
    
    def _on_failure(self):
        self.failure_count += 1
        self.last_failure_time = time.time()
        
        if self.state == CircuitState.HALF_OPEN:
            self.state = CircuitState.OPEN
            self.success_count = 0
        elif self.failure_count >= self.failure_threshold:
            self.state = CircuitState.OPEN


class CircuitBreakerOpen(Exception):
    pass
```

**Token Bucket Rate Limiter:**

```python
import time
import threading

class TokenBucket:
    def __init__(self, rate, capacity):
        self.rate = rate          # Tokens added per second
        self.capacity = capacity  # Max tokens
        self.tokens = capacity
        self.last_refill = time.time()
        self.lock = threading.Lock()
    
    def consume(self, tokens=1):
        with self.lock:
            self._refill()
            
            if self.tokens >= tokens:
                self.tokens -= tokens
                return True
            return False
    
    def wait_and_consume(self, tokens=1):
        """Block until tokens available."""
        while True:
            with self.lock:
                self._refill()
                if self.tokens >= tokens:
                    self.tokens -= tokens
                    return
            
            sleep_time = (tokens - self.tokens) / self.rate
            time.sleep(max(0.001, sleep_time))
    
    def _refill(self):
        now = time.time()
        elapsed = now - self.last_refill
        new_tokens = elapsed * self.rate
        self.tokens = min(self.capacity, self.tokens + new_tokens)
        self.last_refill = now


class SlidingWindowRateLimiter:
    """Distributed-friendly sliding window rate limiter."""
    def __init__(self, limit, window_seconds):
        self.limit = limit
        self.window = window_seconds
        self.requests = []  # (timestamp, count)
    
    def allow(self):
        now = time.time()
        cutoff = now - self.window
        
        # Remove expired entries
        self.requests = [(ts, count) for ts, count in self.requests 
                         if ts > cutoff]
        
        # Check limit
        total = sum(count for _, count in self.requests)
        if total >= self.limit:
            return False
        
        self.requests.append((now, 1))
        return True


class DistributedRateLimiter:
    """Redis-based sliding window rate limiter."""
    LUA_SCRIPT = """
    local key = KEYS[1]
    local limit = tonumber(ARGV[1])
    local window = tonumber(ARGV[2])
    local now = tonumber(ARGV[3])
    
    -- Remove expired entries
    redis.call('ZREMRANGEBYSCORE', key, 0, now - window)
    
    -- Count current entries
    local count = redis.call('ZCARD', key)
    
    if count < limit then
        redis.call('ZADD', key, now, now .. ':' .. math.random())
        redis.call('EXPIRE', key, window / 1000)
        return 1
    end
    
    return 0
    """
    
    def __init__(self, redis_client, limit, window_seconds):
        self.redis = redis_client
        self.limit = limit
        self.window = window_seconds * 1000  # Convert to ms
        self.script = self.redis.register_script(self.LUA_SCRIPT)
    
    def allow(self, key):
        now = int(time.time() * 1000)
        return bool(self.script(keys=[key], 
                                args=[self.limit, self.window, now]))
```

**gRPC Service Definition:**

```protobuf
// order.proto
syntax = "proto3";

package order;

service OrderService {
  rpc CreateOrder(CreateOrderRequest) returns (CreateOrderResponse);
  rpc GetOrder(GetOrderRequest) returns (GetOrderResponse);
  rpc StreamOrderUpdates(StreamOrderRequest) returns (stream OrderUpdate);
}

message CreateOrderRequest {
  string user_id = 1;
  repeated OrderItem items = 2;
  ShippingAddress shipping = 3;
}

message OrderItem {
  string product_id = 1;
  int32 quantity = 2;
  double price = 3;
}

message CreateOrderResponse {
  string order_id = 1;
  OrderStatus status = 2;
}

message GetOrderRequest {
  string order_id = 1;
}

message GetOrderResponse {
  string order_id = 1;
  string user_id = 2;
  repeated OrderItem items = 3;
  OrderStatus status = 4;
  int64 created_at = 5;
}

message StreamOrderRequest {
  string user_id = 1;
}

message OrderUpdate {
  string order_id = 1;
  OrderStatus new_status = 2;
  string message = 3;
  int64 timestamp = 4;
}

enum OrderStatus {
  PENDING = 0;
  CONFIRMED = 1;
  SHIPPED = 2;
  DELIVERED = 3;
  CANCELLED = 4;
}
```

**Service Discovery with Consul:**

```python
import consul
import random

class ConsulServiceDiscovery:
    def __init__(self, consul_host='localhost', consul_port=8500):
        self.consul = consul.Consul(host=consul_host, port=consul_port)
    
    def register(self, service_name, service_id, address, port, 
                 health_check_url=None, tags=None):
        check = None
        if health_check_url:
            check = consul.Check.http(health_check_url, interval='10s',
                                       timeout='5s', deregister='30s')
        
        self.consul.agent.service.register(
            name=service_name,
            service_id=service_id,
            address=address,
            port=port,
            check=check,
            tags=tags or []
        )
    
    def deregister(self, service_id):
        self.consul.agent.service.deregister(service_id)
    
    def discover(self, service_name, healthy_only=True):
        """Discover instances of a service."""
        _, services = self.consul.health.service(service_name, passing=healthy_only)
        
        instances = []
        for service in services:
            svc = service['Service']
            instances.append({
                'id': svc['ID'],
                'address': svc['Address'],
                'port': svc['Port'],
                'tags': svc.get('Tags', [])
            })
        
        return instances
    
    def get_instance(self, service_name):
        """Get a single instance (random selection)."""
        instances = self.discover(service_name)
        if not instances:
            return None
        return random.choice(instances)
```

### Trade-offs

**Service Mesh vs Library-Based:**
- **Service Mesh (Istio):** Language-agnostic, centralized management, sidecar overhead
- **Library (Hystrix/Resilience4j):** Lower latency, language-coupled, per-app config

**gRPC vs REST:**
- **gRPC:** Binary, fast, streaming, schema-first, limited browser support
- **REST:** Text (JSON), universal, caching, human-readable

**Synchronous vs Asynchronous:**
- **Sync (REST/gRPC):** Simple, immediate response, coupling
- **Async (MQ/Events):** Decoupled, resilient, eventual consistency

### Applications

- **E-Commerce:** Order, inventory, payment, shipping services
- **Streaming:** Video encoding, delivery, billing services
- **IoT:** Device management, data processing, alerting services
- **FinTech:** Account, transaction, fraud detection services

---

## Distributed Tracing & Observability

### Concept
Tools and patterns for understanding, monitoring, and debugging distributed systems.

### Principles

**Three Pillars of Observability:**
- **Metrics:** Numeric measurements over time (counters, gauges, histograms)
- **Logs:** Discrete events with timestamps and context
- **Traces:** End-to-end request flow across services

**OpenTelemetry:**
- Unified standard for traces, metrics, and logs
- **API:** Tracer, Meter, Logger interfaces
- **SDK:** Implementation with exporters
- **Exporters:** Jaeger, Zipkin, Prometheus, Datadog
- **Propagation:** W3C TraceContext (traceparent, tracestate headers)

**Distributed Tracing:**
- **Trace:** End-to-end journey of a request
- **Span:** Unit of work within a trace (with context propagation)
- **Context Propagation:** Pass trace_id + span_id across service boundaries
- **Sampling:** Head-based (random) vs tail-based (keep slow/error traces)

**Jaeger/Zipkin Architecture:**
- **Instrumentation:** Library-specific tracers
- **Collection:** Agent → Collector (UDP/HTTP)
- **Storage:** Elasticsearch, Cassandra, or memory
- **UI:** Visualize traces, service dependency graph

**Prometheus/Grafana:**
- **Prometheus:** Pull-based metrics collection, PromQL query language
- **Metric Types:** Counter, Gauge, Histogram, Summary
- **Grafana:** Visualization dashboards, alerting
- **Alertmanager:** Alert routing, grouping, silencing

### Algorithms & Code

**Distributed Tracing (OpenTelemetry Python):**

```python
from opentelemetry import trace, metrics
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.jaeger.thrift import JaegerExporter
from opentelemetry.sdk.resources import Resource
from opentelemetry.propagate import inject, extract
import requests

# Setup tracing
def setup_tracing(service_name, jaeger_endpoint='http://localhost:14268/api/traces'):
    resource = Resource.create({"service.name": service_name})
    provider = TracerProvider(resource=resource)
    
    jaeger_exporter = JaegerExporter(
        collector_endpoint=jaeger_endpoint,
    )
    provider.add_span_processor(BatchSpanProcessor(jaeger_exporter))
    trace.set_tracer_provider(provider)
    
    return trace.get_tracer(service_name)


class TracedHttpClient:
    """HTTP client with distributed tracing propagation."""
    def __init__(self, tracer):
        self.tracer = tracer
    
    def get(self, url, headers=None):
        headers = headers or {}
        
        with self.tracer.start_as_current_span(
            f"HTTP GET {url}",
            attributes={"http.method": "GET", "http.url": url}
        ) as span:
            # Inject trace context into headers
            inject(headers)
            
            response = requests.get(url, headers=headers)
            span.set_attribute("http.status_code", response.status_code)
            
            if response.status_code >= 400:
                span.set_status(trace.StatusCode.ERROR)
            
            return response


class TraceableService:
    """Example service with tracing."""
    def __init__(self, service_name):
        self.tracer = setup_tracing(service_name)
        self.http = TracedHttpClient(self.tracer)
    
    def process_order(self, order_id):
        with self.tracer.start_as_current_span("process_order") as span:
            span.set_attribute("order.id", order_id)
            
            # Sub-operations become child spans
            order = self._fetch_order(order_id)
            self._validate_order(order)
            self._reserve_inventory(order)
            self._process_payment(order)
            
            return {"status": "processed"}
    
    def _fetch_order(self, order_id):
        with self.tracer.start_as_current_span("fetch_order") as span:
            span.set_attribute("order.id", order_id)
            return self.http.get(f"http://order-service/orders/{order_id}")
    
    def _validate_order(self, order):
        with self.tracer.start_as_current_span("validate_order") as span:
            # Validation logic
            pass
    
    def _reserve_inventory(self, order):
        with self.tracer.start_as_current_span("reserve_inventory") as span:
            pass
    
    def _process_payment(self, order):
        with self.tracer.start_as_current_span("process_payment") as span:
            pass
```

**Prometheus Metrics:**

```python
from prometheus_client import Counter, Histogram, Gauge, generate_latest
from flask import Flask, Response
import time

# Define metrics
REQUEST_COUNT = Counter(
    'http_requests_total',
    'Total HTTP requests',
    ['method', 'endpoint', 'status']
)

REQUEST_LATENCY = Histogram(
    'http_request_duration_seconds',
    'HTTP request latency',
    ['method', 'endpoint'],
    buckets=[0.01, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0]
)

ACTIVE_CONNECTIONS = Gauge(
    'active_connections',
    'Current active connections'
)

INVENTORY_LEVEL = Gauge(
    'inventory_level',
    'Current inventory level',
    ['product_id']
)

app = Flask(__name__)

@app.route('/api/orders', methods=['POST'])
def create_order():
    start_time = time.time()
    
    try:
        # Process order
        result = process_order()
        
        REQUEST_COUNT.labels(
            method='POST', endpoint='/api/orders', status=200
        ).inc()
        
        return result
    except Exception as e:
        REQUEST_COUNT.labels(
            method='POST', endpoint='/api/orders', status=500
        ).inc()
        raise
    finally:
        REQUEST_LATENCY.labels(
            method='POST', endpoint='/api/orders'
        ).observe(time.time() - start_time)

@app.route('/metrics')
def metrics():
    return Response(generate_latest(), mimetype='text/plain')
```

**Tail-Based Sampling:**

```python
class TailBasedSampler:
    """Keep traces that are slow or have errors."""
    def __init__(self, max_traces=100, latency_threshold=1.0):
        self.max_traces = max_traces
        self.latency_threshold = latency_threshold
        self.buffer = {}  # trace_id → trace_data
    
    def should_keep(self, trace_data):
        """Determine if a completed trace should be kept."""
        # Always keep error traces
        if any(span['status'] == 'ERROR' for span in trace_data['spans']):
            return True
        
        # Keep slow traces
        duration = trace_data['end_time'] - trace_data['start_time']
        if duration > self.latency_threshold:
            return True
        
        # Keep 10% of normal traces (random sampling)
        import random
        return random.random() < 0.1
    
    def add_span(self, span):
        """Add a span to the buffer."""
        trace_id = span['trace_id']
        if trace_id not in self.buffer:
            self.buffer[trace_id] = {
                'spans': [],
                'start_time': span['start_time'],
                'end_time': span['start_time']
            }
        
        self.buffer[trace_id]['spans'].append(span)
        self.buffer[trace_id]['end_time'] = max(
            self.buffer[trace_id]['end_time'],
            span['end_time']
        )
        
        # Evict old traces if buffer is full
        if len(self.buffer) > self.max_traces:
            self._evict()
    
    def _evict(self):
        """Evict oldest traces."""
        sorted_traces = sorted(
            self.buffer.items(),
            key=lambda x: x[1]['start_time']
        )
        for trace_id, _ in sorted_traces[:len(self.buffer) - self.max_traces]:
            del self.buffer[trace_id]
```

### Trade-offs

**Tracing Overhead:**
- **Always-on:** Complete visibility, performance cost (1-5% overhead)
- **Sampling:** Lower cost, may miss important traces
- **Tail-based:** Keep important traces, higher memory (buffer traces)

**Metrics vs Logs vs Traces:**
- **Metrics:** Best for alerting, aggregation, low cardinality
- **Logs:** Best for debugging, high detail, high volume
- **Traces:** Best for understanding request flow, cross-service

**Push vs Pull Metrics:**
- **Push (StatsD):** Fire-and-forget, client controls frequency
- **Pull (Prometheus):** Server controls scraping, easier service discovery

### Applications

- **Production Debugging:** Trace failed requests across services
- **Performance Optimization:** Identify latency bottlenecks
- **Alerting:** SLO/SLI-based alerts on metrics
- **Capacity Planning:** Forecast resource needs from trends

---

## Summary

Key takeaways for distributed systems:

1. **Theory:** CAP and FLP define fundamental limits; consistency models define the trade-off spectrum
2. **Consensus:** Raft dominates for crash-fault tolerance; PBFT for Byzantine; both are foundational building blocks
3. **Transactions:** 2PC for strong consistency, Saga for eventual; choose based on business requirements
4. **Replication:** Quorums (W+R>N) for tunable consistency; CRDTs for conflict-free merging
5. **Messaging:** Kafka for high-throughput event streaming; RabbitMQ for flexible routing; Pulsar for layered architecture
6. **Storage:** LSM trees for write-heavy workloads; consistent hashing for scalable partitioning
7. **Microservices:** Circuit breakers, rate limiters, and service mesh for resilience; gRPC for efficient RPC
8. **Observability:** OpenTelemetry for unified instrumentation; traces for debugging, metrics for alerting
