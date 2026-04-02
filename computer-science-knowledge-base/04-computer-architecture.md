# Computer Architecture - SKILL.md

## Overview
This skill covers advanced computer architecture concepts at the graduate level, focusing on modern processor design, memory hierarchy, and performance optimization techniques.

---

## Data Representation

### Concept
Understanding how different data types are represented in binary at the hardware level.

### Principles

**Integer Representation:**
- **Sign-Magnitude (原码):** MSB indicates sign, remaining bits magnitude
  - Range: -(2^(n-1)-1) to +(2^(n-1)-1)
  - Two zeros: +0 and -0
- **One's Complement (反码):** Invert all bits for negative numbers
  - Range: -(2^(n-1)-1) to +(2^(n-1)-1)
  - Two zeros: +0 and -0
- **Two's Complement (补码):** Invert bits and add 1 for negative
  - Range: -2^(n-1) to +(2^(n-1)-1)
  - Unique zero, simplifies arithmetic

**Floating Point (IEEE 754):**
- **Single Precision (32-bit):** 1 sign + 8 exponent + 23 mantissa
- **Double Precision (64-bit):** 1 sign + 11 exponent + 52 mantissa
- **Special Values:** +∞, -∞, NaN, ±0, denormals
- **Exponent Bias:** 127 for single, 1023 for double

**Character Encoding:**
- **ASCII:** 7-bit, 128 characters
- **Unicode:** Universal character set, 1,114,112 code points
- **UTF-8:** Variable-length (1-4 bytes), ASCII-compatible
  - 0xxxxxxx → 1 byte
  - 110xxxxx 10xxxxxx → 2 bytes
  - 1110xxxx 10xxxxxx 10xxxxxx → 3 bytes
  - 11110xxx 10xxxxxx 10xxxxxx 10xxxxxx → 4 bytes

### Algorithms & Code

**Two's Complement Arithmetic:**

```c
// Signed overflow detection
bool add_overflow(int32_t a, int32_t b) {
    int32_t result = a + b;
    // Overflow if signs differ but result sign differs from operands
    return ((a ^ b) & (a ^ result)) >> 31;
}

// Multiplication using shift-add
int32_t multiply(int32_t a, int32_t b) {
    int32_t result = 0;
    bool negative = (a < 0) ^ (b < 0);
    a = abs(a);
    b = abs(b);
    
    while (b > 0) {
        if (b & 1) result += a;
        a <<= 1;
        b >>= 1;
    }
    return negative ? -result : result;
}
```

**IEEE 754 to Decimal Conversion:**

```python
def ieee754_to_float(binary_str):
    sign = int(binary_str[0])
    exponent_bits = binary_str[1:9]
    mantissa_bits = binary_str[9:]
    
    if exponent_bits == '00000000':  # Denormalized
        exponent = -126
        mantissa = sum(int(b) * 2**-(i+1) for i, b in enumerate(mantissa_bits))
    elif exponent_bits == '11111111':  # Special values
        if mantissa_bits == '00000000000000000000000':
            return float('-inf') if sign else float('inf')
        else:
            return float('nan')
    else:  # Normalized
        exponent = int(exponent_bits, 2) - 127
        mantissa = 1.0 + sum(int(b) * 2**-(i+1) for i, b in enumerate(mantissa_bits))
    
    return ((-1) ** sign) * (2 ** exponent) * mantissa
```

### Trade-offs

**Integer Representations:**
| Method | Pros | Cons |
|--------|------|------|
| Sign-Magnitude | Intuitive, easy comparison | Two zeros, complex arithmetic |
| One's Complement | Simple negation | Two zeros, carry propagation |
| Two's Complement | Unique zero, simple arithmetic | Asymmetric range |

**Floating Point Trade-offs:**
- **Precision vs Range:** More mantissa bits = higher precision, less range
- **Denormals:** Gradual underflow but performance cost
- **Subnormal flush-to-zero:** Faster but loses precision

### Applications

- **Numerical Computing:** Scientific simulations, financial calculations
- **Graphics:** Color representation, coordinate transformations
- **Network Protocols:** Big-endian vs little-endian byte ordering
- **Cryptography:** Modular arithmetic, big integers

---

## CPU Architecture

### Concept
Understanding modern processor design principles and instruction set architectures.

### Principles

**RISC vs CISC:**
- **RISC (Reduced Instruction Set Computer):** Simple, uniform instructions, load-store architecture (ARM, RISC-V, MIPS)
- **CISC (Complex Instruction Set Computer):** Complex, variable-length instructions, memory operands (x86)
- **Trade-off:** RISC better for pipelining, CISC higher code density

**Instruction Set Comparison:**

| Feature | x86 (CISC) | ARM (RISC) | RISC-V |
|---------|-----------|-----------|--------|
| Register count | 16 (32-bit) / 16+ (64-bit) | 16 (32-bit) / 31 (64-bit) | 32 (RV32I) / 31 (RV64I) |
| Instruction length | Variable (1-15 bytes) | Fixed (4 bytes) | Fixed (2/4/6 bytes) |
| Addressing modes | 10+ | 3-5 | Minimal |
| Endianness | Little | Bi-endian | Little |

**Pipeline Stages (5-stage classic):**
1. **IF (Instruction Fetch):** Fetch instruction from cache/memory
2. **ID (Instruction Decode):** Decode opcode, read registers
3. **EX (Execute):** ALU operations, address calculation
4. **MEM (Memory Access):** Load/store operations
5. **WB (Write Back):** Write results to register file

**Advanced Pipeline Techniques:**
- **Superscalar:** Issue multiple instructions per cycle (2-8 wide)
- **Out-of-Order Execution:** Dynamic scheduling, rename registers
- **Branch Prediction:** Predict branch direction and target
- **Speculative Execution:** Execute predicted path, rollback on misprediction
- **SIMD:** Single Instruction Multiple Data (SSE, AVX, NEON)

**Pipeline Hazards:**
- **Data Hazards:**
  - RAW (Read After Write): True dependency
  - WAR (Write After Read): Anti-dependency
  - WAW (Write After Write): Output dependency
- **Control Hazards:** Branch mispredictions
- **Structural Hazards:** Resource contention

**Forwarding (Bypassing):**
```text
   EX    MEM    WB
    |      |      |
    v      v      v
[ADD R1, R2, R3]
           [SUB R4, R1, R5]  ← Forward R1 from EX/MEM
                  [MUL R6, R1, R7]  ← Forward R1 from MEM/WB
```

**Branch Prediction:**
- **Static:** Always taken, not taken, profile-based
- **Dynamic:** 2-bit saturating counter, bimodal
- **Global History:** GShare (XOR PC with history)
- **Local History:** Per-branch predictors
- **Tournament:** Combine local and global
- **BTB (Branch Target Buffer):** Cache branch targets

### Algorithms & Code

**2-bit Saturating Counter:**

```python
class TwoBitPredictor:
    def __init__(self):
        # 00: strongly not taken
        # 01: weakly not taken
        # 10: weakly taken
        # 11: strongly taken
        self.state = {0}  # State per address
    
    def predict(self, pc):
        return self.state.get(pc, 1) >= 2
    
    def update(self, pc, taken):
        current = self.state.get(pc, 1)
        if taken and current < 3:
            current += 1
        elif not taken and current > 0:
            current -= 1
        self.state[pc] = current
```

**TOMASULO Algorithm (Out-of-Order Execution):**

```python
class Tomasulo:
    def __init__(self, num_res_stations, num_registers):
        self.reservation_stations = [RS() for _ in range(num_res_stations)]
        self.register_file = [{'tag': None, 'value': 0} for _ in range(num_registers)]
        self.common_data_bus = None
        self.reorder_buffer = ROB()
    
    def issue(self, op, rd, rs1, rs2):
        # Find free reservation station
        rs = self.find_free_rs(op)
        rs.busy = True
        rs.op = op
        rs.vj = self.get_value(rs1)
        rs.vk = self.get_value(rs2)
        rs.qj = self.get_tag(rs1)
        rs.qk = self.get_tag(rs2)
        
        # Map destination to reservation station
        self.register_file[rd]['tag'] = rs.id
        self.register_file[rd]['value'] = None
    
    def execute(self, rs):
        # Execute when both operands ready
        if rs.qj is None and rs.qk is None:
            rs.result = rs.compute(rs.vj, rs.vk)
            rs.ready = True
    
    def writeback(self, rs):
        # Broadcast result on CDB
        self.common_data_bus = (rs.id, rs.result)
        
        # Update registers waiting for this result
        for reg in self.register_file:
            if reg['tag'] == rs.id:
                reg['tag'] = None
                reg['value'] = rs.result
        
        # Update reservation stations
        for station in self.reservation_stations:
            if station.qj == rs.id:
                station.qj = None
                station.vj = rs.result
            if station.qk == rs.id:
                station.qk = None
                station.vk = rs.result
```

**SIMD Operations (AVX2):**

```c
#include <immintrin.h>

// Vector addition of 8 floats
void vector_add_float(const float* a, const float* b, float* result, int n) {
    int i = 0;
    for (; i + 8 <= n; i += 8) {
        __m256 va = _mm256_load_ps(a + i);
        __m256 vb = _mm256_load_ps(b + i);
        __m256 vr = _mm256_add_ps(va, vb);
        _mm256_store_ps(result + i, vr);
    }
    // Handle remaining elements
    for (; i < n; i++) {
        result[i] = a[i] + b[i];
    }
}

// Horizontal sum of 8 floats
float horizontal_sum_256(__m256 v) {
    __m128 vlow = _mm256_castps256_ps128(v);
    __m128 vhigh = _mm256_extractf128_ps(v, 1);
    __m128 sum = _mm_add_ps(vlow, vhigh);
    sum = _mm_hadd_ps(sum, sum);
    sum = _mm_hadd_ps(sum, sum);
    return _mm_cvtss_f32(sum);
}
```

### Trade-offs

**Superscalar Width:**
- **Narrow (2-wide):** Low power, low area, moderate IPC
- **Wide (8+ wide):** High IPC but diminishing returns, power hungry

**Branch Prediction:**
- **Simple:** Low latency, low power, lower accuracy
- **Complex (Tournament):** Higher accuracy, more power, larger hardware

**Out-of-Order:**
- **In-order:** Simple, low power, easier verification
- **Out-of-order:** Higher performance, complex, power hungry

### Applications

- **High-Performance Computing:** Scientific simulations
- **Mobile Devices:** Power-efficient ARM cores
- **Embedded Systems:** RISC-V microcontrollers
- **Machine Learning:** SIMD for matrix operations

---

## Memory Hierarchy

### Concept
Multi-level storage system exploiting locality of reference to optimize performance.

### Principles

**Storage Hierarchy:**
```
Registers (1-4 KB)
    ↓ 1 cycle
L1 Cache (32-64 KB) - Instructions + Data
    ↓ 4-10 cycles
L2 Cache (256-512 KB) - Unified
    ↓ 10-20 cycles
L3 Cache (8-32 MB) - Shared
    ↓ 40-75 cycles
Main Memory (8-64 GB) - DRAM
    ↓ 100-300 cycles
SSD (256 GB - 4 TB) - Flash
    ↓ 100,000+ cycles
HDD (1-10 TB) - Magnetic
```

**Locality Principles:**
- **Temporal Locality:** Recently accessed items likely accessed again
- **Spatial Locality:** Items near recently accessed items likely accessed

**Cache Organization:**

| Organization | Complexity | Hit Time | Conflict Misses |
|--------------|------------|----------|-----------------|
| Direct Mapped | Low | Minimal | High |
| Set-Associative (2-8 way) | Medium | Moderate | Medium |
| Fully Associative | High | Slow | Low |

**Cache Performance Metrics:**
- **Hit Rate (h):** Fraction of accesses that hit in cache
- **Miss Rate (m):** 1 - h
- **Average Access Time:** h × Hit Time + m × Miss Penalty
- **AMAT:** h × L1 + (1-h) × L2 ≈ L1 + m × Miss Penalty

**Cache Replacement Policies:**
- **LRU (Least Recently Used):** Replace least recently used
- **Random:** Uniform random selection
- **Pseudo-LRU:** Approximate LRU with less hardware
- **PLRU (Tree-based):** O(log n) comparisons

**Write Policies:**
- **Write-Through:** Write to cache and memory on every store
- **Write-Back:** Write to cache, defer memory write on eviction
  - Need dirty bit to track modified lines

**Cache Coherency (MESI Protocol):**

| State | Description |
|-------|-------------|
| M (Modified) | Dirty, exclusive to this cache |
| E (Exclusive) | Clean, exclusive to this cache |
| S (Shared) | Clean, may exist in other caches |
| I (Invalid) | Line not valid |

**MOESI Protocol (adds Owner state):**
- **O (Owner):** Dirty but shared with other caches (only O can write back)

**NUMA (Non-Uniform Memory Access):**
- Each processor has local memory
- Access to remote memory has higher latency
- OS must schedule processes to maximize local accesses

### Algorithms & Code

**Cache Simulation:**

```python
class Cache:
    def __init__(self, size, line_size, associativity, policy='lru'):
        self.num_sets = size // (line_size * associativity)
        self.line_size = line_size
        self.associativity = associativity
        self.policy = policy
        self.sets = [[None] * associativity for _ in range(self.num_sets)]
        self.hits = 0
        self.misses = 0
    
    def access(self, address):
        block_addr = address // self.line_size
        set_idx = block_addr % self.num_sets
        tag = block_addr // self.num_sets
        
        # Check for hit
        for i, line in enumerate(self.sets[set_idx]):
            if line and line['tag'] == tag:
                self.hits += 1
                self.update_access_order(set_idx, i)
                return 'hit'
        
        # Miss - find replacement
        self.misses += 1
        replace_idx = self.find_replace(set_idx)
        self.sets[set_idx][replace_idx] = {'tag': tag, 'timestamp': 0}
        self.update_access_order(set_idx, replace_idx)
        return 'miss'
    
    def find_replace(self, set_idx):
        for i, line in enumerate(self.sets[set_idx]):
            if line is None:
                return i
        
        if self.policy == 'lru':
            # Find least recently used
            min_time = min(line['timestamp'] for line in self.sets[set_idx])
            return i for i, line in enumerate(self.sets[set_idx]) if line['timestamp'] == min_time
        elif self.policy == 'random':
            return random.randint(0, self.associativity - 1)
    
    def update_access_order(self, set_idx, used_idx):
        for i, line in enumerate(self.sets[set_idx]):
            if line:
                if i == used_idx:
                    line['timestamp'] = 0
                else:
                    line['timestamp'] += 1
```

**TLB (Translation Lookaside Buffer):**

```python
class TLB:
    def __init__(self, num_entries):
        self.entries = [{'vpn': None, 'ppn': None, 'valid': False, 'timestamp': 0} 
                        for _ in range(num_entries)]
    
    def lookup(self, vpn):
        for entry in self.entries:
            if entry['valid'] and entry['vpn'] == vpn:
                return entry['ppn']
        return None
    
    def insert(self, vpn, ppn):
        # Find invalid entry or LRU
        invalid_idx = None
        max_time = -1
        lru_idx = 0
        
        for i, entry in enumerate(self.entries):
            if not entry['valid']:
                invalid_idx = i
                break
            if entry['timestamp'] > max_time:
                max_time = entry['timestamp']
                lru_idx = i
        
        if invalid_idx is not None:
            idx = invalid_idx
        else:
            idx = lru_idx
        
        self.entries[idx] = {'vpn': vpn, 'ppn': ppn, 'valid': True, 'timestamp': 0}
        
        # Update timestamps
        for entry in self.entries:
            if entry['valid'] and self.entries.index(entry) != idx:
                entry['timestamp'] += 1
```

**Address Translation:**

```c
typedef struct {
    unsigned int valid;
    unsigned int dirty;
    unsigned int referenced;
    unsigned int frame_num;
} PageTableEntry;

// Multi-level page table (4-level x86-64)
unsigned long translate_address(unsigned long virt_addr, 
                                 PageTableEntry* pml4,
                                 PageTableEntry* pdpt,
                                 PageTableEntry* pd,
                                 PageTableEntry* pt) {
    
    // Extract page offsets
    unsigned long pml4_idx = (virt_addr >> 39) & 0x1FF;
    unsigned long pdpt_idx = (virt_addr >> 30) & 0x1FF;
    unsigned long pd_idx = (virt_addr >> 21) & 0x1FF;
    unsigned long pt_idx = (virt_addr >> 12) & 0x1FF;
    unsigned long offset = virt_addr & 0xFFF;
    
    // Walk page table
    if (!pml4[pml4_idx].valid) return 0;
    
    if (!pdpt[pdpt_idx].valid) return 0;
    
    if (!pd[pd_idx].valid) return 0;
    
    if (!pt[pt_idx].valid) return 0;
    
    // Construct physical address
    return (pt[pt_idx].frame_num << 12) | offset;
}
```

### Trade-offs

**Cache Size vs Access Time:**
- Larger cache = lower miss rate but higher hit time and power

**Associativity:**
- Higher associativity = fewer conflict misses but more power and complexity

**Line Size:**
- Larger line = better spatial locality but more bandwidth waste

**Coherence Protocol:**
- **MESI:** Simpler, less traffic for private data
- **MOESI:** Better for shared data, more complex state machine

### Applications

- **System Programming:** Cache-aware algorithms, memory allocators
- **Database Systems:** Buffer pool design, lock-free data structures
- **Graphics:** Texture caches, frame buffers
- **Virtual Machines:** Nested page tables, memory ballooning

---

## Memory System

### Concept
Physical memory organization and DRAM technology fundamentals.

### Principles

**SRAM vs DRAM:**

| Characteristic | SRAM | DRAM |
|----------------|------|------|
| Storage | 6 transistors per bit | 1 transistor + 1 capacitor |
| Speed | Fast (1-10 ns) | Slower (50-100 ns) |
| Density | Low | High |
| Cost | High | Low |
| Volatility | Volatile | Volatile (needs refresh) |
| Refresh | Not needed | Every 64ms |

**DRAM Organization:**
- **Banks:** Independent subarrays that can be accessed in parallel
- **Rows / Wordlines:** Activated together
- **Columns / Bitlines:** Select bits from active row
- **RAS-CAS Timing:** Activate row, then read/write column

**DRAM Commands:**
1. **ACT (Activate):** Open a row (tRCD delay)
2. **READ/WRITE:** Read or write data (CL / CWL delay)
3. **PRE (Precharge):** Close row (tRP delay)
4. **REF (Refresh):** Refresh all rows in bank (tRFC delay)

**Memory Bandwidth Calculation:**
```
Bandwidth = (Data Rate per Pin) × (Number of Pins) × (Banks) / (Burst Length)
          = (MT/s) × (Bus Width / 8) × (Banks)
```

**DDR Generations:**

| Generation | Voltage | Transfer Rate | I/O Bus Clock |
|------------|---------|----------------|----------------|
| DDR4 | 1.2V | 1600-3200 MT/s | 800-1600 MHz |
| DDR5 | 1.1V | 4800-6400+ MT/s | 2400-3200 MHz |
| LPDDR4 | 1.1V | 1600-4266 MT/s | 800-2133 MHz |
| LPDDR5 | 0.9V | 5500-8533 MT/s | 2750-4266 MHz |

**DDR5 Improvements:**
- Two 32-bit sub-channels per DIMM
- On-die ECC
- Higher density (up to 64GB per die)
- Better power management

**Virtual Memory:**
- **Page Size:** Typically 4KB (can be 2MB/1GB huge pages)
- **Page Table:** Maps virtual to physical addresses
- **TLB:** Caches recent translations
- **Page Fault:** Exception on invalid access

### Algorithms & Code

**DRAM Access Pattern Optimizer:**

```python
def optimize_dram_access(matrix_a, matrix_b):
    """
    Optimize for DRAM row buffer locality.
    Access elements sequentially within rows.
    """
    rows = len(matrix_a)
    cols = len(matrix_a[0])
    
    # Bad: Random access causes frequent row activations
    def bad_access():
        result = 0
        for j in range(cols):
            for i in range(rows):
                result += matrix_a[i][j]
        return result
    
    # Good: Sequential access utilizes row buffer
    def good_access():
        result = 0
        for i in range(rows):
            for j in range(cols):
                result += matrix_a[i][j]
        return result
    
    return good_access()
```

**Page Replacement Algorithm (Clock/Second-Chance):**

```python
class ClockPageReplacement:
    def __init__(self, num_frames):
        self.frames = [None] * num_frames
        self.ref_bits = [0] * num_frames
        self.clock_hand = 0
        self.hits = 0
        self.misses = 0
    
    def access(self, page_num):
        # Check if page is in frames
        for i, frame in enumerate(self.frames):
            if frame == page_num:
                self.hits += 1
                self.ref_bits[i] = 1
                return 'hit'
        
        # Page fault - find replacement
        self.misses += 1
        
        # Find page with ref_bit = 0
        while True:
            if self.ref_bits[self.clock_hand] == 0:
                self.frames[self.clock_hand] = page_num
                self.clock_hand = (self.clock_hand + 1) % len(self.frames)
                break
            else:
                # Give second chance
                self.ref_bits[self.clock_hand] = 0
                self.clock_hand = (self.clock_hand + 1) % len(self.frames)
        
        return 'miss'
```

**NUMA-Aware Allocation:**

```c
#include <numa.h>
#include <numaif.h>

// Allocate memory on specific NUMA node
void* numa_alloc_local(size_t size, int node) {
    void* ptr = numa_alloc_onnode(size, node);
    if (ptr == NULL) {
        // Fallback to local node
        ptr = numa_alloc_local(size);
    }
    return ptr;
}

// Move memory to preferred node
void move_memory_to_node(void* addr, size_t size, int target_node) {
    int pages = size / numa_pagesize();
    void** pages_arr = calloc(pages, sizeof(void*));
    int* nodes_arr = calloc(pages, sizeof(int));
    int* status_arr = calloc(pages, sizeof(int));
    
    // Prepare page list
    for (int i = 0; i < pages; i++) {
        pages_arr[i] = (void*)((uintptr_t)addr + i * numa_pagesize());
        nodes_arr[i] = target_node;
    }
    
    // Move pages
    numa_move_pages(0, pages, pages_arr, nodes_arr, status_arr, MPOL_MF_MOVE);
    
    free(pages_arr);
    free(nodes_arr);
    free(status_arr);
}
```

### Trade-offs

**DRAM vs SRAM:**
- SRAM: Fast, expensive, used for cache
- DRAM: Slower, cheap, used for main memory

**Row Buffer Size:**
- Larger row buffer = better spatial locality but higher power

**Page Size:**
- Larger pages = smaller page tables but more internal fragmentation
- Huge pages (2MB, 1GB) reduce TLB misses but waste memory

### Applications

- **Memory Controllers:** Scheduling, power management
- **Operating Systems:** Page replacement, NUMA scheduling
- **Databases:** Buffer pools, lock-free structures
- **High-Frequency Trading:** Low-latency memory access patterns

---

## Bus & Interconnect

### Concept
Communication protocols connecting processor, memory, and peripherals.

### Principles

**Bus Classification:**
- **System Bus:** Connects CPU, memory, high-speed peripherals
- **I/O Bus:** Connects peripherals to system bus
- **Point-to-Point:** Direct connections (HyperTransport, QPI)

**Bus Operations:**
- **Arbitration:** Which master gets control
- **Address Phase:** Send target address
- **Data Phase:** Transfer data
- **Burst:** Multiple consecutive transfers

**AMBA (Advanced Microcontroller Bus Architecture):**

| Protocol | Type | Use Case |
|----------|------|----------|
| AHB | High-performance | System bus, memory, DMA |
| APB | Low-power | Peripherals, registers |
| AXI | Advanced | High-throughput, multiple masters |

**AXI Features:**
- Separate read/write channels
- Outstanding transactions
- Burst transfers (up to 256 beats)
- Quality of Service (QoS)
- Low-power interface

**PCIe (PCI Express):**
- **Lanes:** Each lane is a full-duplex serial link
- **Width:** x1, x4, x8, x16 lanes
- **Generations:**
  - Gen 3: 8 GT/s, ~985 MB/s per lane
  - Gen 4: 16 GT/s, ~1969 MB/s per lane
  - Gen 5: 32 GT/s, ~3938 MB/s per lane
- **Protocol Layers:**
  - Transaction Layer: Packets, credits
  - Data Link Layer: ACK/NAK, replay
  - Physical Layer: Encoding, lane training

**DMA (Direct Memory Access):**
- CPU initiates transfer
- DMA controller handles data movement
- CPU free during transfer
- Interrupt signals completion

### Algorithms & Code

**AXI Bus Transaction:**

```python
class AXITransaction:
    def __init__(self, addr, data, size, burst=0, cache=0):
        self.addr = addr
        self.data = data
        self.size = size  # 0=8-bit, 1=16-bit, 2=32-bit, 3=64-bit
        self.burst = burst  # 0=fixed, 1=incr, 2=wrap
        self.cache = cache  # Cache attributes
        self.resp = 'OKAY'
    
    def is_aligned(self):
        return self.addr % (1 << self.size) == 0

class AXIBus:
    def __init__(self):
        self.read_queue = []
        self.write_queue = []
        self.slaves = {}
    
    def register_slave(self, addr_range, handler):
        self.slaves[addr_range] = handler
    
    def read(self, transaction):
        self.read_queue.append(transaction)
        # In hardware, this would be parallel
        return self._process_read(transaction)
    
    def write(self, transaction):
        self.write_queue.append(transaction)
        self._process_write(transaction)
    
    def _find_slave(self, addr):
        for addr_range, handler in self.slaves.items():
            if addr_range[0] <= addr < addr_range[1]:
                return handler
        return None
    
    def _process_read(self, trans):
        slave = self._find_slave(trans.addr)
        if slave:
            return slave.read(trans)
        else:
            trans.resp = 'DECERR'
            return None
```

**PCIe Configuration Space Access:**

```c
// PCIe configuration space access via MMIO
#define PCIE_CONFIG_ADDR 0xCF8
#define PCIE_CONFIG_DATA 0xCFC

uint32_t pcie_config_read(uint8_t bus, uint8_t device, uint8_t function, uint8_t offset) {
    uint32_t addr = 0x80000000 |  // Enable bit
                   ((bus & 0xFF) << 16) |
                   ((device & 0x1F) << 11) |
                   ((function & 0x07) << 8) |
                   (offset & 0xFC);
    
    outl(PCIE_CONFIG_ADDR, addr);
    return inl(PCIE_CONFIG_DATA);
}

void pcie_config_write(uint8_t bus, uint8_t device, uint8_t function, 
                        uint8_t offset, uint32_t value) {
    uint32_t addr = 0x80000000 |
                   ((bus & 0xFF) << 16) |
                   ((device & 0x1F) << 11) |
                   ((function & 0x07) << 8) |
                   (offset & 0xFC);
    
    outl(PCIE_CONFIG_ADDR, addr);
    outl(PCIE_CONFIG_DATA, value);
}
```

**DMA Transfer:**

```c
typedef struct {
    uint32_t src_addr;
    uint32_t dst_addr;
    uint32_t length;
    uint32_t control;  // Enable, direction, interrupt
} DMAChannel;

void dma_transfer(DMAChannel* channel) {
    // Configure channel
    channel->control |= DMA_ENABLE;
    
    // Start transfer
    channel->control |= DMA_START;
    
    // Wait for completion (polling or interrupt)
    while (!(channel->control & DMA_DONE));
    
    // Clear done flag
    channel->control &= ~DMA_DONE;
}

// Scatter-gather DMA
typedef struct {
    uint32_t src_addr;
    uint32_t dst_addr;
    uint32_t length;
} DMAScatterGatherEntry;

void dma_scatter_gather(DMAScatterGatherEntry* entries, int count) {
    for (int i = 0; i < count; i++) {
        DMAChannel channel = {
            .src_addr = entries[i].src_addr,
            .dst_addr = entries[i].dst_addr,
            .length = entries[i].length,
            .control = DMA_ENABLE | DMA_SG_MODE
        };
        dma_transfer(&channel);
    }
}
```

### Trade-offs

**Bus Width vs Frequency:**
- Wider bus = more parallelism but higher pin count and cost
- Higher frequency = higher throughput but more signal integrity issues

**Arbitration Schemes:**
- **Fixed Priority:** Simple but can starve low-priority masters
- **Round Robin:** Fair but higher latency
- **Weighted Round Robin:** Balance fairness and priority

**DMA vs CPU:**
- DMA: Offloads CPU, efficient for large transfers
- CPU: Simpler for small, infrequent transfers

### Applications

- **SoC Design:** ARM cores with AMBA buses
- **High-Performance Computing:** InfiniBand, NVLink
- **Storage:** NVMe over PCIe
- **Networking:** 10/40/100GbE via PCIe

---

## Parallel Processing

### Concept
Executing multiple operations simultaneously to improve performance.

### Principles

**Performance Laws:**
- **Amdahl's Law:** Speedup limited by serial portion
  ```
  Speedup(n) = 1 / ((1-P) + P/n)
  ```
  Where P = parallelizable fraction, n = processors

- **Gustafson's Law:** Scaled speedup with larger problem size
  ```
  ScaledSpeedup(n) = n - α(n-1)
  ```
  Where α = serial fraction

**Parallelism Levels:**
1. **Instruction Level Parallelism (ILP):** Superscalar, out-of-order
2. **Data Level Parallelism (DLP):** SIMD, vector operations
3. **Thread Level Parallelism (TLP):** Multicore, multithreading
4. **Task Level Parallelism:** Distributed systems

**Multiprocessor Architectures:**
- **SMP (Symmetric Multiprocessing):** All processors equal, shared memory
- **NUMA (Non-Uniform Memory Access):** Each processor has local memory
- **Cluster:** Multiple nodes connected by network

**GPU Architecture:**
- **SIMT (Single Instruction Multiple Threads):**
- **Warp:** 32 threads execute in lockstep
- **Streaming Multiprocessor (SM):** Multiple warps, shared resources
- **CUDA Blocks:** Group of threads that share memory
- **Thread Hierarchy:** Grid → Block → Thread

**Heterogeneous Computing:**
- CPU: Good for sequential, branchy code
- GPU: Good for parallel, data-parallel code
- FPGA: Good for custom, bit-level operations
- ASIC: Fixed-function, highest efficiency

### Algorithms & Code

**Amdahl's Law Calculator:**

```python
def amdahl_speedup(parallel_fraction, processors):
    """Calculate theoretical speedup using Amdahl's Law."""
    serial_fraction = 1 - parallel_fraction
    speedup = 1 / (serial_fraction + parallel_fraction / processors)
    return speedup

def amdahl_efficiency(parallel_fraction, processors):
    """Calculate parallel efficiency."""
    speedup = amdahl_speedup(parallel_fraction, processors)
    return speedup / processors

# Example: 95% parallelizable on 64 cores
speedup = amdahl_speedup(0.95, 64)  # ≈ 16.7x
efficiency = amdahl_efficiency(0.95, 64)  # ≈ 26%
```

**CUDA Matrix Multiplication:**

```cuda
__global__ void matrix_mul(float* A, float* B, float* C, int N) {
    int row = blockIdx.y * blockDim.y + threadIdx.y;
    int col = blockIdx.x * blockDim.x + threadIdx.x;
    
    if (row < N && col < N) {
        float sum = 0.0f;
        for (int k = 0; k < N; k++) {
            sum += A[row * N + k] * B[k * N + col];
        }
        C[row * N + col] = sum;
    }
}

// Shared memory optimized version
__global__ void matrix_mul_shared(float* A, float* B, float* C, int N) {
    __shared__ float tile_A[BLOCK_SIZE][BLOCK_SIZE];
    __shared__ float tile_B[BLOCK_SIZE][BLOCK_SIZE];
    
    int row = blockIdx.y * blockDim.y + threadIdx.y;
    int col = blockIdx.x * blockDim.x + threadIdx.x;
    float sum = 0.0f;
    
    for (int t = 0; t < (N + BLOCK_SIZE - 1) / BLOCK_SIZE; t++) {
        // Load tiles into shared memory
        tile_A[threadIdx.y][threadIdx.x] = A[row * N + t * BLOCK_SIZE + threadIdx.x];
        tile_B[threadIdx.y][threadIdx.x] = B[(t * BLOCK_SIZE + threadIdx.y) * N + col];
        __syncthreads();
        
        // Compute partial sum
        for (int k = 0; k < BLOCK_SIZE; k++) {
            sum += tile_A[threadIdx.y][k] * tile_B[k][threadIdx.x];
        }
        __syncthreads();
    }
    
    if (row < N && col < N) {
        C[row * N + col] = sum;
    }
}

// Launch kernel
dim3 blockDim(BLOCK_SIZE, BLOCK_SIZE);
dim3 gridDim((N + BLOCK_SIZE - 1) / BLOCK_SIZE, 
             (N + BLOCK_SIZE - 1) / BLOCK_SIZE);
matrix_mul_shared<<<gridDim, blockDim>>>(A, B, C, N);
```

**OpenMP Parallel Prefix Sum:**

```c
#include <omp.h>

void parallel_prefix_sum(int* arr, int n) {
    int* temp = malloc(n * sizeof(int));
    
    for (int step = 1; step < n; step *= 2) {
        #pragma omp parallel for
        for (int i = step; i < n; i++) {
            temp[i] = arr[i] + arr[i - step];
        }
        
        #pragma omp parallel for
        for (int i = step; i < n; i++) {
            arr[i] = temp[i];
        }
    }
    
    free(temp);
}

// Work-efficient parallel scan (Blelloch)
void parallel_scan(int* in, int* out, int n) {
    if (n <= 1) {
        if (n == 1) out[0] = in[0];
        return;
    }
    
    int* temp = malloc(n * sizeof(int));
    
    // Up-sweep (reduce)
    for (int d = 0; d < log2(n); d++) {
        #pragma omp parallel for
        for (int k = 0; k < n; k += (1 << (d+1))) {
            int i = k + (1 << (d+1)) - 1;
            int j = k + (1 << d) - 1;
            temp[i] = in[i] + in[j];
        }
        #pragma omp parallel for
        for (int i = 0; i < n; i++) {
            in[i] = temp[i];
        }
    }
    
    // Down-sweep
    in[n-1] = 0;
    for (int d = log2(n) - 1; d >= 0; d--) {
        #pragma omp parallel for
        for (int k = 0; k < n; k += (1 << (d+1))) {
            int i = k + (1 << d) - 1;
            int j = k + (1 << (d+1)) - 1;
            temp[i] = in[j];
            temp[j] = in[i] + in[j];
        }
        #pragma omp parallel for
        for (int i = 0; i < n; i++) {
            in[i] = temp[i];
        }
    }
    
    #pragma omp parallel for
    for (int i = 0; i < n; i++) {
        out[i] = in[i];
    }
    
    free(temp);
}
```

### Trade-offs

**Parallel Programming Models:**
- **Shared Memory (OpenMP):** Easier but limited scalability
- **Message Passing (MPI):** Better scalability, more complex
- **Hybrid (OpenMP + MPI):** Best of both worlds

**GPU vs CPU:**
- GPU: Higher throughput for data-parallel workloads
- CPU: Better for branchy, sequential code

**Synchronization:**
- Coarse-grained: Simple, less contention
- Fine-grained: Better parallelism, higher overhead

### Applications

- **Machine Learning:** Matrix operations, neural network training
- **Scientific Computing:** PDE solvers, molecular dynamics
- **Computer Vision:** Image processing, video encoding
- **Financial Modeling:** Monte Carlo simulations

---

## Power Management

### Concept
Techniques to reduce power consumption while maintaining performance.

### Principles

**Power Components:**
- **Dynamic Power (P_dyn):** Switching activity
  ```
  P_dyn = α × C × V^2 × f
  ```
  Where α = activity factor, C = capacitance, V = voltage, f = frequency

- **Static Power (P_static):** Leakage current
  ```
  P_static = I_leakage × V
  ```

**Power-Saving Techniques:**
- **DVFS (Dynamic Voltage and Frequency Scaling):** Reduce V and f together
- **Clock Gating:** Disable clock to idle units
- **Power Gating:** Turn off power to idle units
- **Sleep States:** C-states for cores, P-states for performance

**C-States (Sleep States):**

| State | Description | Latency |
|-------|-------------|---------|
| C0 | Active | 0 |
| C1 | Halt | ~1 μs |
| C1E | Enhanced Halt | ~10 μs |
| C3 | Sleep | ~50 μs |
| C6 | Deep Power Down | ~100 μs |
| C7 | Deep Sleep | ~200 μs |

**P-States (Performance States):**
- Different frequency/voltage operating points
- Trade-off between performance and power

**Power Wall:**
- Power density limits frequency scaling
- Thermal design power (TDP) constraints
- Dark silicon: Only portion of chip can be active

### Algorithms & Code

**DVFS Controller:**

```c
typedef struct {
    float utilization;  // 0.0 to 1.0
    float target;      // Target utilization (e.g., 0.8)
    float kp, ki, kd;  // PID coefficients
    float integral;
    float last_error;
} DVFSController;

typedef struct {
    float voltage;     // Volts
    float frequency;   // MHz
} OperatingPoint;

OperatingPoint dvfs_points[] = {
    {0.7, 800},   // P0
    {0.8, 1000},  // P1
    {0.9, 1200},  // P2
    {1.0, 1400},  // P3
    {1.1, 1600},  // P4
};

int dvfs_control(DVFSController* ctrl, float current_util) {
    // PID controller
    float error = ctrl->target - current_util;
    ctrl->integral += error;
    float derivative = error - ctrl->last_error;
    ctrl->last_error = error;
    
    float output = ctrl->kp * error + 
                   ctrl->ki * ctrl->integral + 
                   ctrl->kd * derivative;
    
    // Map to operating point
    int p_state = (int)clamp(output + 2, 0, 4);
    set_p_state(p_state);
    
    return p_state;
}
```

**Clock Gating:**

```verilog
// Always-on clock
input clk;
input enable;
input rst;

// Clock gated clock
reg gated_clk;
always @(*) begin
    gated_clk = clk && enable;
end

// Flip-flop with clock gating
always @(posedge gated_clk or posedge rst) begin
    if (rst)
        q <= 0;
    else
        q <= d;
end

// More efficient hardware clock gating
// (uses latch to prevent glitches)
input clk;
input enable;
wire clk_en_latch;
wire gated_clk;

assign clk_en_latch = enable;
reg en_latch;
always @(clk) begin
    if (!clk)
        en_latch <= clk_en_latch;
end
assign gated_clk = clk && en_latch;
```

**Power Gating:**

```c
void power_gate_unit(int unit_id) {
    // Save state if needed
    save_unit_state(unit_id);
    
    // Assert power gate signal
    POWER_GATE[unit_id] = 1;
    
    // Wait for power down
    while (!POWER_GOOD[unit_id]);
}

void power_ungate_unit(int unit_id) {
    // Assert power enable
    POWER_GATE[unit_id] = 0;
    
    // Wait for power up
    while (!POWER_GOOD[unit_id]);
    
    // Restore state
    restore_unit_state(unit_id);
}
```

### Trade-offs

**DVFS:**
- Lower frequency/voltage: Less power, less performance
- Higher frequency/voltage: More power, more performance

**Clock Gating:**
- Fine-grained: Better power savings, more complex
- Coarse-grained: Simpler, less savings

**Power Gating:**
- Aggressive: Maximum power savings, longer wake latency
- Conservative: Less savings, faster wake

### Applications

- **Mobile Devices:** Extend battery life
- **Data Centers:** Reduce energy costs
- **Embedded Systems:** Thermal management
- **High-Performance Computing:** Manage thermal constraints

---

## Summary

This skill covers advanced computer architecture topics essential for understanding modern processor design and performance optimization. Key takeaways:

1. **Data Representation:** Two's complement dominates for arithmetic, IEEE 754 for floating-point
2. **CPU Architecture:** Out-of-order execution and branch prediction are key to performance
3. **Memory Hierarchy:** Multi-level caches exploit locality, MESI maintains coherence
4. **Memory System:** DRAM organization affects bandwidth and latency
5. **Bus & Interconnect:** PCIe and AMBA are dominant standards
6. **Parallel Processing:** GPU and multicore enable massive parallelism
7. **Power Management:** DVFS and gating are essential for energy efficiency

Understanding these principles is crucial for system optimization, compiler design, and creating high-performance applications.
