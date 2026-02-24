#  GPU Automation


---

## Table of Contents

- [Part I: System Design Questions](#part-i-system-design-questions)
  - [GPU Health Check System Design](#gpu-health-check-system-design-very-high-probability)
  - [GPU Triage Automation Service](#gpu-triage-automation-service)
  - [GPU Fleet Load Balancer](#gpu-fleet-load-balancer)
  - [Single Machine → Cluster Pattern](#single-machine--cluster-pattern)
  - [Fault Tolerance for GPU Fleet State](#fault-tolerance-for-gpu-fleet-state-high-probability)
  - [Capacity Tracking & Allocation System](#capacity-tracking--allocation-system)
  - [Monitoring Agent Rollout to 65K Nodes](#monitoring-agent-rollout-to-65k-nodes)
- [Part II: Python Coding Questions](#part-ii-python-coding-questions)
  - [Q1: Parse GPU Health Logs & Detect Failures](#q1-parse-gpu-health-logs--detect-failures)
  - [Q2: GPU Cluster Node Availability — Merge Intervals](#q2-gpu-cluster-node-availability--merge-intervals)
  - [Q3: Rate Limiter for GPU Triage API](#q3-rate-limiter-for-gpu-triage-api)
  - [Q4: Detect Cascading GPU Failures (Graph BFS)](#q4-detect-cascading-gpu-failures-graph-bfs)
  - [Q5: LRU Cache for GPU State Lookups](#q5-lru-cache-for-gpu-state-lookups)
  - [Q6: Max Non-Adjacent GPU Repair Jobs (DP)](#q6-max-non-adjacent-gpu-repair-jobs-dp)
  - [Q7: Connection Pool Manager (OOP)](#q7-connection-pool-manager-oop)
  - [Q8: Top K Failing GPUs from Event Stream](#q8-top-k-failing-gpus-from-event-stream)
- [Part III: Shell Scripting Questions](#part-iii-shell-scripting-questions)
  - [Q9: GPU Fleet Health Dashboard Script](#q9-gpu-fleet-health-dashboard-script)
  - [Q10: Automated Node Drain & Repair Workflow](#q10-automated-node-drain--repair-workflow)
  - [Q11: NVIDIA GPU Diagnostics Collector](#q11-nvidia-gpu-diagnostics-collector)
  - [Q12: Log Rotation & Alerting Pipeline](#q12-log-rotation--alerting-pipeline)
  - [Q13: Kubernetes GPU Pod Health Checker](#q13-kubernetes-gpu-pod-health-checker)
  - [Q14: Parallel NCCL Bandwidth Test Runner](#q14-parallel-nccl-bandwidth-test-runner)

---

# Part I: System Design Questions

## GPU Health Check System Design [VERY HIGH Probability]

**Q: How would you design a health check system for GPU nodes? Think of it like load balancer health checks but for GPUs.**

### Answer: Multi-Layer Health Checks (LB Analogy)

| Layer | Check | Description |
|-------|-------|-------------|
| **Layer 0** | Host alive | ICMP/SSH reachability |
| **Layer 1** | GPU hardware | DCGM health check: temperature, ECC, Xid errors |
| **Layer 2** | NIC/RDMA connectivity | `ib_send_bw` latency test to a known peer |
| **Layer 3** | Functional | Run a small NCCL all-reduce, verify expected bandwidth |

**Key Design Principles:**

- **Configurable thresholds and intervals** — just like LB health check intervals, timeout periods, and unhealthy thresholds. A GPU must fail N consecutive checks before being marked unhealthy.
- **Graceful draining** — exactly like LB connection draining. When a GPU is going unhealthy, drain active training jobs gracefully before removing from the pool.
- **Backend set management** — treat the GPU fleet like an LB backend set. Healthy GPUs are in the active pool; unhealthy ones are in repair; recovered ones go through validation before re-entering.
- **Aggregate health** — a "cluster health score" based on the percentage of healthy GPUs, similar to how an LB evaluates overall backend health.

---

## GPU Triage Automation Service

> The primary system design you may be asked to whiteboard. Directly relevant to this team's work.

### Functional Requirements

- Ingest health telemetry from 65,000+ GPU nodes in near real-time
- Detect GPU failures, degradation, and anomalies automatically
- Execute automated triage workflows (diagnose, drain, repair, validate)
- Minimize customer impact with proactive replacement
- Provide APIs for status queries and manual overrides

### Non-Functional Requirements

| Requirement | Target | Rationale |
|-------------|--------|-----------|
| Detection Latency | < 60 seconds for critical failures | Customer SLA and 1-Day repair SLO |
| Throughput | 1M+ metric data points per minute | 65K nodes × 8 GPUs × metrics at 10s intervals |
| Availability | 99.99% for triage system itself | Must be more reliable than fleet it monitors |
| Consistency | Strong for repairs, eventual for dashboards | Cannot double-repair; dashboards tolerate staleness |

### Five-Layer Architecture

**Layer 1: Data Collection**
DCGM agent on each bare metal node collects GPU metrics every 10s. Custom host agent collects NIC, NVLink, PCIe, thermal, and power data. Agents publish to regional Kafka topics partitioned by cluster-id.

**Layer 2: Stream Processing**
Real-time anomaly detection via sliding window aggregations on Xid error rates, ECC error trends, and temperature spikes. ML model compares current provisioning failure rates vs historical baseline. Correlation engine links GPU errors with NIC/network events to distinguish GPU failure from network partition.

**Layer 3: Decision Engine**
Rule-based triage maps Xid codes to decision tree (reset, drain, replace). ML-enhanced prediction estimates failure probability. State machine per GPU: `healthy → degraded → draining → under_repair → validating → healthy`. Strong consistency via MySQL + leader election.

**Layer 4: Action Orchestration**
Drain orchestrator gracefully migrates workloads. Diagnostic runner executes GPU burn-in, NCCL tests, memory stress tests. DC Ops integration creates repair tickets via API. Validation pipeline runs post-repair checks before returning to pool.

**Layer 5: API & Observability**
REST APIs for cluster health queries, manual triage triggers, repair status. Dashboards with fleet-wide health heatmap, repair SLO tracking, Xid trends. PagerDuty/OpsGenie for human escalation.

### Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| **Why Kafka?** | Handles 1M+ events/min, replay capability, decouples producers/consumers, multiple consumer groups |
| **Why MySQL for state?** | Strong consistency for repair decisions — avoid two systems repairing same GPU. JD explicitly mentions MySQL |
| **Why state machine?** | Prevents invalid transitions. Makes system auditable and debuggable |
| **Blast radius management** | If 100 GPUs report unhealthy simultaneously, detect as monitoring/network issue. Rate-limit repair actions |

---

## GPU Fleet Load Balancer

> Maps LB concepts directly to GPU fleet management — speaks Jonathan's language.

### LB → GPU Fleet Concept Mapping

| LB Concept | GPU Fleet Equivalent | Implementation |
|------------|---------------------|----------------|
| Frontend Listener | GPU Request API | REST API for cluster provisioning |
| Backend Set | GPU Pool (per model) | H100, H200, B200 pools per region/AD |
| Backend Server | Individual GPU Node | Bare metal instance with 8 GPUs |
| Health Check | GPU Health Monitor | DCGM + NCCL + NIC checks (multi-layer) |
| Load Balancing Policy | Placement Algorithm | Locality-aware: same data-hall preferred |
| Connection Draining | Workload Draining | Graceful job checkpoint + migration |
| Session Persistence | Cluster Affinity | Training job stays on allocated GPU set |
| Bandwidth Shape | Cluster Size | Customer selects GPU count; allocate contiguous block |

### Request Flow

1. Customer requests 256-GPU H200 cluster via API
2. Request Router checks available capacity in requested region
3. Placement Engine selects optimal GPU set: healthy, within same network locality, minimal fragmentation
4. Reservation Manager atomically reserves GPU set (distributed lock to prevent double-allocation)
5. Provisioning Pipeline configures network isolation, RDMA fabric, bootstraps cluster
6. Health Monitor continuously checks cluster; if GPU degrades, trigger triage
7. On decommission: drain workloads, release GPUs back to pool

### Key Design Decisions

- **Strong consistency** for allocation state (MySQL + sync replication)
- **Eventual consistency** for health metrics (dashboards can be slightly stale)
- **Placement locality** to minimize cross-hall traffic in multi-tier Clos topology
- **Blast radius control** — don't remove all backends if health checks all fail at once

---

## Single Machine → Cluster Pattern

> OCI wants engineers who naturally think: "This works on one machine, but what happens at 65,000 nodes?"

### Phase 1 — Single Machine

```python
from collections import Counter

class GPULogAggregator:
    def __init__(self):
        self.error_counts = Counter()
        self.alerts = []

    def process_event(self, gpu_id, event_type, value):
        self.error_counts[gpu_id] += 1
        if event_type in ("xid_48", "ecc_uncorrectable"):
            self.alerts.append({'gpu': gpu_id, 'event': event_type})

    def top_failing(self, k=5):
        return self.error_counts.most_common(k)

# Time: O(1) per event, O(n log k) for top-k | Space: O(n)
```

### Phase 2 — Scale to 65,000 Nodes (5 Dimensions)

| Dimension | Single Machine | 65K Node Cluster | Key Trade-off |
|-----------|---------------|-------------------|---------------|
| **Ingestion** | Direct function call | Kafka streaming | Latency 10ms → 100ms but unlimited throughput |
| **Top-K query** | O(n log k) in memory | Two-level merge | O(c × k log k) where c=clusters |
| **State** | Python dict | MySQL + Redis cache | Read: eventual via cache. Write: strong via MySQL |
| **Failure impact** | Total outage | Partial degradation | Isolation at cluster and region level |
| **Coordination** | None needed | Leader election + locks | Only for triage decisions |

**Dimension 1 — Data Collection:** Switch from pull to push. Each node pushes events to Kafka topics partitioned by `cluster_id`.

**Dimension 2 — Aggregation:** Two-level merge: per-cluster aggregators (~500 nodes each), then global merge. O(c × k) not O(N).

**Dimension 3 — State Management:** MySQL for triage decisions (strong consistency), Redis for dashboards (eventual). 65K nodes × 8 GPUs = 520K rows — MySQL handles without sharding.

**Dimension 4 — Fault Tolerance:** Kafka 7-day retention for replay. Circuit breaker: if >10% fleet reports unhealthy, pause triage. Regional isolation.

**Dimension 5 — Complexity Changes:** Ingestion: direct call → Kafka streaming. Top-K: O(n log k) → O(c × k log k). State: dict → MySQL + Redis. Failure: total outage → partial degradation.

### How to Handle This Live

1. *"First, let me identify what changes at scale"* — pause and think out loud
2. *"Data collection switches from pull to push — we need an event bus like Kafka"*
3. *"State management needs consistency guarantees — MySQL for triage, Redis for dashboards"*
4. *"Aggregation becomes two-level: per-cluster local, then global merge"*
5. *"Fault tolerance: Kafka replay for recovery, circuit breakers for blast radius, regional isolation"*
6. *"Time complexity changes from O(n) to O(c×k) where c=clusters, k=summary size"*

---

## Fault Tolerance for GPU Fleet State [HIGH Probability]

**Q: How do you handle fault tolerance in a system managing GPU fleet state?**

1. **State replication** — strongly consistent datastore (MySQL with synchronous replication or etcd)
2. **Idempotent operations** — all repair and triage actions must be idempotent for safe retries
3. **Leader election** for triage decision-making to avoid conflicting repair actions
4. **Event sourcing** for audit trail — every state transition logged with Kafka for replay
5. **Graceful degradation** — if monitoring fails, GPUs continue operating; we lose visibility, not functionality
6. **Cascading prevention** — if 100 nodes appear unhealthy simultaneously, detect as monitoring failure, not fleet failure

---

## Capacity Tracking & Allocation System

**Q: Design a capacity tracking and allocation system for GPUs across multiple OCI regions.**

1. **Inventory layer** — real-time tracking of every GPU: region, AD, rack, node, model, health status, allocation state. MySQL with strong consistency
2. **Allocation engine** — check available capacity, apply placement hints for locality, reserve atomically with distributed locks
3. **Capacity forecasting** — time-series analysis of demand by region, hardware procurement lead times, utilization trends
4. **Fragmentation management** — track fragmentation ratio and trigger defragmentation (workload migration) when threshold exceeded
5. **APIs and dashboards** — capacity heatmaps per region/AD, allocation rates, repair pipeline throughput

---

## Monitoring Agent Rollout to 65K Nodes

**Q: How would you automate deployment of monitoring agents to 65,000 GPU nodes?**

1. **Package as immutable artifact** — containerized, versioned, signed
2. **Canary deployment** — 1% of nodes in one AD, monitor 2 hours: agent overhead, collection success, no workload impact
3. **Progressive waves:** 5% → 25% → 50% → 100%, each with automated health gates (error rate < 0.1%)
4. **Rollback automation** — any wave failing health gates triggers automatic rollback
5. **Regional sequencing** — one region at a time, starting with least-utilized
6. **Config-driven** — agents pull config from central service, thresholds update without redeployment

---

# Part II: Python Coding Questions

### Question Index

| # | Question | Difficulty | Pattern |
|---|----------|------------|---------|
| Q1 | Parse GPU Health Logs & Detect Failures | Medium | Log Parsing / Dict |
| Q2 | GPU Cluster Node Availability Tracker | Medium | Interval Merge |
| Q3 | Rate Limiter for Triage API | Medium | Sliding Window |
| Q4 | Detect Cascading GPU Failures | Hard | Graph BFS/DFS |
| Q5 | LRU Cache for GPU State Lookups | Medium | HashMap + DLL |
| Q6 | Max Non-Adjacent GPU Repair Jobs | Medium | Dynamic Programming |
| Q7 | Connection Pool Manager | Medium | OOP / Threading |
| Q8 | Top K Failing GPUs from Stream | Medium | Heap / Counter |

### Most Likely Patterns for This Role

| Pattern | Likelihood | Why |
|---------|-----------|-----|
| Dictionary/HashMap | Very High | Log parsing, metrics aggregation, state tracking |
| Interval Problems | High | Maintenance windows, availability calculations |
| Graph BFS/DFS | High | Failure cascading, network topology traversal |
| Sliding Window | High | Rate limiting, anomaly detection, moving averages |
| Heap / Priority Queue | Medium | Top-K problems, priority-based scheduling |
| Dynamic Programming | Medium | Resource optimization, scheduling (confirmed asked) |
| OOP / Design Patterns | High | Connection pooling, state machines, cache design |

---

## Q1: Parse GPU Health Logs & Detect Failures

> **Difficulty:** Medium | **Pattern:** Log Parsing / Dictionary

**Problem:** You receive GPU health log entries as strings with format: `"<timestamp> <node_id> <gpu_id> <metric> <value>"`. Group events by node/GPU, return GPUs needing triage (temperature > 80 OR xid_error == 48 OR ecc_uncorrectable > 0) with alert reasons and severity.

```python
from collections import defaultdict

def detect_failing_gpus(logs):
    gpu = defaultdict(lambda: defaultdict(list))
    for log in logs:
        ts, node, gid, metric, val = log.split()
        gpu[(node, gid)][metric].append(float(val))

    RULES = {
        'temperature': lambda v: max(v) > 80,
        'xid_error':   lambda v: 48 in v,
        'ecc_uncorrectable': lambda v: max(v) > 0
    }

    alerts = []
    for (node, gid), metrics in gpu.items():
        triggered = [m for m, fn in RULES.items()
                     if m in metrics and fn(metrics[m])]
        if triggered:
            sev = 'critical' if 'xid_error' in triggered else 'warning'
            alerts.append({'node': node, 'gpu': gid,
                           'alerts': triggered, 'severity': sev})
    return sorted(alerts, key=lambda x: (x['severity'] != 'critical', x['node']))

# Time: O(n) parse + O(m log m) sort | Space: O(m), m=unique GPUs
```

> **Discussion:** Extensible rules pattern — easy to add new triage rules without changing core logic. In production, these rules would stream via Kafka consumer. The severity classification enables prioritized triage queues.

---

## Q2: GPU Cluster Node Availability — Merge Intervals

> **Difficulty:** Medium | **Pattern:** Interval Merge

**Problem:** Given maintenance windows as intervals `[start, end]` in hours, find total unavailable time (merge overlapping windows) and identify longest continuous downtime.

```python
def analyze_downtime(intervals):
    if not intervals:
        return {'total': 0, 'longest': 0, 'merged': []}
    intervals.sort()
    merged = [intervals[0][:]]
    for s, e in intervals[1:]:
        if s <= merged[-1][1]:
            merged[-1][1] = max(merged[-1][1], e)
        else:
            merged.append([s, e])
    durations = [e - s for s, e in merged]
    return {'total': sum(durations), 'longest': max(durations), 'merged': merged}

# Follow-up: find available (gap) windows
def find_gaps(merged, total_range):
    points = [total_range[0]] + [x for iv in merged for x in iv] + [total_range[1]]
    return [[points[i], points[i+1]] for i in range(0, len(points), 2)
            if points[i] < points[i+1]]

# Time: O(n log n) sort-dominated | Space: O(n)
```

---

## Q3: Rate Limiter for GPU Triage API

> **Difficulty:** Medium | **Pattern:** Sliding Window

**Problem:** Design a rate limiter allowing at most N requests per window (in seconds) per client.

### Single Process Solution

```python
from collections import defaultdict, deque
import time, threading

class SlidingWindowRateLimiter:
    def __init__(self, max_req, window_sec):
        self.max_req, self.window = max_req, window_sec
        self.clients = defaultdict(deque)
        self.lock = threading.Lock()

    def allow(self, client_id):
        with self.lock:
            now = time.time()
            q = self.clients[client_id]
            while q and q[0] <= now - self.window:
                q.popleft()
            if len(q) < self.max_req:
                q.append(now)
                return True
            return False

# Time: O(1) amortized | Space: O(k) per client
```

### Distributed Version (20 API Servers — Redis + Lua)

```python
import redis, time

class DistributedRateLimiter:
    """Sliding window via Redis sorted set + Lua for atomicity."""
    LUA = """
    local key, now = KEYS[1], tonumber(ARGV[1])
    local window, max_req = tonumber(ARGV[2]), tonumber(ARGV[3])
    redis.call('ZREMRANGEBYSCORE', key, 0, now - window)
    if redis.call('ZCARD', key) < max_req then
        redis.call('ZADD', key, now, now .. ':' .. math.random())
        redis.call('EXPIRE', key, window)
        return 1
    end
    return 0
    """

    def __init__(self, r, max_req, window_sec):
        self.r, self.max_req, self.window = r, max_req, window_sec

    def allow(self, client_id):
        return self.r.eval(self.LUA, 1, f"rl:{client_id}",
                           time.time(), self.window, self.max_req) == 1

# Time: O(log n) per request | Space: O(max_req) per client
# +1ms RTT, but guarantees global consistency across 20 servers
```

| Aspect | Single Process | 20-Server Distributed |
|--------|---------------|----------------------|
| Time per request | O(1) amortized (deque cleanup) | O(log n) Redis sorted set + 1ms RTT |
| Consistency | Perfect (single process) | Strongly consistent (Redis + Lua atomicity) |
| Failure mode | Process crash = limit reset | Redis crash = fail-open or fail-closed |
| Scale limit | One process | Redis handles 100K+ ops/sec |

---

## Q4: Detect Cascading GPU Failures (Graph BFS)

> **Difficulty:** Hard | **Pattern:** Graph BFS/DFS

**Problem:** GPUs connected via NVLink/RDMA. Given adjacency list and initially failed GPUs, determine which GPUs fail in cascade (a GPU fails if >50% of its neighbors have failed). Return total failed set.

```python
from collections import deque, defaultdict

def cascade_failures(graph, initial_failed):
    failed = set(initial_failed)
    queue = deque(initial_failed)
    failed_nb = defaultdict(int)

    for gpu in initial_failed:
        for nb in graph.get(gpu, []):
            failed_nb[nb] += 1

    while queue:
        for nb in graph.get(queue.popleft(), []):
            if nb not in failed and failed_nb[nb] > len(graph[nb]) / 2:
                failed.add(nb)
                queue.append(nb)
                for n2 in graph.get(nb, []):
                    failed_nb[n2] += 1
    return failed

# Time: O(V + E) each GPU processed once | Space: O(V)
```

> **Real-World Application:** This is blast radius estimation — critical for deciding whether to drain an entire rack. The 50% threshold is configurable; in production, use different thresholds for NVLink vs RDMA connections.

---

## Q5: LRU Cache for GPU State Lookups

> **Difficulty:** Medium | **Pattern:** HashMap + Doubly Linked List

```python
from collections import OrderedDict

class GPUStateCache:
    def __init__(self, capacity):
        self.cap = capacity
        self.cache = OrderedDict()

    def get(self, gpu_id):
        if gpu_id in self.cache:
            self.cache.move_to_end(gpu_id)
            return self.cache[gpu_id]
        return None

    def put(self, gpu_id, state):
        if gpu_id in self.cache:
            self.cache.move_to_end(gpu_id)
        self.cache[gpu_id] = state
        if len(self.cache) > self.cap:
            self.cache.popitem(last=False)

# Time: O(1) get/put | Space: O(capacity)
# OrderedDict = hash table + doubly-linked list internally
```

---

## Q6: Max Non-Adjacent GPU Repair Jobs (DP)

> **Difficulty:** Medium | **Pattern:** Dynamic Programming
>
> ⚠️ **Confirmed asked in Oracle OCI interviews.** Cannot schedule two adjacent jobs simultaneously. Find maximum total priority.

```python
def max_repair_priority(priorities):
    prev2, prev1 = 0, 0
    for p in priorities:
        prev2, prev1 = prev1, max(prev1, prev2 + p)
    return prev1

def max_repair_with_indices(priorities):
    n = len(priorities)
    dp = [0] * (n + 2)
    for i in range(n - 1, -1, -1):
        dp[i] = max(dp[i+1], priorities[i] + dp[i+2])
    selected, i = [], 0
    while i < n:
        if priorities[i] + dp[i+2] >= dp[i+1]:
            selected.append(i); i += 2
        else:
            i += 1
    return dp[0], selected

# Time: O(n) | Space: O(1) value-only, O(n) with reconstruction
```

---

## Q7: Connection Pool Manager (OOP)

> **Difficulty:** Medium | **Pattern:** OOP / Threading
>
> ⚠️ **Connection pooling confirmed asked at Oracle OCI.** Thread-safe pool for GPU fleet state database.

```python
import threading, time
from queue import Queue, Empty

class ConnectionPool:
    def __init__(self, max_conns, timeout=5):
        self.pool = Queue(max_conns)
        self.max, self.active = max_conns, 0
        self.lock = threading.Lock()
        self.timeout = timeout

    def acquire(self):
        with self.lock:
            if self.active < self.max:
                self.active += 1
                return {'id': self.active, 'created': time.time()}
        try:
            return self.pool.get(timeout=self.timeout)
        except Empty:
            raise TimeoutError("No connections available")

    def release(self, conn):
        self.pool.put(conn)

# Time: O(1) acquire/release | Space: O(max_connections)
# Real bottleneck: lock contention — consider per-shard pools at scale
```

---

## Q8: Top K Failing GPUs from Event Stream

> **Difficulty:** Medium | **Pattern:** Heap / Counter

```python
import heapq
from collections import Counter

class TopKFailingGPUs:
    def __init__(self):
        self.counts = Counter()

    def process_event(self, gpu_id):
        self.counts[gpu_id] += 1

    def top_k(self, k):
        # Min-heap of size k: O(n log k) vs O(n log n) full sort
        heap = []
        for gpu, count in self.counts.items():
            if len(heap) < k:
                heapq.heappush(heap, (count, gpu))
            elif count > heap[0][0]:
                heapq.heapreplace(heap, (count, gpu))
        return sorted(heap, reverse=True)

# Time: O(n log k) | Space: O(n + k)
```

---

# Part III: Shell Scripting Questions

| # | Question | Difficulty | Pattern |
|---|----------|------------|---------|
| Q9 | GPU Fleet Health Dashboard Script | Medium | awk/sed/grep |
| Q10 | Automated Node Drain & Repair | Medium | Process Mgmt |
| Q11 | NVIDIA GPU Diagnostics Collector | Medium | nvidia-smi/parsing |
| Q12 | Log Rotation & Alerting Pipeline | Medium | cron/logrotate |
| Q13 | Kubernetes GPU Pod Health Checker | Medium | kubectl/jq |
| Q14 | Parallel NCCL Bandwidth Test Runner | Hard | SSH/parallel exec |

---

## Q9: GPU Fleet Health Dashboard Script

> **Difficulty:** Medium | **Pattern:** awk/sed/grep + nvidia-smi

```bash
#!/bin/bash
# gpu_fleet_health.sh -- Parse nvidia-smi, flag unhealthy GPUs
set -euo pipefail
TEMP_LIMIT=85; LOG="/var/log/gpu_health/$(date +%Y%m%d_%H%M).csv"

nvidia-smi --query-gpu=index,temperature.gpu,utilization.gpu,\
memory.used,memory.total,ecc.errors.uncorrectable.volatile \
--format=csv,noheader,nounits | while IFS=, read -r idx temp util mu mt ecc; do
  mem_pct=$((mu * 100 / mt))
  status="OK"
  [ "$temp" -gt "$TEMP_LIMIT" ] && status="TEMP_HIGH"
  [ "$ecc" -gt 0 ] && status="ECC_ERROR"
  echo "$idx,$temp,$util%,$mem_pct%,$ecc,$status"
  [ "$status" != "OK" ] && echo "ALERT: GPU-$idx $status" >&2
done | tee "$LOG"

BAD=$(awk -F, '$NF!="OK"' "$LOG" | wc -l)
echo "Checked $(wc -l < "$LOG") GPUs, $BAD unhealthy"
[ "$BAD" -gt 0 ] && exit 1 || exit 0
```

---

## Q10: Automated Node Drain & Repair Workflow

> **Difficulty:** Medium | **Pattern:** Process Management

```bash
#!/bin/bash
# node_drain_repair.sh -- Drain, diagnose, ticket
set -euo pipefail
NODE="$1"; API="https://fleet.internal/v1"

echo "[$(date +%H:%M:%S)] Draining $NODE..."
curl -s -X POST "$API/nodes/$NODE/drain" | jq -r '.status'

for i in $(seq 1 30); do
  COUNT=$(curl -s "$API/nodes/$NODE/workloads" | jq '.count')
  [ "$COUNT" -eq 0 ] && break; sleep 10
done
[ "$COUNT" -gt 0 ] && { echo "ERROR: workloads still running"; exit 1; }

echo "Running diagnostics..."
DIAG=$(ssh "$NODE" "nvidia-smi -q | grep -c 'Err'" 2>/dev/null || echo "999")

TICKET=$(curl -s -X POST "$API/tickets" \
  -d "{\"node\":\"$NODE\",\"errors\":$DIAG,\"priority\":\"P1\"}" | jq -r '.id')
echo "Ticket: $TICKET | Node: $NODE | Status: AWAITING_REPAIR"
```

---

## Q11: NVIDIA GPU Diagnostics Collector

> **Difficulty:** Medium | **Pattern:** nvidia-smi / JSON parsing

```bash
#!/bin/bash
# gpu_diagnostics.sh -- Collect GPU health as JSON
set -euo pipefail
OUT="/var/log/gpu_diag/$(hostname)_$(date +%Y%m%d_%H%M).json"

echo "{\"host\":\"$(hostname)\",\"ts\":\"$(date -u +%FT%TZ)\",\"gpus\":["
nvidia-smi --query-gpu=index,temperature.gpu,power.draw,utilization.gpu,\
memory.used,memory.total,ecc.errors.uncorrectable.volatile,\
pcie.link.gen.current --format=csv,noheader,nounits | \
while IFS=, read -r idx t pwr util mu mt ecc gen; do
  st=$([ "$ecc" -gt 0 ] && echo "FAIL" || echo "PASS")
  echo "{\"gpu\":$idx,\"temp\":$t,\"power\":\"${pwr}W\",\
  \"ecc\":$ecc,\"pcie_gen\":$gen,\"status\":\"$st\"},"
done
echo "]}" | tee "$OUT"
grep -q '"FAIL"' "$OUT" && exit 1 || exit 0
```

---

## Q12: Log Rotation & Alerting Pipeline

> **Difficulty:** Medium | **Pattern:** cron / logrotate

```bash
#!/bin/bash
# gpu_log_monitor.sh -- Rotate logs, alert on critical GPU errors
set -euo pipefail
LOG_DIR="/var/log/gpu"; MAX_MB=100; RETAIN=7
ALERT="https://alerts.internal/v1/notify"

# Rotate oversized logs
find "$LOG_DIR" -name "*.log" -size +${MAX_MB}M -exec sh -c \
  'mv "$1" "$1.$(date +%Y%m%d).bak" && touch "$1"' _ {} \;
find "$LOG_DIR" -name "*.bak" -mtime +$RETAIN -delete

# Scan for critical errors
for f in "$LOG_DIR"/*.log; do
  ERRS=$(grep -c -E "Xid.*48|ECC.*uncorrectable|fallen off" "$f" 2>/dev/null || echo 0)
  [ "$ERRS" -gt 0 ] && curl -s -X POST "$ALERT" \
    -d "{\"node\":\"$(hostname)\",\"file\":\"$f\",\"errors\":$ERRS}" \
    && echo "ALERT: $f ($ERRS errors)"
done
echo "Done. $(find "$LOG_DIR" -name "*.log" | wc -l) active logs."
```

---

## Q13: Kubernetes GPU Pod Health Checker

> **Difficulty:** Medium | **Pattern:** kubectl / jq

```bash
#!/bin/bash
# k8s_gpu_pod_health.sh -- Check GPU pod health
set -euo pipefail
NS="${1:-default}"

echo "=== GPU Pod Health ==="
kubectl get pods -n "$NS" -l "nvidia.com/gpu=true" \
  -o custom-columns='NAME:.metadata.name,NODE:.spec.nodeName,\
  STATUS:.status.phase,RESTARTS:.status.containerStatuses[0].restartCount' \
  --no-headers | while read -r name node status restarts; do
  [ "$status" != "Running" ] && echo "BAD: $name on $node (status=$status)"
  [ "${restarts:-0}" -gt 3 ] && echo "BAD: $name restart_loop ($restarts)"
done

echo "=== GPU Node Capacity ==="
kubectl get nodes -l "nvidia.com/gpu.present=true" \
  -o custom-columns='NODE:.metadata.name,\
  ALLOC:.status.allocatable.nvidia\.com/gpu,\
  CAP:.status.capacity.nvidia\.com/gpu' --no-headers
```

---

## Q14: Parallel NCCL Bandwidth Test Runner

> **Difficulty:** Hard | **Pattern:** SSH / Parallel Execution

```bash
#!/bin/bash
# nccl_bandwidth_test.sh -- Parallel NCCL across nodes
set -euo pipefail
HOSTFILE="$1"; PARALLEL=4; DIR="/tmp/nccl_results"; mkdir -p "$DIR"

run_test() {
  ssh -o ConnectTimeout=5 "$1" bash -c '
    NGPU=$(nvidia-smi -L | wc -l)
    if command -v all_reduce_perf &>/dev/null; then
      BW=$(all_reduce_perf -b 1M -e 1G -g $NGPU 2>&1 | tail -1 | awk "{print \$NF}")
      echo "{\"node\":\"$(hostname)\",\"gpus\":$NGPU,\"bw_gbps\":$BW,\"status\":\"pass\"}"
    else
      echo "{\"node\":\"$(hostname)\",\"gpus\":$NGPU,\"status\":\"no_nccl\"}"
    fi' > "$DIR/$1.json" 2>&1
}

active=0
while read -r node; do
  run_test "$node" &
  active=$((active + 1))
  [ "$active" -ge "$PARALLEL" ] && { wait -n; active=$((active - 1)); }
done < "$HOSTFILE"
wait

PASS=$(grep -l '"pass"' "$DIR"/*.json 2>/dev/null | wc -l)
TOTAL=$(ls "$DIR"/*.json 2>/dev/null | wc -l)
echo "$PASS/$TOTAL nodes passed"
[ "$PASS" -lt "$TOTAL" ] && { grep -L '"pass"' "$DIR"/*.json; exit 1; }
```

---

> **Coding Tips — Python:** Use `collections` stdlib (`defaultdict`, `Counter`, `OrderedDict`, `deque`). Think out loud. Handle edge cases. Always state Big-O after solving. Use classes for stateful problems.

> **Coding Tips — Shell:** Always start with `set -euo pipefail`. Use `awk` over complex grep/sed chains. Use `jq` for JSON. Know parallel execution with `&` and `wait`. Make scripts idempotent. Log everything with `tee`.
