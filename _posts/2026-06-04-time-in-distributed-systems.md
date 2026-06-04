---
title: "Time in Distributed Systems"
date: 2026-06-04 00:00:00 +0000
categories: [distributed-systems, theory]
tags: [distributed-systems, clocks, lamport, vector-clocks, ntp, spanner, causality]
---

# Time in Distributed Systems

Time is one of the most deceptively difficult problems in distributed systems. On a single machine, time is straightforward — you call `System.currentTimeMillis()` and get a number. Every part of your program agrees on that number. You can safely say "event A happened before event B" if A's timestamp is lower than B's.

The moment you have two machines, all of that breaks.

This post builds the concept from scratch: why physical clocks fail, what logical time actually means, and how systems like Lamport clocks, vector clocks, and Google Spanner each solve a different piece of the puzzle.

---

## The Physical Clock Problem

Every machine has a hardware clock — a quartz crystal that vibrates at roughly 32,768 times per second. This is what your OS reads when you call `time.time()`. The problem is that quartz drifts. Temperature, voltage, and age all cause the frequency to shift slightly. A typical server clock drifts by up to **200 ppm** — that is roughly **17 seconds per day**, or **1 millisecond every 10 seconds**.

To fix this, machines run **NTP (Network Time Protocol)**, which periodically corrects the clock by comparing it against time servers. With good NTP, two machines on the same LAN can stay within **1 ms** of each other. Over the internet, it is typically **10–50 ms**.

That sounds fine — but 1 ms is 1,000,000 nanoseconds. Any two events that happen within that window on different machines cannot be reliably ordered by timestamp.

There is a second, nastier problem. NTP does not just slow down or speed up your clock — when your clock is too far ahead, NTP will **jump it backward**. Your wall clock can go:

```
14:32:07.500
14:32:07.499   ← NTP stepped the clock back
14:32:07.501
```

Any code measuring elapsed time with `end - start` from wall-clock timestamps can produce a **negative duration**. This has caused real production outages.

> **Rule:** Never use wall-clock time to measure elapsed duration. Use a monotonic clock — one that only ever increases and is never adjusted by NTP.
>
> | | Wall Clock | Monotonic Clock |
> |---|---|---|
> | Linux | `CLOCK_REALTIME` | `CLOCK_MONOTONIC` |
> | Java | `System.currentTimeMillis()` | `System.nanoTime()` |
> | Python | `time.time()` | `time.monotonic()` |
> | Go | `time.Now()` | `time.Since(t)` |

Wall clocks are for displaying or logging a timestamp. Monotonic clocks are for measuring how long something took.

![NTP stratum hierarchy diagram](/assets/img/dist-time/fig06-ntp-stratum-hierarchy.png)
*Figure 1 — The NTP hierarchy. GPS-backed atomic clocks at Stratum 0 feed Stratum 1 servers, which feed the pool.ntp.org servers your machines actually sync against. Each hop adds uncertainty.*

---

## Why Timestamps Cannot Order Events

Here is a concrete failure. Two servers — Node A and Node B — both write to the same replicated key using **last-write-wins**: whichever write has the higher timestamp is kept.

- Node A writes `"alice"` at its clock time `T = 1000 ms`
- Node B writes `"bob"` at its clock time `T = 999 ms`

The system keeps `"alice"` because `1000 > 999`.

But what if Node B's quartz oscillator is running 1 ms fast? In real wall time, Node B wrote *after* Node A. `"bob"` should have won. The clock skew silently reversed the ordering, and we just lost a write with **no error, no warning, no indication anything went wrong**.

![Clock skew corrupting last-write-wins](/assets/img/dist-time/fig07-clock-skew-last-write-wins.png)
*Figure 2 — A 1 ms quartz drift reverses the apparent order of writes. The system confidently makes the wrong decision.*

And timestamps cannot even help you detect that this happened.

There is a third problem beyond drift and NTP jumps: **process pauses**. A JVM garbage collector can stop all threads for 30 seconds. A virtual machine can be live-migrated. An OS can schedule your process out. When the process resumes, its clock looks correct — but it has no idea that 30 seconds passed while it was frozen.

This matters for **lease-based locking**. A process acquires a lease valid for 10 seconds, pauses for 30 seconds due to GC, then resumes and writes to the shared resource — still believing it holds the lease. In those 30 seconds, the lease expired and another process took it. You now have two writers simultaneously, and physical time gave you no warning.

![Process pause lease violation](/assets/img/dist-time/fig08-process-pause-lease-violation.png)
*Figure 3 — A GC pause longer than the lease TTL causes a split-brain write. The frozen process has no way to know it was paused.*

These three problems — clock skew, NTP jumps, and process pauses — mean you **cannot use physical timestamps to determine the ordering of events across machines**. You need a different model.

---

## Logical Time — What We Actually Need

Here is Lamport's key insight from his 1978 paper: for most distributed system correctness problems, we do not actually need to know *when* something happened. We only need to know *whether* one event could have caused another.

This is the **happened-before** relation, written `→`.

> **Definition — Happened-Before (→)**
>
> Event `a` happened-before event `b` if:
> 1. `a` and `b` are on the same process, and `a` came first in execution order, **or**
> 2. `a` is the sending of a message, and `b` is the receipt of that same message, **or**
> 3. There is a chain: `a → x → ... → b` (transitivity)
>
> If neither `a → b` nor `b → a`, then `a` and `b` are **concurrent** — they have no causal relationship. There is no meaningful sense in which one "came first."

This is a **partial order**, not a total order. Not every pair of events is comparable. This is a fundamental property of distributed systems, not a limitation we can engineer away.

---

## Lamport Clocks

To make happened-before concrete and checkable, Lamport invented a simple mechanism: a logical integer counter that each process maintains and advances according to three rules.

> **Definition — Lamport Clock**
>
> Each process keeps an integer counter `C`, initialised to 0.
>
> - **Local event:** `C := C + 1`
> - **Send message:** `C := C + 1`, then attach `C` to the message
> - **Receive message:** `C := max(C, C_received) + 1`
>
> **Guarantee:** If `a → b`, then `C(a) < C(b)`.

The receive rule is the key. When you receive a message with timestamp 10 and your counter is only 3, you jump your counter to 11. This propagates causal knowledge — your counter now reflects that you have seen at least 10 events in the causal history that led to this message.

```python
class LamportClock:
    def __init__(self):
        self.c = 0

    def tick(self):
        """Call before any local event or send."""
        self.c += 1
        return self.c

    def send(self, message: dict) -> dict:
        self.c += 1
        message['ts'] = self.c
        return message

    def receive(self, message: dict) -> int:
        self.c = max(self.c, message['ts']) + 1
        return self.c
```

Let us trace through a concrete example with three processes:

| Event | Process | Action | Result |
|---|---|---|---|
| e1 | P | local event | C(P) = 1 |
| e2 | P | send to Q | C(P) = 2, msg carries ts=2 |
| e3 | Q | local event | C(Q) = 1 |
| e4 | Q | receive from P (ts=2) | C(Q) = max(1,2)+1 = **3** |
| e5 | Q | send to R | C(Q) = 4, msg carries ts=4 |
| e6 | R | local event | C(R) = 1 |
| e7 | R | receive from Q (ts=4) | C(R) = max(1,4)+1 = **5** |

Notice: `e2 → e4` (P sent, Q received), and indeed `C(e2)=2 < C(e4)=3`. The guarantee holds.

![Lamport clock space-time diagram](/assets/img/dist-time/fig09-lamport-clock-trace.png)
*Figure 4 — Lamport clock trace. Causal arrows always point from lower to higher timestamps. Concurrent events (e.g. P's e1 and Q's e3) happen to get timestamps 1 and 1 — any relative order is valid.*

### The Limitation

Lamport clocks satisfy: `a → b  ⟹  C(a) < C(b)`

But the **converse is false**: `C(a) < C(b)` does **not** imply `a → b`.

Two completely unrelated events on different processes will still have a Lamport ordering between them — but that ordering is meaningless. You cannot tell from Lamport timestamps alone whether two events are causally related or concurrent.

This matters practically. Suppose you want to detect write conflicts: did these two writes happen independently (concurrent — conflict!) or did one see the other (causal — no conflict). Lamport clocks cannot answer that question.

---

## Vector Clocks

Vector clocks fix the converse. Instead of one integer per process, each process maintains a **vector of integers** — one entry per process in the system.

> **Definition — Vector Clock**
>
> Process `i` in a system of `n` processes maintains `V[0..n-1]`, where `V[j]` represents how many events from process `j` this process knows about (directly or through received messages).
>
> - **Local event:** `V[i] += 1`
> - **Send message:** `V[i] += 1`, attach full vector `V` to message
> - **Receive message:** `V[k] = max(V[k], msg.V[k])` for all k, then `V[i] += 1`
>
> **Guarantee (Strong Clock Condition):** `a → b`  **if and only if**  `V(a) < V(b)` (element-wise)
>
> The biconditional holds in **both directions**. You can detect concurrency definitively.

```python
class VectorClock:
    def __init__(self, n: int, i: int):
        self.n = n
        self.i = i
        self.V = [0] * n

    def tick(self):
        self.V[self.i] += 1
        return self.V.copy()

    def send(self, message: dict) -> dict:
        self.V[self.i] += 1
        message['vc'] = self.V.copy()
        return message

    def receive(self, message: dict):
        for k in range(self.n):
            self.V[k] = max(self.V[k], message['vc'][k])
        self.V[self.i] += 1
        return self.V.copy()


def happened_before(Va, Vb) -> bool:
    return all(Va[k] <= Vb[k] for k in range(len(Va))) and Va != Vb

def concurrent(Va, Vb) -> bool:
    return not happened_before(Va, Vb) and not happened_before(Vb, Va)
```

### How to compare two vector timestamps

Given `Va = [2, 3, 0]` and `Vb = [3, 1, 1]`:
- Is `Va < Vb` element-wise? `2≤3` ✓, but `3>1` ✗ — No.
- Is `Vb < Va` element-wise? `3>2` ✗ — No.
- Neither dominates → **concurrent**.

Given `Va = [2, 1, 0]` and `Vb = [3, 2, 1]`:
- Is `Va < Vb` element-wise? `2≤3` ✓, `1≤2` ✓, `0≤1` ✓ — Yes.
- **`a → b`** (a happened-before b).

![Vector clock comparison examples](/assets/img/dist-time/fig10-vector-clock-comparison.png)
*Figure 5 — Three comparison cases: happened-before, happened-after, and concurrent. The bar charts show element-wise values; when neither vector dominates, the events are concurrent.*

### Where vector clocks are used

**Amazon Dynamo** uses version vectors (a variant) to detect write conflicts in its key-value store. When two replicas each accepted a write that neither saw the other's, the vectors will be concurrent — flagging a conflict for the application to resolve. This is how Dynamo implements its "eventually consistent, always available" model.

**Riak** extends this with dotted version vectors, which more precisely track which replica performed a write.

**CRDTs** (Conflict-free Replicated Data Types — used in collaborative editors like Figma) use causality tracking built on the same foundation.

### The size problem

Vector clocks scale `O(n)` with the number of processes. In a system with thousands of nodes, attaching a full vector to every message becomes expensive. This motivated variants:
- **Version vectors** — one entry per storage replica rather than per client
- **Dotted version vectors** — separate the "what happened" from "who did it"
- **Interval tree clocks** — handle dynamic process creation and deletion

---

## Physical Time Done Right — TrueTime

Logical clocks solve causality but give up physical time entirely. Google took a different route with Spanner: keep physical time, but reason explicitly about its uncertainty.

> **Definition — TrueTime**
>
> TrueTime is Google's time API, backed by GPS receivers and atomic clocks in every datacenter. Instead of returning a single timestamp, it returns a **guaranteed interval** `[earliest, latest]`. The true current time is guaranteed to be somewhere inside that interval.
>
> The width of the interval — called **epsilon (ε)** — is typically **1–7 ms** under normal conditions.

```python
# TrueTime API
TT.now()     # → { earliest: t_e, latest: t_l }
TT.after(t)  # → True iff t is definitely in the past (true_time > t)
TT.before(t) # → True iff t is definitely in the future
```

### Commit wait — how Spanner buys linearizability

Spanner needs to guarantee that if transaction T1 commits before T2 starts in real time, then T1's timestamp is lower than T2's. With uncertain clocks this seems impossible. Here is how they do it:

1. At commit time, pick `S = TT.now().latest` — the upper bound of current time
2. **Wait** until `TT.after(S)` — until true time has definitely passed `S`
3. Commit with timestamp `S`

Now any transaction that starts after this commit will call `TT.now()` and get `earliest > S` — guaranteed. The commit timestamp ordering matches real-time ordering. That is **linearizability**.

![TrueTime commit wait diagram](/assets/img/dist-time/fig11-truetime-commit-wait.png)
*Figure 6 — Commit wait in action. Spanner waits out the clock uncertainty window (ε ≈ 1–7 ms) before committing, ensuring any later transaction sees a strictly larger timestamp.*

The cost is a 1–7 ms delay at each commit. Most systems consider global linearizability without specialised hardware to be impossible. Spanner achieves it by measuring the impossible precisely and waiting it out.

### Hybrid Logical Clocks

Most systems cannot put GPS receivers in every datacenter. **Hybrid Logical Clocks (HLC)**, proposed by Kulkarni et al. (2014), offer a practical middle ground: track physical time as closely as NTP allows, but fall back to a logical counter when the wall clock has not advanced.

```python
class HybridLogicalClock:
    """(physical_ms, logical_counter) — stays close to wall time
    while maintaining Lamport clock guarantees."""

    def __init__(self):
        self.pt = 0   # physical component (ms)
        self.l  = 0   # logical counter

    def tick(self) -> tuple:
        wall = self._wall_ms()
        if wall > self.pt:
            self.pt, self.l = wall, 0
        else:
            self.l += 1
        return (self.pt, self.l)

    def receive(self, msg_pt: int, msg_l: int) -> tuple:
        wall = self._wall_ms()
        new_pt = max(self.pt, msg_pt, wall)
        if   new_pt == self.pt == msg_pt: self.l = max(self.l, msg_l) + 1
        elif new_pt == self.pt:           self.l += 1
        elif new_pt == msg_pt:            self.l = msg_l + 1
        else:                             self.l = 0
        self.pt = new_pt
        return (self.pt, self.l)

    def _wall_ms(self) -> int:
        import time; return int(time.time() * 1000)
```

HLC is used in **CockroachDB** and **YugabyteDB** as a way to get roughly-physical timestamps with Lamport guarantees — no GPS hardware required.

---

## The Consistency Hierarchy

Every distributed database makes an explicit choice about which ordering guarantees it provides. These choices sit on a spectrum:

| Model | Guarantee | Cost | Example |
|---|---|---|---|
| **Eventual consistency** | Replicas converge eventually, no ordering guarantee | Lowest latency, highest availability | Cassandra, DynamoDB (default) |
| **Causal consistency** | Causally related ops seen in order by all nodes | Moderate — requires vector tracking | MongoDB causal sessions |
| **Sequential consistency** | All nodes see the same total order of ops | Higher — requires coordination | ZooKeeper (ZAB), etcd (Raft) |
| **Linearizability** | Total order matches real-time wall clock | Highest — requires consensus or bounded clocks | Google Spanner, FoundationDB |

![Consistency hierarchy pyramid](/assets/img/dist-time/fig12-consistency-hierarchy-pyramid.png)
*Figure 7 — The consistency hierarchy. Moving up gives stronger guarantees; moving down gives better availability and latency. Most systems let you choose per-operation.*

There is a well-known equivalence worth knowing: **Total Order Broadcast ≡ Consensus ≡ Atomic Commit**. These three problems reduce to each other. Any system that solves one can solve the others. This is why Paxos and Raft appear everywhere — they are consensus algorithms, and consensus is the primitive that strong consistency is built on.

---

## Summary

| Mechanism | What problem it solves | Key guarantee | Limitation |
|---|---|---|---|
| **NTP wall clock** | Rough global time reference | ±1–50 ms to UTC | Can jump backward; too coarse for event ordering |
| **Monotonic clock** | Measuring elapsed time | Never goes backward | Local only — meaningless across machines |
| **Lamport clock** | Total ordering of events | `a→b ⟹ C(a)<C(b)` | Cannot detect concurrency (converse fails) |
| **Vector clock** | Detecting causality and concurrency | `a→b ⟺ V(a)<V(b)` | Size scales O(n) with processes |
| **TrueTime** | Linearizable physical timestamps | Bounded uncertainty ε ≈ 1–7 ms | Requires GPS/atomic clock hardware |
| **Hybrid Logical Clock** | Physical + logical, no hardware | Close to wall time + Lamport props | Approximate, not exact physical ordering |

The progression tells a story. Physical clocks are unreliable for ordering across machines. Lamport clocks give you a consistent order but cannot distinguish causality from coincidence. Vector clocks give you full causal knowledge but at a size cost. TrueTime gives you physical ordering back — but only by measuring and waiting out the uncertainty.

There is no free lunch. Every mechanism trades something. The right choice depends on what your system needs to be correct.

---

## Where to Go Next

**The paper you must read:**
- Lamport, L. "Time, Clocks, and the Ordering of Events in a Distributed System." *CACM*, 1978. Free at [lamport.azurewebsites.net/pubs/time-clocks.pdf](https://lamport.azurewebsites.net/pubs/time-clocks.pdf). Six pages. Read it.

**For vector clocks:**
- Mattern, F. "Virtual Time and Global States of Distributed Systems." 1989. — the definitive theoretical treatment.
- Fidge, C. "Timestamps in Message-Passing Systems." 1988. — independent discovery with a slightly different presentation.

**For production systems:**
- Kleppmann, M. *Designing Data-Intensive Applications.* O'Reilly, 2017. Chapters 8 and 9 are the best engineering treatment of these ideas in print.
- Corbett et al. "Spanner: Google's Globally Distributed Database." *OSDI 2012.* — TrueTime and linearizability at global scale.
- DeCandia et al. "Dynamo: Amazon's Highly Available Key-value Store." *SOSP 2007.* — vector clocks in production.

**For hybrid approaches:**
- Kulkarni et al. "Logical Physical Clocks and Consistent Snapshots." 2014. arXiv:1409.7349. — the HLC paper.

**Free course:**
- MIT 6.824 Distributed Systems (MIT OpenCourseWare). Lectures 2–4 cover exactly this material with lab assignments.

---

## References

[1] Lamport, L. "Time, Clocks, and the Ordering of Events in a Distributed System." *CACM* 21(7), 1978.

[2] Fidge, C. "Timestamps in Message-Passing Systems That Preserve the Partial Ordering." *ACSC*, 1988.

[3] Mattern, F. "Virtual Time and Global States of Distributed Systems." *Parallel and Distributed Algorithms*, Elsevier, 1989.

[4] DeCandia, G. et al. "Dynamo: Amazon's Highly Available Key-value Store." *SOSP 2007.*

[5] Corbett, J. C. et al. "Spanner: Google's Globally Distributed Database." *OSDI 2012.*

[6] Kulkarni, S. et al. "Logical Physical Clocks and Consistent Snapshots in Globally Distributed Databases." *OPODIS 2014.* arXiv:1409.7349.

[7] Kleppmann, M. *Designing Data-Intensive Applications.* O'Reilly, 2017.

[8] Mills, D. L. *Network Time Protocol (Version 4).* RFC 5905, IETF, 2010.