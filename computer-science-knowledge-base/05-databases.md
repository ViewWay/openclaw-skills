# Databases - SKILL.md

## Overview
Graduate-level database systems covering relational theory, query processing, transaction management, recovery, and distributed database architectures.

---

## Relational Model

### Concept
Mathematical foundation of relational databases based on set theory and first-order predicate logic.

### Principles

**Relational Algebra:**
- **Selection (σ):** Filter rows by predicate → σ_{age>20}(Student)
- **Projection (π):** Choose columns → π_{name,age}(Student)
- **Cross Product (×):** Cartesian product of two relations
- **Join (⋈):** Cross product filtered by condition
  - **Theta Join (⋈_θ):** General condition
  - **Equi-Join:** Equality condition
  - **Natural Join (⋈):** Equality on all common attributes
- **Set Operations:** Union (∪), Intersection (∩), Difference (−)
- **Rename (ρ):** Rename relations/attributes
- **Division (÷):** "For all" queries → R ÷ S

**Functional Dependencies (FD):**
- X → Y means Y is functionally determined by X
- **Trivial FD:** Y ⊆ X
- **Closure (X⁺):** All attributes determined by X
- Armstrong's Axioms:
  - Reflexivity: Y ⊆ X ⇒ X → Y
  - Augmentation: X → Y ⇒ XZ → YZ
  - Transitivity: X → Y, Y → Z ⇒ X → Z

**Normal Forms:**

| Normal Form | Condition | Anomaly Eliminated |
|-------------|-----------|-------------------|
| 1NF | Atomic values, no repeating groups | Multi-valued attributes |
| 2NF | 1NF + no partial dependencies | Insertion/deletion anomalies |
| 3NF | 2NF + no transitive dependencies | Transitive anomalies |
| BCNF | Every determinant is a candidate key | All anomalies from FDs |
| 4NF | BCNF + no multi-valued dependencies | MVD anomalies |

**ER Model:**
- **Entities:** Objects with independent existence
- **Attributes:** Properties (simple, composite, derived, multi-valued)
- **Relationships:** Associations (1:1, 1:N, M:N)
- **Cardinality Constraints:** min..max notation

### Algorithms & Code

**FD Closure Computation:**

```python
def compute_closure(attributes, fds):
    """Compute X+ given attribute set X and FDs."""
    closure = set(attributes)
    changed = True
    while changed:
        changed = False
        for fd in fds:
            lhs, rhs = set(fd[0]), set(fd[1])
            if lhs.issubset(closure) and not rhs.issubset(closure):
                closure |= rhs
                changed = True
    return closure

# Example
fds = [('AB', 'C'), ('C', 'D'), ('D', 'A')]
print(compute_closure('B', fds))  # {'B'}
print(compute_closure('AB', fds))  # {'A','B','C','D'}
```

**BCNF Decomposition:**

```python
def bcnf_decompose(relation, fds):
    """Decompose relation into BCNF."""
    result = [relation]
    for fd in fds:
        lhs, rhs = set(fd[0]), set(fd[1])
        closure = compute_closure(lhs, fds)
        # If lhs is not a superkey, decompose
        if closure != set(relation):
            r1 = lhs | rhs
            r2 = set(relation) - rhs
            result.remove(relation)
            result.extend([r1, r2])
            # Recursively decompose
            return bcnf_decompose(r1, fds) + bcnf_decompose(r2, fds)
    return result
```

**ER to Relational Mapping:**

```python
def er_to_relational(entity, relationships):
    """Convert ER diagram to relational schema."""
    schema = {}
    
    # Strong entities → tables
    for e in entity['strong']:
        attrs = [a['name'] for a in e['attributes'] if not a.get('multi_valued')]
        pk = [a['name'] for a in e['attributes'] if a.get('primary_key')]
        schema[e['name']] = {'columns': attrs, 'pk': pk}
    
    # Weak entities → tables with owner FK
    for e in entity.get('weak', []):
        owner_fk = f"{e['owner']}_id"
        attrs = [owner_fk] + [a['name'] for a in e['attributes']]
        schema[e['name']] = {'columns': attrs, 'pk': [owner_fk, e['partial_key']]}
    
    # M:N relationships → junction table
    for r in relationships:
        if r['cardinality'] == 'M:N':
            attrs = [f"{r['entity1']}_id", f"{r['entity2']}_id"]
            schema[r['name']] = {'columns': attrs, 'pk': attrs}
    
    return schema
```

### Trade-offs

**Normalization vs Denormalization:**
- **Normalized:** No anomalies, smaller storage, slower joins
- **Denormalized:** Faster reads, redundant data, update anomalies

**Normal Form Selection:**
- **OLTP:** Higher normalization (3NF/BCNF) for write correctness
- **OLAP:** Lower normalization for read performance

### Applications

- **Schema Design:** Designing normalized schemas for applications
- **Data Warehousing:** Star/snowflake schemas for analytics
- **ORM Mapping:** Translating between objects and relations

---

## SQL

### Concept
Declarative query language for defining, manipulating, and controlling relational data.

### Principles

**DDL (Data Definition Language):**
```sql
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id),
    total DECIMAL(10,2) CHECK (total >= 0),
    status VARCHAR(20) DEFAULT 'pending',
    created_at TIMESTAMPTZ DEFAULT NOW(),
    INDEX idx_user_status (user_id, status)
);

CREATE INDEX CONCURRENTLY idx_orders_created 
ON orders USING btree (created_at DESC);

CREATE MATERIALIZED VIEW order_summary AS
SELECT user_id, COUNT(*) as order_count, SUM(total) as total_spent
FROM orders WHERE status = 'completed'
GROUP BY user_id;
```

**DML (Data Manipulation Language):**
```sql
-- Upsert (PostgreSQL)
INSERT INTO users (email, name) 
VALUES ('user@example.com', 'Alice')
ON CONFLICT (email) DO UPDATE SET name = EXCLUDED.name;

-- Window functions
SELECT 
    department,
    name,
    salary,
    RANK() OVER (PARTITION BY department ORDER BY salary DESC) as rank,
    SUM(salary) OVER (PARTITION BY department) as dept_total,
    salary - AVG(salary) OVER (PARTITION BY department) as diff_from_avg
FROM employees;

-- CTE (Common Table Expressions)
WITH RECURSIVE org_tree AS (
    SELECT id, name, manager_id, 1 as level
    FROM employees WHERE manager_id IS NULL
    UNION ALL
    SELECT e.id, e.name, e.manager_id, t.level + 1
    FROM employees e JOIN org_tree t ON e.manager_id = t.id
)
SELECT * FROM org_tree ORDER BY level;
```

**DCL (Data Control Language):**
```sql
GRANT SELECT, INSERT ON orders TO app_user;
REVOKE DELETE ON orders FROM app_user;
```

**JOIN Types:**
```sql
-- INNER JOIN: Only matching rows
SELECT o.id, u.name FROM orders o 
INNER JOIN users u ON o.user_id = u.id;

-- LEFT JOIN: All left + matching right (NULL if no match)
SELECT u.name, COUNT(o.id) as order_count
FROM users u LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.name;

-- FULL OUTER JOIN: All rows from both sides
SELECT u.name, o.id
FROM users u FULL OUTER JOIN orders o ON u.id = o.user_id;

-- CROSS JOIN: Cartesian product
SELECT d.name, e.name 
FROM departments d CROSS JOIN employees e;

-- LATERAL JOIN: Subquery can reference outer query
SELECT u.name, recent_orders.*
FROM users u,
LATERAL (SELECT * FROM orders WHERE user_id = u.id 
         ORDER BY created_at DESC LIMIT 3) recent_orders;
```

**Query Optimization:**
```sql
-- Analyze query plan
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT * FROM orders WHERE user_id = 123;

-- Common plan node types:
-- Seq Scan → full table scan
-- Index Scan → index lookup + table fetch
-- Index Only Scan → index-only (covering index)
-- Bitmap Heap Scan → bitmap of matching pages
-- Hash Join → build hash table on smaller relation
-- Merge Join → sorted merge (good for pre-sorted data)
-- Nested Loop → for small outer / indexed inner
```

### Algorithms & Code

**Query Plan Cost Estimator:**

```python
class QueryOptimizer:
    def __init__(self, stats):
        self.stats = stats  # Table statistics
    
    def estimate_sequential_scan_cost(self, table):
        n_pages = self.stats[table]['num_pages']
        n_tuples = self.stats[table]['num_tuples']
        return {
            'io_cost': n_pages,  # One I/O per page
            'cpu_cost': n_tuples * 0.01,  # Per-tuple evaluation
            'total': n_pages + n_tuples * 0.01
        }
    
    def estimate_index_scan_cost(self, table, index, selectivity):
        n_tuples = self.stats[table]['num_tuples']
        n_pages = self.stats[table]['num_pages']
        matching_tuples = n_tuples * selectivity
        
        # B+ tree height
        tree_height = 3  # Typical for millions of rows
        
        # Random I/O for matching tuples
        io_cost = tree_height + matching_tuples  # Approximate
        cpu_cost = matching_tuples * 0.01
        
        return {
            'io_cost': io_cost,
            'cpu_cost': cpu_cost,
            'total': io_cost + cpu_cost
        }
    
    def estimate_hash_join_cost(self, build_table, probe_table):
        build_pages = self.stats[build_table]['num_pages']
        probe_pages = self.stats[probe_table]['num_pages']
        build_tuples = self.stats[build_table]['num_tuples']
        probe_tuples = self.stats[probe_table]['num_tuples']
        
        # Build phase: read build table, create hash table
        build_cost = build_pages  # I/O
        hash_cost = build_tuples * 0.015  # Hash computation
        
        # Probe phase: read probe table, probe hash table
        probe_io = probe_pages
        probe_cpu = probe_tuples * 0.015
        
        return {
            'total': build_cost + hash_cost + probe_io + probe_cpu
        }
    
    def choose_join_order(self, tables, join_conditions):
        """Dynamic programming for join ordering."""
        n = len(tables)
        # dp[subset] = best plan for that subset of tables
        dp = {}
        
        # Base case: single tables
        for t in tables:
            frozenset_t = frozenset([t])
            dp[frozenset_t] = {
                'cost': self.estimate_sequential_scan_cost(t)['total'],
                'plan': f"Scan({t})"
            }
        
        # Fill DP table
        for size in range(2, n + 1):
            for subset in combinations(tables, size):
                subset = frozenset(subset)
                best_cost = float('inf')
                best_plan = None
                
                # Try all ways to split subset into two parts
                for s1, s2 in self._splits(subset):
                    join_cost = dp[s1]['cost'] + dp[s2]['cost'] + \
                                self._join_overhead(s1, s2)
                    if join_cost < best_cost:
                        best_cost = join_cost
                        best_plan = f"Join({dp[s1]['plan']}, {dp[s2]['plan']})"
                
                dp[subset] = {'cost': best_cost, 'plan': best_plan}
        
        return dp[frozenset(tables)]
    
    def _splits(self, subset):
        items = list(subset)
        for i in range(1, len(items)):
            for combo in combinations(items, i):
                yield frozenset(combo), subset - frozenset(combo)
    
    def _join_overhead(self, s1, s2):
        return 100  # Simplified
```

### Trade-offs

**Index Types:**
- **B+ Tree:** General purpose, range queries, O(log n)
- **Hash:** Exact match only, O(1) average
- **Bitmap:** Low-cardinality columns, read-heavy
- **GIN/GiST:** Full-text, array, geometric data

**Join Algorithms:**
| Algorithm | Best For | Memory | I/O Pattern |
|-----------|----------|--------|-------------|
| Nested Loop | Small outer, indexed inner | O(1) | Random |
| Hash Join | Equi-join, no order | O(build) | Sequential |
| Merge Join | Pre-sorted data | O(1) | Sequential |

### Applications

- **Application Development:** CRUD operations, reporting
- **Data Analytics:** Window functions, CTEs, aggregation
- **Database Administration:** Query tuning, index strategy

---

## Index Structures

### Concept
Data structures that enable efficient data retrieval without scanning entire tables.

### Principles

**B+ Tree:**
- Balanced tree with all data in leaf nodes
- Internal nodes: keys only (routing)
- Leaf nodes: keys + pointers to data (linked list)
- Typical order: 50-100 (fan-out)
- Height: O(log_fanout(N)), typically 3-4 levels for billions of rows

**B+ Tree Operations:**
- **Search:** Traverse from root, binary search within nodes → O(log n)
- **Insert:** Find leaf, insert, split if overflow
- **Delete:** Find leaf, delete, merge/redistribute if underflow

**Hash Index:**
- O(1) average lookup
- No range query support
- Collision handling: chaining or open addressing

**Specialized Indexes:**
- **Bitmap Index:** Bit vector per distinct value, efficient for AND/OR
- **GIN (Generalized Inverted Index):** For array/full-text search
- **GiST (Generalized Search Tree):** Extensible for custom data types
- **SP-GiST:** Space-partitioned trees (e.g., quadtree, radix tree)

**Covering Index:**
- Index contains all columns needed by query
- Enables "Index Only Scan" — no table lookup needed
- `CREATE INDEX idx_covering ON orders(user_id) INCLUDE (status, total);`

**Index Matching Rules:**
- **Leftmost Prefix:** Multi-column index `(a, b, c)` can serve:
  - `WHERE a = ?`
  - `WHERE a = ? AND b = ?`
  - `WHERE a = ? AND b = ? AND c = ?`
  - NOT: `WHERE b = ?` or `WHERE c = ?`

### Algorithms & Code

**B+ Tree Implementation:**

```python
class BPlusTreeNode:
    def __init__(self, order, leaf=False):
        self.order = order
        self.leaf = leaf
        self.keys = []
        self.children = []  # Pointers to child nodes (internal) or data (leaf)
        self.next_leaf = None  # Linked list for leaf nodes
        self.parent = None

class BPlusTree:
    def __init__(self, order=50):
        self.order = order
        self.root = BPlusTreeNode(order, leaf=True)
    
    def search(self, key):
        """Find the leaf node containing key."""
        node = self.root
        while not node.leaf:
            i = self._find_position(node.keys, key)
            node = node.children[i]
        return node
    
    def insert(self, key, value):
        """Insert key-value pair into the tree."""
        leaf = self.search(key)
        self._insert_into_leaf(leaf, key, value)
        
        if len(leaf.keys) >= self.order:
            self._split_leaf(leaf)
    
    def _insert_into_leaf(self, leaf, key, value):
        i = self._find_position(leaf.keys, key)
        leaf.keys.insert(i, key)
        leaf.children.insert(i, value)
    
    def _split_leaf(self, leaf):
        """Split an overflowing leaf node."""
        mid = len(leaf.keys) // 2
        
        new_leaf = BPlusTreeNode(self.order, leaf=True)
        new_leaf.keys = leaf.keys[mid:]
        new_leaf.children = leaf.children[mid:]
        new_leaf.next_leaf = leaf.next_leaf
        leaf.next_leaf = new_leaf
        
        leaf.keys = leaf.keys[:mid]
        leaf.children = leaf.children[:mid]
        
        promote_key = new_leaf.keys[0]
        self._insert_into_parent(leaf, promote_key, new_leaf)
    
    def _split_internal(self, node):
        """Split an overflowing internal node."""
        mid = len(node.keys) // 2
        promote_key = node.keys[mid]
        
        new_node = BPlusTreeNode(self.order, leaf=False)
        new_node.keys = node.keys[mid+1:]
        new_node.children = node.children[mid+1:]
        
        node.keys = node.keys[:mid]
        node.children = node.children[:mid+1]
        
        for child in new_node.children:
            child.parent = new_node
        
        self._insert_into_parent(node, promote_key, new_node)
    
    def _insert_into_parent(self, left, key, right):
        if left.parent is None:
            # Create new root
            new_root = BPlusTreeNode(self.order, leaf=False)
            new_root.keys = [key]
            new_root.children = [left, right]
            left.parent = new_root
            right.parent = new_root
            self.root = new_root
        else:
            parent = left.parent
            i = parent.children.index(left)
            parent.keys.insert(i, key)
            parent.children.insert(i + 1, right)
            right.parent = parent
            
            if len(parent.keys) >= self.order:
                self._split_internal(parent)
    
    def range_query(self, start, end):
        """Find all keys in [start, end]."""
        leaf = self.search(start)
        results = []
        while leaf:
            for i, key in enumerate(leaf.keys):
                if key > end:
                    return results
                if key >= start:
                    results.append((key, leaf.children[i]))
            leaf = leaf.next_leaf
        return results
    
    @staticmethod
    def _find_position(keys, key):
        lo, hi = 0, len(keys)
        while lo < hi:
            mid = (lo + hi) // 2
            if keys[mid] < key:
                lo = mid + 1
            else:
                hi = mid
        return lo
```

**Bitmap Index:**

```python
import numpy as np

class BitmapIndex:
    def __init__(self, values):
        self.distinct_values = sorted(set(values))
        self.bitmaps = {}
        for val in self.distinct_values:
            self.bitmaps[val] = np.array([1 if v == val else 0 for v in values], dtype=np.uint8)
    
    def lookup(self, value):
        return self.bitmaps.get(value, np.zeros(0, dtype=np.uint8))
    
    def and_query(self, val1, val2):
        return self.bitmaps[val1] & self.bitmaps[val2]
    
    def or_query(self, val1, val2):
        return self.bitmaps[val1] | self.bitmaps[val2]
    
    def count(self, value):
        return np.sum(self.bitmaps[value])
    
    # WAH (Word-Aligned Hybrid) compression
    def compress_bitmap(self, bitmap):
        """Run-length encode consecutive zeros/ones in word-sized chunks."""
        WORD_SIZE = 32
        compressed = []
        i = 0
        while i < len(bitmap):
            chunk = bitmap[i:i+WORD_SIZE]
            if len(chunk) == WORD_SIZE and np.all(chunk == chunk[0]):
                # Run: 1-bit flag (1=run) + 1-bit value + 30-bit count
                run_word = (1 << 31) | (chunk[0] << 30) | (1)  # Run of 1 word
                compressed.append(run_word)
                i += WORD_SIZE
            else:
                # Literal: 0-bit flag + 31-bit data
                literal = int(''.join(map(str, chunk[:WORD_SIZE])), 2)
                compressed.append(literal)
                i += WORD_SIZE
        return compressed
```

### Trade-offs

**B+ Tree vs Hash Index:**
- B+ Tree: Range queries, ordered output, higher write amplification
- Hash: Faster point lookups, no range support, resize overhead

**Bitmap Index:**
- Best for: Low-cardinality columns (gender, status), read-heavy
- Worst for: High-cardinality, write-heavy (expensive to update)

**Covering vs Regular Index:**
- Covering: Faster reads, larger index, slower writes
- Regular: Smaller, more general purpose

### Applications

- **OLTP Systems:** B+ tree indexes for fast point lookups
- **Data Warehousing:** Bitmap indexes for dimensional queries
- **Full-Text Search:** GIN/GiST indexes
- **Geospatial:** R-tree / GiST for spatial queries

---

## Transactions

### Concept
A transaction is a unit of work that must execute atomically and consistently, providing ACID guarantees.

### Principles

**ACID Properties:**
- **Atomicity:** All or nothing (undo logs)
- **Consistency:** Database moves from one valid state to another
- **Isolation:** Concurrent transactions don't interfere
- **Durability:** Committed data survives crashes (redo logs)

**Isolation Levels:**

| Level | Dirty Read | Non-Repeatable Read | Phantom Read |
|-------|-----------|---------------------|--------------|
| Read Uncommitted | Yes | Yes | Yes |
| Read Committed | No | Yes | Yes |
| Repeatable Read | No | No | Yes* |
| Serializable | No | No | No |

*MySQL InnoDB's Repeatable Read prevents phantom reads via next-key locking.

**Concurrency Problems:**
- **Dirty Read:** Reading uncommitted data from another transaction
- **Non-Repeatable Read:** Same query returns different rows (update/delete)
- **Phantom Read:** Same query returns additional rows (insert)
- **Write Skew:** Two transactions read overlapping data and make conflicting writes

**MVCC (Multi-Version Concurrency Control):**
- Each write creates a new version of a row
- Readers see a consistent snapshot (no blocking)
- Uses transaction IDs (txid) and visibility rules
- PostgreSQL: xmin/xmax columns, MVCC snapshot
- MySQL InnoDB: Undo log for rollback, read view

**Lock Types:**
- **Shared Lock (S):** Read lock, multiple readers allowed
- **Exclusive Lock (X):** Write lock, exclusive access
- **Intent Locks (IS/IX):** Hierarchical locking hint
- **Gap Lock:** Locks the gap between index records
- **Next-Key Lock:** Record lock + gap lock (InnoDB default)

**Two-Phase Locking (2PL):**
1. **Growing Phase:** Acquire locks, no releases
2. **Shrinking Phase:** Release locks, no acquisitions
- **Strict 2PL:** Hold all X locks until commit/abort
- **Rigorous 2PL:** Hold all locks until commit/abort

### Algorithms & Code

**MVCC Implementation (Simplified PostgreSQL):**

```python
import time

class RowVersion:
    def __init__(self, data, xmin, xmax=None):
        self.data = data
        self.xmin = xmin  # Creating transaction ID
        self.xmax = xmax  # Deleting transaction ID (None = still visible)

class Transaction:
    def __init__(self, txid):
        self.txid = txid
        self.status = 'active'
        self.snapshot = set()  # Active transactions at start

class MVCCStore:
    def __init__(self):
        self.table = {}  # key → list of RowVersions
        self.tx_counter = 0
        self.active_txs = {}
    
    def begin(self):
        self.tx_counter += 1
        tx = Transaction(self.tx_counter)
        tx.snapshot = set(self.active_txs.keys())
        self.active_txs[tx.txid] = tx
        return tx
    
    def read(self, tx, key):
        """Read the version visible to this transaction."""
        if key not in self.table:
            return None
        
        for version in reversed(self.table[key]):
            if self._visible(tx, version):
                return version.data
        return None
    
    def write(self, tx, key, value):
        """Create a new version of the row."""
        # Check for write-write conflict
        for version in self.table.get(key, []):
            if version.xmax is None and version.xmin != tx.txid:
                if version.xmin not in tx.snapshot:
                    raise Exception("Write-write conflict")
        
        new_version = RowVersion(value, tx.txid)
        if key not in self.table:
            self.table[key] = []
        self.table[key].append(new_version)
    
    def commit(self, tx):
        tx.status = 'committed'
        del self.active_txs[tx.txid]
    
    def abort(self, tx):
        tx.status = 'aborted'
        del self.active_txs[tx.txid]
    
    def _visible(self, tx, version):
        """Determine if a row version is visible to a transaction."""
        # Created by a committed transaction that started before us
        creator_committed = version.xmin not in self.active_txs
        if not creator_committed:
            return False
        if version.xmin in tx.snapshot:
            return False
        
        # Not deleted, or deleted by a transaction not yet committed
        if version.xmax is None:
            return True
        if version.xmax in self.active_txs:
            return True  # Deleter still active → version visible
        if version.xmax in tx.snapshot:
            return True  # Deleter started after our snapshot
        
        return False
```

**Two-Phase Commit (2PC):**

```python
class Coordinator:
    def __init__(self, participants):
        self.participants = participants
        self.log = []
    
    def commit_transaction(self, txid):
        """Phase 1: Prepare"""
        self.log.append(('prepare', txid))
        
        votes = []
        for participant in self.participants:
            try:
                vote = participant.prepare(txid)
                votes.append(vote)
            except Exception:
                votes.append('abort')
        
        """Phase 2: Commit or Abort"""
        if all(v == 'prepared' for v in votes):
            self.log.append(('commit', txid))
            for participant in self.participants:
                participant.commit(txid)
            return 'committed'
        else:
            self.log.append(('abort', txid))
            for participant in self.participants:
                participant.abort(txid)
            return 'aborted'

class Participant:
    def __init__(self):
        self.prepared = {}
        self.log = []
    
    def prepare(self, txid):
        """Prepare to commit. Write prepare record to log."""
        # Write undo + redo log records
        # Hold all locks
        self.log.append(('prepare', txid))
        self.prepared[txid] = True
        return 'prepared'
    
    def commit(self, txid):
        """Commit the transaction."""
        self.log.append(('commit', txid))
        # Release all locks
        del self.prepared[txid]
    
    def abort(self, txid):
        """Abort the transaction."""
        self.log.append(('abort', txid))
        # Undo all changes
        # Release all locks
        if txid in self.prepared:
            del self.prepared[txid]
```

**Optimistic Concurrency Control (OCC):**

```python
class OCCTransaction:
    def __init__(self, txid):
        self.txid = txid
        self.read_set = {}   # key → version_read
        self.write_set = {}  # key → new_value
    
    def read(self, store, key):
        version, value = store.read_with_version(key)
        self.read_set[key] = version
        return value
    
    def write(self, key, value):
        self.write_set[key] = value
    
    def commit(self, store):
        """Validate and commit."""
        # Validation phase: check read set versions haven't changed
        for key, expected_version in self.read_set.items():
            current_version, _ = store.read_with_version(key)
            if current_version != expected_version:
                return False  # Conflict, abort
        
        # Write phase: apply all writes
        for key, value in self.write_set.items():
            store.write(key, value, self.txid)
        
        return True
```

### Trade-offs

**Pessimistic vs Optimistic Concurrency:**
- **Pessimistic (2PL):** Block early, lower abort rate, deadlock risk
- **Optimistic (OCC):** No blocking, higher abort rate under contention

**MVCC vs 2PL:**
- **MVCC:** Readers don't block writers, requires vacuum/cleanup
- **2PL:** Simpler, readers block writers in strict mode

**Isolation Level Selection:**
- **Read Committed:** Good default, no phantom protection
- **Repeatable Read:** Stronger guarantees, may serialize less
- **Serializable:** Strongest, highest overhead

### Applications

- **Banking:** Transfer funds atomically across accounts
- **E-Commerce:** Inventory management, order processing
- **Reservation Systems:** Seat booking, hotel reservations
- **Multi-user Applications:** Concurrent edits without corruption

---

## Logging & Recovery

### Concept
Techniques to ensure durability of committed transactions and atomic rollback of aborted ones.

### Principles

**WAL (Write-Ahead Logging):**
1. Write log record to stable storage BEFORE writing data page to disk
2. Log records contain: transaction ID, operation type, before-image (undo), after-image (redo)
3. Log is sequential append-only → fast

**Log Record Types:**
- **START:** Transaction begins
- **UPDATE:** Page modification (old + new values)
- **COMMIT:** Transaction committed (force log to disk)
- **ABORT:** Transaction aborted
- **CHECKPOINT:** Record of dirty pages and active transactions
- **CLR (Compensation Log Record):** Undo operation record

**ARIES Recovery Algorithm (3 Phases):**

1. **Analysis Phase:**
   - Replay log from last checkpoint
   - Determine: dirty pages, active transactions at crash

2. **Redo Phase:**
   - Replay all logged operations (including aborted ones)
   - Reestablish state at crash moment
   - Use LSN (Log Sequence Number) to avoid redundant work

3. **Undo Phase:**
   - Undo all operations of uncommitted transactions
   - Write CLRs for undo operations (for idempotent recovery)

**Checkpoint Types:**
- **Sharp (Consistent):** Stop all transactions, flush all pages
- **Fuzzy:** Note dirty pages, continue processing; log checkpoint record

**Log Buffering Strategies:**
- **Force-at-Commit:** Flush log buffer on commit (durable but slow)
- **Group Commit:** Batch multiple commits' log flushes
- **No-Force:** Don't flush immediately (risk of lost commits)

### Algorithms & Code

**WAL Manager:**

```python
import struct
import os

class WALManager:
    def __init__(self, log_file):
        self.log_file = log_file
        self.lsn = 0
        self.active_txs = {}  # txid → first_lsn
        self.dirty_pages = {}  # page_id → recovery_lsn
    
    def append(self, txid, op_type, page_id, undo_data, redo_data):
        """Append a log record."""
        self.lsn += 1
        record = {
            'lsn': self.lsn,
            'txid': txid,
            'op_type': op_type,  # 'start', 'update', 'commit', 'abort', 'clr'
            'page_id': page_id,
            'undo': undo_data,  # Before-image
            'redo': redo_data,  # After-image
            'prev_lsn': self.active_txs.get(txid, 0)
        }
        self._write_record(record)
        
        if op_type == 'start':
            self.active_txs[txid] = self.lsn
        
        return self.lsn
    
    def commit(self, txid):
        """Commit transaction and force log to disk."""
        self.append(txid, 'commit', None, None, None)
        self._force_log()
        del self.active_txs[txid]
    
    def checkpoint(self):
        """Write fuzzy checkpoint record."""
        record = {
            'lsn': self.lsn + 1,
            'op_type': 'checkpoint',
            'active_txs': dict(self.active_txs),
            'dirty_pages': dict(self.dirty_pages)
        }
        self._write_record(record)
        self._force_log()
    
    def _write_record(self, record):
        """Serialize and append log record to file."""
        with open(self.log_file, 'ab') as f:
            data = str(record).encode() + b'\n'
            f.write(data)
    
    def _force_log(self):
        """Ensure log is flushed to stable storage."""
        os.fsync(os.open(self.log_file, os.O_RDONLY))
```

**ARIES Recovery:**

```python
class ARIESRecovery:
    def __init__(self, log_file, page_store):
        self.log_file = log_file
        self.page_store = page_store
    
    def recover(self):
        """3-phase ARIES recovery."""
        # Read all log records
        records = self._read_log()
        
        # Find last checkpoint
        checkpoint = self._find_last_checkpoint(records)
        
        # Phase 1: Analysis
        dirty_pages, active_txs = self._analysis(records, checkpoint)
        
        # Phase 2: Redo
        self._redo(records, dirty_pages)
        
        # Phase 3: Undo
        self._undo(records, active_txs)
    
    def _analysis(self, records, checkpoint):
        """Determine dirty pages and active transactions at crash."""
        dirty_pages = checkpoint.get('dirty_pages', {}) if checkpoint else {}
        active_txs = checkpoint.get('active_txs', {}) if checkpoint else {}
        committed_txs = set()
        
        start_idx = checkpoint['lsn'] if checkpoint else 0
        for rec in records:
            if rec['lsn'] < start_idx:
                continue
            
            if rec['op_type'] == 'start':
                active_txs[rec['txid']] = rec['lsn']
            elif rec['op_type'] == 'commit':
                committed_txs.add(rec['txid'])
                if rec['txid'] in active_txs:
                    del active_txs[rec['txid']]
            elif rec['op_type'] == 'update':
                dirty_pages[rec['page_id']] = rec['lsn']
        
        # Remove committed transactions
        active_txs = {k: v for k, v in active_txs.items() if k not in committed_txs}
        
        return dirty_pages, active_txs
    
    def _redo(self, records, dirty_pages):
        """Redo all updates for dirty pages."""
        for rec in records:
            if rec['op_type'] not in ('update', 'clr'):
                continue
            if rec['page_id'] not in dirty_pages:
                continue
            
            page = self.page_store.get(rec['page_id'])
            # Only redo if page LSN < record LSN
            if page is None or page.get('lsn', 0) < rec['lsn']:
                self.page_store.apply(rec['page_id'], rec['redo'], rec['lsn'])
    
    def _undo(self, records, active_txs):
        """Undo all operations of active (uncommitted) transactions."""
        # Collect all records to undo, in reverse LSN order
        to_undo = []
        for rec in reversed(records):
            if rec['txid'] in active_txs and rec['op_type'] in ('update',):
                to_undo.append(rec)
        
        for rec in to_undo:
            # Apply undo (restore before-image)
            self.page_store.apply(rec['page_id'], rec['undo'])
            # Write Compensation Log Record
            self._write_clr(rec)
    
    def _write_clr(self, original_rec):
        """Write a Compensation Log Record for undo."""
        clr = {
            'op_type': 'clr',
            'txid': original_rec['txid'],
            'page_id': original_rec['page_id'],
            'undo': original_rec['undo'],
            'redo': original_rec['undo'],  # Redo of undo
            'undo_next_lsn': original_rec.get('prev_lsn', 0)
        }
        # Write to log...
```

### Trade-offs

**Checkpoint Frequency:**
- Frequent: Faster recovery, more overhead during normal operation
- Infrequent: Less overhead, slower recovery

**Log Flushing:**
- Force-at-commit: Guaranteed durability, higher latency
- Group commit: Better throughput, slight latency increase

**Shadow Paging vs WAL:**
- Shadow Paging: Simple, no log needed, random I/O
- WAL: Sequential I/O, more complex, industry standard

### Applications

- **Database Systems:** Crash recovery for all major RDBMS
- **File Systems:** Journaling (ext4), copy-on-write (ZFS, Btrfs)
- **Message Queues:** Durable message delivery
- **Distributed Systems:** Write-ahead logs for state machines (Raft)

---

## Distributed Databases

### Concept
Databases that span multiple nodes, providing scalability, availability, and geographic distribution.

### Principles

**CAP Theorem:**
- **Consistency:** Linearizable reads
- **Availability:** Every request gets a response
- **Partition Tolerance:** System works despite network partitions
- **Pick 2 of 3:** In practice, CP or AP (network partitions are real)

**BASE (Opposite of ACID):**
- **Basically Available:** System responds (may be stale)
- **Soft State:** State may change without input
- **Eventual Consistency:** Converges to consistent state over time

**Replication:**
- **Primary-Secondary (Leader-Follower):** Writes go to leader, replicated to followers
- **Multi-Master:** Any node accepts writes, conflict resolution needed
- **Raft/Paxos:** Consensus-based replication for strong consistency

**Sharding (Partitioning):**
- **Hash Sharding:** `node = hash(key) % N`
- **Range Sharding:** Key ranges assigned to nodes
- **Consistent Hashing:** Virtual nodes on a ring, minimal data movement on resize

**Distributed Transactions:**
- **2PC:** Coordinator prepares all participants, then commits/aborts
  - Blocking problem: Coordinator crash after prepare → participants locked
- **TCC (Try-Confirm-Cancel):** Business-level compensation
- **Saga:** Chain of local transactions with compensating actions

**NewSQL Systems:**
- **TiDB:** MySQL-compatible, Raft-based, HTAP
- **CockroachDB:** PostgreSQL-compatible, Raft-based, serializable
- **Spanner:** Google's globally distributed, TrueTime API

**Redis:**
- **Data Structures:** String, List, Hash, Set, Sorted Set, Stream
- **Persistence:** RDB (snapshot) + AOF (append-only file)
- **Sentinel:** Automatic failover for high availability
- **Cluster:** 16384 hash slots across nodes

**MongoDB:**
- **Document Model:** BSON (binary JSON), flexible schema
- **Replica Set:** Primary + secondaries, automatic election
- **Sharding:** Config servers + shard servers + mongos routers

### Algorithms & Code

**Consistent Hashing:**

```python
import hashlib
import bisect

class ConsistentHash:
    def __init__(self, nodes, virtual_nodes=150):
        self.ring = {}  # hash → node
        self.sorted_keys = []
        self.virtual_nodes = virtual_nodes
        
        for node in nodes:
            for i in range(virtual_nodes):
                key = self._hash(f"{node}:{i}")
                self.ring[key] = node
                self.sorted_keys.append(key)
        
        self.sorted_keys.sort()
    
    def get_node(self, key):
        """Find the node responsible for a key."""
        if not self.ring:
            return None
        
        h = self._hash(key)
        idx = bisect.bisect_right(self.sorted_keys, h)
        if idx == len(self.sorted_keys):
            idx = 0
        
        return self.ring[self.sorted_keys[idx]]
    
    def add_node(self, node):
        """Add a new node to the ring."""
        for i in range(self.virtual_nodes):
            key = self._hash(f"{node}:{i}")
            self.ring[key] = node
            bisect.insort(self.sorted_keys, key)
    
    def remove_node(self, node):
        """Remove a node from the ring."""
        keys_to_remove = []
        for i in range(self.virtual_nodes):
            key = self._hash(f"{node}:{i}")
            keys_to_remove.append(key)
        
        for key in keys_to_remove:
            del self.ring[key]
            idx = bisect.bisect_left(self.sorted_keys, key)
            self.sorted_keys.pop(idx)
    
    @staticmethod
    def _hash(key):
        return int(hashlib.md5(key.encode()).hexdigest(), 16)
```

**Saga Pattern:**

```python
class SagaStep:
    def __init__(self, name, action, compensate):
        self.name = name
        self.action = action
        self.compensate = compensate

class SagaOrchestrator:
    def __init__(self):
        self.steps = []
        self.completed = []
    
    def add_step(self, name, action, compensate):
        self.steps.append(SagaStep(name, action, compensate))
    
    def execute(self):
        """Execute saga with compensation on failure."""
        for i, step in enumerate(self.steps):
            try:
                result = step.action()
                self.completed.append((step, result))
            except Exception as e:
                # Compensate all completed steps in reverse
                for completed_step, result in reversed(self.completed):
                    try:
                        completed_step.compensate(result)
                    except Exception:
                        # Log compensation failure for manual intervention
                        pass
                raise Exception(f"Saga failed at step '{step.name}': {e}")

# Example: E-commerce order
orchestrator = SagaOrchestrator()
orchestrator.add_step(
    'reserve_inventory',
    action=lambda: reserve_item('item-123', 1),
    compensate=lambda res: release_item('item-123', 1)
)
orchestrator.add_step(
    'charge_payment',
    action=lambda: charge_card('card-456', 99.99),
    compensate=lambda res: refund_card('card-456', 99.99)
)
orchestrator.add_step(
    'create_shipment',
    action=lambda: create_shipping_label('addr-789'),
    compensate=lambda res: cancel_shipping_label(res['label_id'])
)
orchestrator.execute()
```

**Redis Sentinel Simulation:**

```python
import time
import random

class RedisSentinel:
    def __init__(self, sentinels, master, replicas):
        self.sentinels = sentinels
        self.master = master
        self.replicas = replicas
        self.quorum = len(sentinels) // 2 + 1
    
    def monitor(self):
        """Periodically check master health."""
        while True:
            if not self._is_alive(self.master):
                self._failover()
            time.sleep(1)
    
    def _is_alive(self, node):
        """Check if node responds to PING."""
        return random.random() > 0.1  # 90% uptime simulation
    
    def _failover(self):
        """Perform automatic failover."""
        # Phase 1: Sentinels agree master is down
        votes = sum(1 for s in self.sentinels 
                    if not self._is_alive(self.master))
        
        if votes < self.quorum:
            return  # Not enough agreement
        
        # Phase 2: Elect new master from replicas
        # Choose replica with highest replication offset
        best_replica = max(self.replicas, 
                          key=lambda r: r.get('offset', 0))
        
        # Phase 3: Promote replica to master
        print(f"Failover: promoting {best_replica} to master")
        self.replicas.remove(best_replica)
        self.replicas.append(self.master)
        self.master = best_replica
        
        # Phase 4: Reconfigure other replicas
        for replica in self.replicas:
            replica['master'] = self.master
    
    def get_master(self):
        return self.master
```

### Trade-offs

**Consistency vs Availability:**
- **CP (Consistent):** Block during partitions, always correct
- **AP (Available):** Respond during partitions, may be stale

**Sharding Strategy:**
- **Hash:** Even distribution, no range queries
- **Range:** Efficient range queries, hotspot risk
- **Consistent Hash:** Minimal redistribution on resize

**Transaction Model:**
- **2PC:** Strong consistency, blocking, vulnerable to coordinator failure
- **TCC:** Business-level, requires compensation logic
- **Saga:** Long-running transactions, eventual consistency

### Applications

- **E-Commerce:** Product catalogs, shopping carts, order processing
- **Social Media:** User profiles, feeds, messaging
- **IoT:** Time-series data, device state management
- **Financial Services:** Transaction processing, fraud detection

---

## Summary

Key takeaways for database systems:

1. **Relational Model:** Normalization eliminates anomalies; BCNF is ideal for OLTP
2. **SQL:** Master window functions, CTEs, and query plans for optimization
3. **Index Structures:** B+ tree is the workhorse; specialized indexes for specific workloads
4. **Transactions:** ACID via locking (2PL) or MVCC; isolation levels trade consistency for concurrency
5. **Recovery:** WAL + ARIES is the industry standard for crash recovery
6. **Distributed:** CAP drives architectural choices; consistent hashing and Raft enable scalable systems
