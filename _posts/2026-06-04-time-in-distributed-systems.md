---
title: "Time in Distributed Systems"
date: 2026-06-04 00:00:00 +0000
categories: [distributed-systems, theory]
tags: [distributed-systems, clocks, lamport, vector-clocks, ntp, spanner, causality]
---

# Time in Distributed Systems
# From Physics to Distributed Algorithms

From the oscillation of atoms and crystals, through the chaos of networked machines, to the elegant mathematics of logical clocks — a ground-up journey through one of the hardest problems in computer science.

---

## Before We Begin — Why Is Time Hard?

In daily life, time feels obvious. You glance at your phone — `14:32:07`. Every device in the room roughly agrees. When you say "I sent the email before the meeting," everyone understands that ordering without question. Time feels *shared, continuous, and global*.

Distributed systems shatter this completely. Each machine has its own clock. Clocks drift apart over time. Messages take unpredictable time to travel. There is no way for any machine to know the "true" current time, or to confirm whether its view of "now" matches another machine's "now."

> **The Core Problem:** In a distributed system, **there is no global clock**. Each machine's clock is an independent physical device that drifts at its own rate. Clock synchronization over a network introduces uncertainty bounded by network latency. There is no shared notion of simultaneity — a fact with deep roots in both physics and engineering.

We will build intuition from the ground up: starting with what time *is* physically and how clocks actually work at the hardware level, then understanding exactly how distributed systems break that intuition, then exploring the elegant abstractions — Lamport clocks, vector clocks, TrueTime — that engineers have invented to reason about time without ever needing perfect clock agreement.

---

# Section 01 — The Physics of Time

Before understanding how computers measure time, we need to understand what time *is* and how it is measured in physics. This is not just philosophy — the properties of time measurement at the physical level directly determine what guarantees you can make in software.

---

## What Is Time, Physically?

In classical Newtonian physics, time was assumed to be absolute — a universal backdrop against which all events could be stamped with a unique moment. Newton wrote in *Principia Mathematica* (1687):

> "Absolute, true, and mathematical time, of itself, and from its own nature, flows equably without relation to anything external."
>
> — Isaac Newton, *Principia Mathematica*, 1687 [1]

Einstein's Special Relativity (1905) demolished this. Time is *relative to the observer's frame of reference*. Two observers moving at different velocities will disagree about the duration between the same two events — and both are correct within their own frames. Crucially, there is no universal "now" that applies everywhere simultaneously. [2]

> **Does Relativity Matter in Practice?** For most distributed systems, relativistic effects are negligible. GPS satellites, however, orbit at ~14,000 km/h and their clocks run fast by about 38 microseconds per day due to relativistic effects. GPS software explicitly corrects for this [3]. The deeper point: even in classical physics, there is no mechanism by which two spatially separated clocks can stay perfectly synchronized — they are fundamentally independent physical devices.

---

## How Do We Measure Time?

All clocks work by the same principle: **counting periodic oscillations** of something stable.

> **Definition — Oscillation (Physics · Mechanics)**
>
> An oscillation is a repeated back-and-forth motion around an equilibrium position. A pendulum swings left and right. A guitar string vibrates. All of these are oscillators. The number of complete cycles per second is the **frequency**, measured in Hertz (Hz). One Hz = one cycle per second. A clock counts these cycles — after N cycles of a known frequency, exactly N/frequency seconds have passed. The entire challenge of clockmaking is keeping that frequency perfectly stable. [4, 17]

---

## A Brief History of Timekeeping

| Clock Type | Oscillator | Frequency | Drift Per Day |
|---|---|---|---|
| Sundial | Earth's rotation | ~11.6 μHz | Seconds to minutes |
| Pendulum clock (1656) | Pendulum swing | ~1–2 Hz | ~1–10 seconds |
| Mechanical watch | Balance wheel spring | ~3–10 Hz | ~0.5–5 seconds |
| Quartz oscillator (1927) | Crystal vibration | 32,768 Hz | ~15 ms – 1 second |
| Cesium atomic clock (1955) | Electron transitions | 9,192,631,770 Hz | < 1 nanosecond |
| Optical lattice clock (2010s) | Optical laser transitions | ~10¹⁵ Hz | < 0.1 femtoseconds |

*Sources: Marrison (1948) [16], Essen & Parry (1955) [14], Ludlow et al. (2015) [18]*

Each step improved stability by orders of magnitude. The jump from mechanical to quartz is roughly 1000x. The jump from quartz to atomic is another 1,000,000x.

![Timekeeping evolution diagram — a vertical timeline from sundial at the top to optical lattice clock at the bottom, with each row showing the oscillator type, a small icon, and its drift per day. Colors get progressively more precise (warm red → cool blue). Label each step with the year it became practical.](/assets/img/dist-time/fig01-timekeeping-evolution.png)
*Figure 1 — The evolution of timekeeping from sundials to optical atomic clocks, each step improving stability by orders of magnitude.*

---

# Section 02 — Quartz Oscillators — Inside Your Computer's Clock

Every computer, smartphone, and server contains a quartz crystal oscillator as its primary timekeeping mechanism. Understanding how it works — and why it drifts — is directly relevant to why distributed systems have clock problems.

---

## What Is Quartz?

> **Definition — Quartz (SiO₂) (Chemistry · Mineralogy)**
>
> Quartz is a mineral composed of silicon dioxide (SiO₂) — the same compound as glass and sand. Its atoms are arranged in a highly ordered **crystalline lattice**: silicon atoms bonded to four oxygen atoms in a tetrahedral arrangement, repeating in a regular three-dimensional pattern. It is the second most abundant mineral in Earth's continental crust. What makes quartz special for timekeeping is that it is both *piezoelectric* and has very stable mechanical resonance properties. [Nye, 1957]

---

## What Is Piezoelectricity?

> **Definition — Piezoelectricity (Physics · Materials Science)**
>
> Piezoelectricity (from Greek *piezein* = to squeeze) is a property of certain crystals whereby **mechanical stress produces an electric charge**, and conversely, **an applied electric voltage produces mechanical deformation**. It was discovered by Pierre and Jacques Curie in 1880 [4].
>
> In quartz specifically: when you squeeze or stretch the crystal, the asymmetric arrangement of Si and O atoms causes a net electric dipole moment across the crystal, generating a measurable voltage. When you apply a voltage, the crystal physically deforms by a tiny amount. This two-way coupling between mechanical and electrical energy — *stress → charge, voltage → strain* — is what makes quartz so useful as an oscillator.

![Piezoelectric effect diagram — two panels side by side. Left panel: a quartz crystal at rest, silicon and oxygen atoms shown in a lattice, no net charge. Right panel: the same crystal under compression (arrows pushing inward from top and bottom), showing the atoms slightly displaced, with + charges on one face and − charges on the other, and a voltmeter across the faces reading a voltage. Label Si and O atoms, label the voltage output, label the two effects: Direct Effect (stress → voltage) and Inverse Effect (voltage → stress).](/assets/img/dist-time/fig02-piezoelectric-effect.png)
*Figure 2 — The piezoelectric effect in quartz. Mechanical stress displaces the crystal's atoms, separating charge and producing a measurable voltage. Applying voltage causes the reverse: physical deformation.*

---

## How a Quartz Oscillator Works

A quartz crystal is cut into a precise shape (often a thin rectangular plate or tuning-fork shape) so that it has a well-defined **resonant frequency** — the frequency at which it naturally vibrates most efficiently, like how a wine glass has a natural pitch it rings at when struck.

> **Definition — Resonant Frequency & Q Factor (Physics · Acoustics)**
>
> Every elastic object has one or more **natural frequencies** at which it oscillates with maximum amplitude when disturbed. For a quartz crystal, this frequency is set by its physical dimensions and cut angle. The thicker the crystal, the lower its resonant frequency.
>
> The **Q factor** (quality factor) is a dimensionless measure of how "sharp" the resonance is — how precisely the frequency is defined and how long the crystal rings without energy loss. A pendulum has Q ≈ 100. A quartz crystal has Q ≈ 10,000–100,000. This is why quartz oscillators are so much more stable than mechanical clocks. [15]

The oscillator circuit works as follows:

1. **An oscillator circuit applies an alternating voltage to the crystal.** The crystal deforms slightly at the applied frequency.
2. **At the resonant frequency, the crystal vibrates with maximum amplitude.** The high Q factor means it loses very little energy per cycle and oscillates very stably.
3. **The vibrating crystal generates a voltage signal via piezoelectricity.** This signal is amplified and fed back into the circuit to maintain the oscillation.
4. **A digital counter counts the cycles.** A 32,768 Hz crystal (= 2¹⁵ Hz) is standard because it divides down to exactly 1 Hz with 15 successive divide-by-2 operations in binary hardware — a cheap and elegant trick.

![Quartz oscillator circuit diagram — a clean schematic showing: (1) a tuning-fork shaped quartz crystal in the centre, labelled "32,768 Hz resonator"; (2) an amplifier connected in a feedback loop around the crystal; (3) a digital counter receiving the oscillator output and counting down 2^15 divisions to produce a 1 Hz tick; (4) arrows showing the signal flow: oscillator → crystal → amplifier → back to crystal, and a branch from oscillator → ÷32768 counter → 1 Hz output. Use a clean dark background with white/teal lines.](/assets/img/dist-time/fig03-quartz-oscillator-circuit.png)
*Figure 3 — A quartz oscillator circuit. The piezoelectric crystal stabilises the feedback loop at its resonant frequency. The binary counter divides 32,768 Hz down to 1 Hz.*

---

## Why Quartz Clocks Drift

Despite their elegance, quartz oscillators are imperfect timekeepers. Their resonant frequency shifts due to several physical effects:

| Cause | Mechanism | Typical Effect |
|---|---|---|
| **Temperature** | Thermal expansion changes crystal dimensions and thus resonant frequency | 10–100 ppm per 10°C change |
| **Supply voltage** | Changes in power supply alter electronic properties of oscillator circuit | 0.1–1 ppm per volt |
| **Aging** | Slow structural changes: mass redistribution, stress relief, contamination | 1–10 ppm per year |
| **Shock & vibration** | Physical impact can permanently shift resonant frequency | Variable |

A typical computer-grade quartz oscillator drifts by **100–200 ppm** (parts per million). That is 8–17 seconds per day, or roughly **1 millisecond every 10 seconds**.

> **Drift in Distributed Systems Context**
>
> Two servers that last synchronized 10 seconds ago could already disagree by 2 ms. Over a minute without NTP correction, the skew can reach 10–20 ms. This is the physical root cause of clock skew in distributed systems. [5]

---

# Section 03 — Atomic Clocks — The Gold Standard

The instability of quartz oscillators motivated the search for a fundamentally more stable periodic phenomenon. Physicists found it in the quantum mechanics of atoms.

---

## What Is an Atomic Clock?

> **Definition — Atomic Clock (Physics · Quantum Mechanics)**
>
> An atomic clock uses the precisely fixed frequency of electromagnetic radiation emitted or absorbed when an atom transitions between two specific energy levels. Because **quantum energy levels are determined by fundamental physical constants** — not by physical dimensions that can change — atomic clocks are extraordinarily stable.
>
> The frequency of the cesium-133 hyperfine transition is so stable that since 1967, the **SI second has been defined** as exactly 9,192,631,770 oscillations of this transition. [6]

---

## The Cesium Hyperfine Transition

> **Definition — Hyperfine Transition (Quantum Mechanics · Spectroscopy)**
>
> Cesium-133 has a single valence electron in its outermost shell. The nucleus has a **spin** (angular momentum), and the electron also has a spin. These two spins can be either aligned (parallel ↑↑) or anti-aligned (antiparallel ↑↓). The energy difference between these two states is extremely small — this is called a **hyperfine splitting** because it is far smaller than the main electron energy levels.
>
> When a cesium atom transitions between these two hyperfine states, it emits or absorbs a photon of electromagnetic radiation at exactly **9,192,631,770 Hz** — in the microwave range (~3.26 cm wavelength). This frequency depends only on fundamental properties of the cesium nucleus and electron, which do not change. [14]
>
> In a cesium clock: a beam of cesium atoms is prepared in one spin state, passed through a microwave cavity tuned to 9.19 GHz, and then detected. The microwave frequency is continuously adjusted to maximise the transition rate — locking onto the exact quantum resonance. The locked microwave oscillator provides the clock's reference.

![Cesium atomic clock schematic — a horizontal diagram showing: (1) an oven on the left emitting a beam of cesium atoms; (2) a state selector magnet that filters out atoms in the wrong spin state; (3) a microwave cavity (Ramsey cavity) in the middle with a 9.192 GHz signal applied; (4) a detector on the right measuring how many atoms transitioned; (5) a feedback loop from detector back to the microwave frequency source, labelled "frequency lock"; (6) the output labelled "9,192,631,770 Hz = 1 second per 9.19 billion cycles". Show the two spin states ↑↑ and ↑↓ with a small energy diagram inset.](/assets/img/dist-time/fig04-cesium-atomic-clock.png)
*Figure 4 — Schematic of a cesium beam atomic clock. The microwave frequency is continuously locked to the cesium-133 hyperfine transition, producing the world's most stable frequency reference.*

> **The Definition of a Second**
>
> Since the 13th General Conference on Weights and Measures (1967), one second is defined as: *"the duration of 9,192,631,770 periods of the radiation corresponding to the transition between the two hyperfine levels of the ground state of the cesium-133 atom."* [6] This is not derived from astronomy. It was chosen to match the previous astronomical second to high precision. UTC itself is maintained by averaging ~400 atomic clocks worldwide.

---

## GPS and Time Distribution

> **Definition — GPS (Global Positioning System) (Engineering · Space Systems)**
>
> GPS is a satellite-based system operated by the U.S. Department of Defense. Each GPS satellite carries multiple atomic clocks (typically 2 cesium + 2 rubidium) and continuously broadcasts its precise time and position. A receiver determines its position by measuring the **time of arrival** of signals from at least four satellites — since the signal travels at the speed of light, timing accuracy directly gives position accuracy.
>
> GPS receivers can determine absolute time to within ~100 nanoseconds of UTC. Time servers at the top of the NTP hierarchy use GPS receivers. Google's Spanner/TrueTime infrastructure places GPS receivers in every datacenter specifically for this sub-microsecond accuracy. [7]
>
> **Relativistic corrections:** GPS satellites orbit at ~20,200 km altitude. Special relativity slows their clocks by ~7 µs/day (velocity). General relativity speeds them up by ~45 µs/day (gravity). Net: +38 µs/day, which GPS software corrects explicitly. [3]

![GPS time distribution diagram — three layers stacked vertically. Top layer: space, showing 3 GPS satellites in orbit, each with a label "Cs + Rb atomic clocks, ±100 ns to UTC". Middle layer: ground infrastructure, showing a GPS antenna on a building roof feeding a "Stratum 1 NTP server". Bottom layer: data centre, showing multiple servers connected to the NTP server via local network, each with a label showing ±1–50 ms accuracy. Arrows show the signal flowing downward. Include a small inset showing the relativistic correction: +38 µs/day corrected.](/assets/img/dist-time/fig05-gps-time-distribution.png)
*Figure 5 — How time flows from GPS satellites to your servers. Each layer adds uncertainty: from ±100 ns at GPS receivers to ±1–50 ms at typical servers.*

---

# Section 04 — NTP — Synchronising Clocks Over a Network

---

## What Is NTP?

> **Definition — NTP (Network Time Protocol) (Networking · RFC 5905)**
>
> NTP is a protocol for synchronising computer clocks over a network, designed by David L. Mills at the University of Delaware in 1985. [8] It uses a hierarchical system of time servers called **strata**. Stratum 0 sources are directly attached to atomic clocks or GPS receivers. Each stratum level adds ~1–20 ms of uncertainty. NTP clients continuously **discipline** their local oscillators, making small frequency adjustments rather than abrupt time jumps.
>
> NTP measures **round-trip delay** to estimate one-way latency, then adjusts the local clock accordingly. If the local clock is off by less than 125 ms, NTP *slews* it (adjusts the rate gradually). If off by more than 125 ms, NTP may *step* it (jump abruptly). Abrupt steps are dangerous for software that assumes time never goes backward.

![NTP stratum hierarchy — a pyramid diagram with 3 tiers. Tier 1 (top, smallest, dark background): "Stratum 0 — Reference Sources: Cesium atomic clocks / GPS / Radio (DCF77)". Tier 2 (middle, medium, slate): "Stratum 1 — Primary Time Servers: directly connected to Stratum 0, accuracy ±microseconds, examples: time.google.com, time.nist.gov". Tier 3 (bottom, largest, teal): "Stratum 2 — Secondary Servers: pool.ntp.org, accuracy ±1–50 ms, your servers sync here". Dashed arrows pointing downward between layers labelled "NTP sync". On the right side, a vertical axis labelled "Accuracy" going from "±microseconds" at top to "±50 ms" at bottom.](/assets/img/dist-time/fig06-ntp-stratum-hierarchy.png)
*Figure 6 — The NTP stratum hierarchy. Each level introduces additional uncertainty. Most production servers sit at Stratum 2 or 3, with ±1–50 ms accuracy.*

---

## Monotonic vs. Wall Clock

> **Definition — Monotonic Clock (Operating Systems)**
>
> A **monotonic clock** only ever increases — it never goes backward, and is never adjusted by NTP. It measures time elapsed since an arbitrary point (usually system boot).
>
> A **wall clock** (real-time clock) reports the current UTC time and *can* be adjusted by NTP — it can jump forward or backward.
>
> | | Wall Clock | Monotonic Clock |
> |---|---|---|
> | Linux | `CLOCK_REALTIME` | `CLOCK_MONOTONIC` |
> | Java | `System.currentTimeMillis()` | `System.nanoTime()` |
> | Python | `time.time()` | `time.monotonic()` |
> | Go | `time.Now()` | `time.Since()` |
>
> **Rule:** Use wall clocks for timestamps you store or display. Use monotonic clocks for measuring durations, timeouts, and intervals. **Never subtract two wall-clock readings to measure elapsed time.** [5]

> **⚠ The NTP Backward Jump**
>
> If your server's clock drifts more than 125 ms ahead, NTP will step it backward — suddenly the wall clock shows a time in the past. Any code using wall-clock arithmetic for durations will compute negative elapsed times, incorrect timeouts, or stale lease timestamps. This has caused real production outages. Always use monotonic clocks for elapsed time. [5]

---

# Section 05 — Why Physical Time Fails Distributed Systems

Now that we understand the full physical stack — quartz oscillators, atomic clocks, NTP — we can see precisely why physical time is still insufficient for distributed systems, even with NTP running.

---

## Scenario: Last-Write-Wins with Clock Skew

Two servers — Node A and Node B — both accept writes to a replicated key-value store with *last-write-wins* conflict resolution.

- Node A writes `"alice"` at its local clock time `T = 1000 ms`
- Node B writes `"bob"` at its local clock time `T = 999 ms`

The system concludes Node A's write was "later" and keeps `"alice"`. But what if Node B's quartz oscillator was running 1 ms fast? In real time, Node B wrote *after* Node A. Node B's write should win — but the clock skew reversed the apparent order. We just **silently lost Node A's write** with no error, no warning.

> "The fact that it is impossible to say in an absolute sense whether one of two events occurred first is a fundamental difficulty, and not just the result of our lack of clever engineering."
>
> — Leslie Lamport, "Time, Clocks, and the Ordering of Events in a Distributed System," CACM 1978 [9]

![Last-write-wins clock skew diagram — a horizontal timeline at the bottom labelled "Real time →". Two swim lanes above it: "Node A" and "Node B". Node A has a write event at real-time position 100. Node B has a write event at real-time position 105 (later). But Node A's clock shows T=1000 ms and Node B's clock shows T=999 ms (because Node B's quartz runs 1 ms fast). A red arrow labelled "Clock skew reverses order" points to the discrepancy. A box on the right shows the incorrect outcome: "System keeps 'alice' (Node A) — wrong!" with a red X, and the correct outcome below: "Should keep 'bob' (Node B)" with a green tick.](/assets/img/dist-time/fig07-clock-skew-last-write-wins.png)
*Figure 7 — Clock skew silently corrupting last-write-wins conflict resolution. Node B wrote later in real time but has a lower timestamp due to its quartz oscillator running fast.*

---

## The Three Fundamental Failure Modes

**1. Clock skew is unavoidable**

NTP synchronises to within 1–50 ms over the internet. But "under 1 ms" still means two machines can disagree by up to 1,000,000 nanoseconds. Events closer together in time than the skew bound cannot be reliably ordered by timestamp.

**2. Network delays are variable and unbounded**

A message from A to B might take 0.5 ms normally, or 500 ms during congestion. When you receive a timestamped message, you cannot determine how long it was in transit — you cannot infer the sender's "current time" from the timestamp.

**3. Processes can pause for arbitrary durations**

A process can be preempted by the OS scheduler, stop-the-world garbage collected, paged out to disk, or live-migrated as a VM — pausing for milliseconds to minutes. When it resumes, its wall clock appears correct but its understanding of "now" is stale.

> **From DDIA — The Paused Process Problem [5]**
>
> Kleppmann illustrates this in Chapter 8: a process holds a lease lock, pauses for 30 seconds due to garbage collection, then resumes and attempts to write — believing it still holds the lock. In that 30 seconds, the lease has expired and another process acquired it. The first process's write will corrupt shared state. **No clock-based mechanism can prevent this** because the paused process cannot distinguish "I paused briefly" from "I paused for 30 seconds."

![Process pause lease violation — a sequence diagram with two swim lanes: "Process A" and "Process B", and a shared resource "Distributed Lock" in the middle. Timeline flows downward. Step 1: Process A acquires lease, valid for 10 seconds. Step 2: Process A is shown pausing (grey box labelled "GC pause: 30 seconds"). Step 3: While A is paused, the lease expires (dashed horizontal line). Step 4: Process B acquires the lease. Step 5: Process A resumes, still believes it holds the lease, writes to the shared resource. Step 6: Red X labelled "Conflict — data corrupted". Annotate with "Wall clock looks fine from A's perspective".](/assets/img/dist-time/fig08-process-pause-lease-violation.png)
*Figure 8 — A process pause longer than the lease duration causes two processes to simultaneously believe they hold an exclusive lock. Physical clocks cannot detect this.*

---

# Section 06 — The Shift to Logical Time

The physical clock problems motivate a crucial conceptual shift. Lamport's 1978 insight: for most distributed system correctness questions, we do not need to know *when* something happened in wall-clock time. We only need to know the **causal relationship** between events.

---

## Causality and Happened-Before

> **Definition — Causality in Distributed Systems (Theory · Distributed Systems)**
>
> Event A is **causally before** event B if A *could have influenced* B. This happens in two cases: (1) A and B are on the same process and A happened before B in execution order, or (2) A sent a message that B received (or a chain of such messages connects them).
>
> If neither A is causally before B, nor B is causally before A, then A and B are **concurrent** — they had no causal relationship. Concurrent events are genuinely unordered: it is not just that we do not know the order, it is that no meaningful global order exists between them. [9, 11]

> **Definition — Happened-Before Relation (→) (Theory · Formal Definition)**
>
> Formally defined by Lamport (1978) [9], **→** is the smallest relation satisfying:
>
> 1. If a and b are events in the same process and a comes before b, then `a → b`
> 2. If a is the sending of a message and b is the receipt of that same message, then `a → b`
> 3. *(Transitivity)* If `a → b` and `b → c`, then `a → c`
>
> This defines a **strict partial order** — irreflexive, asymmetric, and transitive. It is *partial* because not all events are comparable: concurrent events satisfy neither `a → b` nor `b → a`.

| Relation | Notation | Meaning |
|---|---|---|
| Happened-before | `a → b` | a could have caused b; causal arrow points forward |
| Concurrent | `a ‖ b` | neither `a → b` nor `b → a`; genuinely unordered |
| Partial order | `→` | not every pair is comparable — unlike physical time |

---

# Section 07 — Lamport Clocks

Having defined the happened-before relation abstractly, Lamport needed a concrete mechanism to implement it. His solution: a simple logical counter that respects the causal order.

---

## What Is a Lamport Clock?

> **Definition — Lamport Clock (Logical Clock) (Algorithm · Distributed Systems)**
>
> A **Lamport clock** is an integer counter maintained by each process, advanced according to three rules, such that the counter values respect the happened-before relation: if `a → b`, then `C(a) < C(b)`. Lamport clocks assign a *consistent total order* to all events — though this total order is not unique and may not reflect real-time ordering of concurrent events. [9]

---

## The Three Rules

**Rule 1 — Local event:** before executing, increment `C := C + 1`

**Rule 2 — Send message:** increment, then include the timestamp:
```
C := C + 1
send(message, timestamp=C)
```

**Rule 3 — Receive message:** take the max of local and received, then increment:
```
C := max(C, C_received) + 1
```

---

## Implementation

```python
class LamportClock:
    """A Lamport logical clock for one process."""

    def __init__(self):
        self.counter = 0

    def local_event(self):
        """Call before any local event. Returns new timestamp."""
        self.counter += 1
        return self.counter

    def send(self, message):
        """Stamp a message before sending. Returns stamped message."""
        self.counter += 1
        message['timestamp'] = self.counter
        return message

    def receive(self, message):
        """Update clock on receiving a message. Returns new timestamp."""
        self.counter = max(self.counter, message['timestamp']) + 1
        return self.counter

# Key guarantee:
#   if a → b  then  C(a) < C(b)
#
# BUT the converse is NOT guaranteed:
#   C(a) < C(b)  does NOT imply  a → b
#   (concurrent events may receive any relative order)
```

---

## A Worked Example

Three processes P, Q, R exchanging messages. Let us trace the clock values:

| Event | Process | Rule Applied | Clock Value |
|---|---|---|---|
| e1 | P | Local | C(P)=1 |
| e2 | P | Send to Q | C(P)=2, message carries ts=2 |
| e3 | P | Local | C(P)=3 |
| e4 | Q | Local | C(Q)=1 |
| e5 | Q | Receive from P (ts=2) | C(Q)=max(1,2)+1=3 |
| e6 | Q | Send to R | C(Q)=4, message carries ts=4 |
| e7 | R | Local | C(R)=1 |
| e8 | R | Local | C(R)=2 |
| e9 | R | Receive from Q (ts=4) | C(R)=max(2,4)+1=5 |
| e10 | R | Send to P | C(R)=6, message carries ts=6 |
| e11 | P | Receive from R (ts=6) | C(P)=max(3,6)+1=7 |

Notice: e2 → e5 (P sent, Q received), so C(e2)=2 < C(e5)=3 ✓. The causal chain P→Q→R is faithfully reflected in the timestamps increasing.

![Lamport clock trace space-time diagram — three vertical lines representing processes P, Q, R from left to right. Time flows downward. Events are dots on each process line, labelled with their Lamport timestamp in a circle. Arrows between processes show messages: arrow from P's e2 (ts=2) to Q's e5 (ts=3), labelled "msg ts=2"; arrow from Q's e6 (ts=4) to R's e9 (ts=5), labelled "msg ts=4"; arrow from R's e10 (ts=6) to P's e11 (ts=7), labelled "msg ts=6". Colour the dots: teal for local events, gold for sends, rust/orange for receives. Include a legend. Annotate: "Clock condition: a→b implies C(a) < C(b)".](/assets/img/dist-time/fig09-lamport-clock-trace.png)
*Figure 9 — A Lamport clock trace across three processes. Causal arrows always go from lower to higher timestamps. Concurrent events (e.g. P's e3 and Q's e4) are unrelated and may get any relative timestamps.*

---

## The Critical Limitation

> **⚠ The Converse Is Not True**
>
> Lamport clocks satisfy the **Clock Condition**: `a → b` implies `C(a) < C(b)`.
>
> But `C(a) < C(b)` does **not** imply `a → b`. Two concurrent events on different processes might receive timestamps 3 and 7. You cannot tell from those timestamps alone whether a causal relationship exists. To actually detect concurrency, you need **vector clocks**.

---

# Section 08 — Vector Clocks

Vector clocks, introduced independently by Colin Fidge (1988) and Friedemann Mattern (1989), extend Lamport's idea to capture the full happened-before relation — not just a consistent order, but the exact causal structure. [10, 11]

---

## What Is a Vector Clock?

> **Definition — Vector Clock (Algorithm · Distributed Systems)**
>
> A **vector clock** is a vector of integers, one per process in the system. Process *i* maintains `V[1..n]`, where `V[j]` represents the number of events from process *j* that process *i* knows about (directly or transitively through messages).
>
> The key property: two vector timestamps can be compared to determine their causal relationship **definitively** — something Lamport clocks cannot do. [10, 11]

---

## The Algorithm

Process *i* out of *n* total:

```python
class VectorClock:
    """A vector clock for process i in a system of n processes."""

    def __init__(self, n: int, i: int):
        self.n = n
        self.i = i          # this process's index (0-based)
        self.V = [0] * n    # vector initialised to all zeros

    def local_event(self):
        """Increment own entry for a local event."""
        self.V[self.i] += 1
        return self.V.copy()

    def send(self, message: dict):
        """Stamp message before sending."""
        self.V[self.i] += 1
        message['vector'] = self.V.copy()
        return message

    def receive(self, message: dict):
        """Merge received vector, then increment own entry."""
        received = message['vector']
        # Element-wise max — merge causal knowledge
        for k in range(self.n):
            self.V[k] = max(self.V[k], received[k])
        self.V[self.i] += 1   # then count this receive event
        return self.V.copy()


def happened_before(Va: list, Vb: list) -> bool:
    """True if Va < Vb element-wise (a happened-before b)."""
    return all(Va[k] <= Vb[k] for k in range(len(Va))) and Va != Vb

def concurrent(Va: list, Vb: list) -> bool:
    """True if neither a→b nor b→a."""
    return not happened_before(Va, Vb) and not happened_before(Vb, Va)

# Strong Clock Condition (biconditional — both directions hold):
#   a → b  ⟺  V(a) < V(b)   (element-wise ≤, not equal)
#
# Unlike Lamport clocks, the converse IS guaranteed here.
# You can definitively detect concurrency with vector clocks.
```

---

## Comparing Vector Timestamps

Given two vector timestamps `Va` and `Vb`:

| Condition | Meaning |
|---|---|
| `Va[k] <= Vb[k]` for all k, and `Va ≠ Vb` | `a → b` (a happened-before b) |
| `Vb[k] <= Va[k]` for all k, and `Va ≠ Vb` | `b → a` (b happened-before a) |
| Neither of the above | `a ‖ b` (concurrent) |

![Vector clock comparison diagram — a 3x3 grid of small examples, each showing two vector timestamps [P, Q, R] and the conclusion. Example 1: Va=[2,1,0] Vb=[3,2,1] → "a → b" (all Va ≤ Vb). Example 2: Va=[3,2,1] Vb=[2,1,0] → "b → a" (all Vb ≤ Va). Example 3: Va=[2,3,0] Vb=[3,1,1] → "a ‖ b" (neither dominates — P: 2<3 but Q: 3>1). Each example uses coloured bars to visualise the vector values. Label each with the relation and a colour-coded verdict (teal=causal, rust=concurrent).](/assets/img/dist-time/fig10-vector-clock-comparison.png)
*Figure 10 — Comparing vector timestamps. Element-wise comparison determines causality. When neither vector dominates the other, the events are concurrent.*

---

## The Strong Clock Condition

> **Strong Clock Condition** [11]
>
> `a → b`  **if and only if**  `V(a) < V(b)`
>
> The biconditional holds in both directions. Unlike Lamport clocks, you can definitively answer both "did a causally precede b?" and "are a and b concurrent?" — just by comparing their vector timestamps.

---

## Real-World Use

- **Amazon Dynamo** [12]: uses version vectors (a variant of vector clocks) to detect conflicting writes to the same key. Conflicts surface at read time for application-level resolution.
- **Riak**: uses dotted version vectors — a modern refinement — for more precise conflict detection.
- **CRDTs** (Conflict-free Replicated Data Types): used in Figma, Google Docs collaborative editing, and others. Rely on causality tracking deeply related to vector clocks.

---

## Limitation: Size

Vector clocks scale linearly with the number of processes. In a system with thousands of nodes, vectors become impractically large. This led to variants:

- **Version vectors** — one entry per replica rather than per client
- **Dotted version vectors** — separate causality tracking from write identification
- **Interval tree clocks** — dynamic process creation/deletion

---

# Section 09 — TrueTime & Spanner

Logical clocks abandon physical time entirely. But Google took a different approach: *what if we made physical time reliable enough* by acknowledging its uncertainty explicitly?

---

## What Is TrueTime?

> **Definition — TrueTime API (Infrastructure · Spanner)**
>
> TrueTime is Google's globally distributed time service, first described in the 2012 Spanner paper [7]. Instead of returning a single timestamp, it returns an *interval* `[t_earliest, t_latest]`. The true current UTC time is **guaranteed** to lie within this interval. The interval width (epsilon, ε) reflects current clock uncertainty — typically **1–7 ms** under normal conditions.
>
> Infrastructure: every Google datacenter has time master servers equipped with GPS receivers and atomic clocks (rubidium oscillators). Machine-local daemons poll these masters, apply Marzullo's algorithm to intersect time intervals, and report the resulting bounded uncertainty to the TrueTime API.

```python
# TrueTime API — simplified interface

TT.now()      # → TTInterval { earliest: t_early, latest: t_late }
TT.after(t)   # → bool  — True iff t is definitely in the past
TT.before(t)  # → bool  — True iff t is definitely in the future

# Commit wait — how Spanner achieves external consistency:
#
# 1. At commit:  S = TT.now().latest   (upper bound on true time)
# 2. Wait until: TT.after(S) == True   (true time has passed S)
# 3. Now any transaction starting after this sees a TrueTime
#    lower bound > S — so it will see this transaction's effects.
#
# This gives linearizability across a globally distributed database.
```

---

## The Commit Wait Trick

1. **Acquire timestamp S = TT.now().latest** — this is an upper bound; the true time is at most S.
2. **Wait until TT.after(S)** — the true wall-clock time is now definitely past S.
3. **Commit is externally consistent** — any transaction starting after this point has a timestamp > S and will observe this transaction's effects.

The wait is typically 1–7 ms. This is the price of global linearizability.

![TrueTime commit wait diagram — a horizontal timeline for one server. Mark: (1) "Start commit" at position 0; (2) "S = TT.now().latest" — shown as the right edge of an uncertainty interval [t_early, t_late]; (3) a "WAIT" zone (grey shaded) from position S forward; (4) "TT.after(S) = True — commit complete" at position S + ε; (5) a second transaction starting after the commit, with its TT.now().earliest > S. Label the uncertainty interval as "ε = 1–7 ms typically". Annotate: "Waiting out the uncertainty window buys linearizability".](/assets/img/dist-time/fig11-truetime-commit-wait.png)
*Figure 11 — Spanner's commit wait. By waiting until the true time has definitely passed the commit timestamp, Spanner guarantees external consistency without a global lock.*

> **Why This Is Remarkable**
>
> Spanner achieves **external consistency** (linearizability) across a globally distributed system by reasoning explicitly about clock uncertainty — waiting out the uncertainty window before committing. Most systems consider global linearizability without specialised hardware to be essentially impossible; Spanner does it by *quantifying* the impossible and designing around it. [7]

---

## Hybrid Logical Clocks (HLC)

Most systems cannot afford GPS clocks in every datacenter. Hybrid Logical Clocks offer a practical alternative.

> **Definition — Hybrid Logical Clock (HLC) (Algorithm · 2014)**
>
> Proposed by Kulkarni et al. (2014) [13], a **Hybrid Logical Clock** is a `(physical_time, logical_counter)` pair. The physical component tracks NTP wall-clock time as closely as possible. The logical counter only increments when the wall clock has not advanced — providing Lamport clock guarantees even during NTP instability.
>
> Used in: CockroachDB, YugabyteDB, MongoDB global clusters.

```python
class HybridLogicalClock:
    """Hybrid Logical Clock — tracks (physical_time, logical_counter)."""

    def __init__(self):
        self.pt = 0   # physical component (NTP wall clock, ms)
        self.l  = 0   # logical component

    def tick(self):
        """Generate a new HLC timestamp for a local event or send."""
        wall = self._now_ms()
        if wall > self.pt:
            self.pt = wall
            self.l = 0
        else:
            self.l += 1
        return (self.pt, self.l)

    def receive(self, msg_pt: int, msg_l: int):
        """Update HLC on receiving a message."""
        wall = self._now_ms()
        new_pt = max(self.pt, msg_pt, wall)
        if   new_pt == self.pt == msg_pt: self.l = max(self.l, msg_l) + 1
        elif new_pt == self.pt:           self.l += 1
        elif new_pt == msg_pt:            self.l = msg_l + 1
        else:                             self.l = 0
        self.pt = new_pt
        return (self.pt, self.l)

    def _now_ms(self) -> int:
        import time
        return int(time.time() * 1000)
```

---

# Section 10 — Ordering Events: A Hierarchy

Let us synthesise everything into a clear hierarchy of ordering guarantees, from weakest to strongest.

| Level | Guarantee | Implementation |
|---|---|---|
| **No ordering** | Replicas converge eventually; may serve stale or reordered data | Eventual consistency (Cassandra, DynamoDB default) |
| **Causal consistency** | Causally related operations seen in causal order by all nodes | Vector clocks, causal+ consistency |
| **Sequential consistency** | All processes see the same total order; consistent with local program order | Zookeeper (ZAB), Raft |
| **Linearizability** | Total order consistent with real-time wall clock | Spanner (TrueTime), etcd (Raft) |

![Consistency hierarchy pyramid — a tall pyramid divided into 4 horizontal bands, labelled from bottom to top: "Eventual Consistency (weakest, cheapest)", "Causal Consistency", "Sequential Consistency", "Linearizability (strongest, most expensive)". On the left side, a vertical arrow pointing upward labelled "Stronger guarantees". On the right side, a vertical arrow pointing downward labelled "Higher availability / lower latency". Each band has a small icon of a representative system: Cassandra for eventual, vector clocks for causal, Raft log for sequential, Spanner globe for linearizability.](/assets/img/dist-time/fig12-consistency-hierarchy-pyramid.png)
*Figure 12 — The consistency hierarchy. Stronger guarantees require more coordination and sacrifice availability or latency. Each level builds on the one below.*

> **The Equivalence Triangle**
>
> These three problems are equivalent — solving any one gives you the others:
> **Total Order Broadcast ≡ Consensus ≡ Atomic Commit** (2PC variant)
>
> This equivalence, known since the 1980s, explains why consensus protocols (Paxos, Raft) are so central to distributed systems. Systems like Zookeeper (ZAB protocol), etcd (Raft), and Kafka (transactional mode) all implement total order broadcast as their core primitive.

---

# Section 11 — Where to Go Deeper

## Foundational Papers

- **Lamport (1978)** — "Time, Clocks, and the Ordering of Events in a Distributed System." *CACM 21(7).* The paper that started it all. Free at [lamport.azurewebsites.net](https://lamport.azurewebsites.net/pubs/time-clocks.pdf)

- **Fidge (1988)** — "Timestamps in Message-Passing Systems." Vector clocks, independent discovery.

- **Mattern (1989)** — "Virtual Time and Global States of Distributed Systems." Vector clocks, the definitive theoretical treatment.

- **DeCandia et al. (2007)** — "Dynamo: Amazon's Highly Available Key-value Store." *SOSP '07.* Production use of version vectors.

- **Corbett et al. (2012)** — "Spanner: Google's Globally Distributed Database." *OSDI '12.* TrueTime and external consistency.

- **Kulkarni et al. (2014)** — "Logical Physical Clocks and Consistent Snapshots." arXiv:1409.7349. Hybrid Logical Clocks.

## Books

- **Kleppmann, M.** — *Designing Data-Intensive Applications.* O'Reilly, 2017. Chapters 8–9 are the best practical treatment of time for engineers.

- **Tanenbaum & Van Steen** — *Distributed Systems: Principles and Paradigms.* 3rd ed. Chapter 6 on clock synchronisation. Free at [distributed-systems.net](https://www.distributed-systems.net)

- **Vitillo, R.** — *Understanding Distributed Systems.* 2021. Modern, concise; excellent on time and clocks.

- **Petrov, A.** — *Database Internals.* O'Reilly, 2019. Part II on distributed systems and ordering.

## Blogs & Talks

- **Kleppmann, M.** — "Clocks and Time in Distributed Systems." Strange Loop talk. YouTube. Visual, accessible.

- **Kingsbury, K.** — Jepsen blog at [jepsen.io](https://jepsen.io). Real-world analyses of how databases handle consistency.

- **Lamport, L.** — Collected papers and commentary at [lamport.azurewebsites.net](https://lamport.azurewebsites.net). Primary source.

- **Sheehy, J.** — "There Is No Now." ACM Queue, 2015. Short, essential essay.

## Courses

- **MIT 6.824** — Distributed Systems. Free at MIT OpenCourseWare. Lectures 2–4 on time and ordering.
- **CMU 15-440** — Distributed Systems. Slides available online. Clear visuals on vector clocks.

---

# Summary — Concepts at a Glance

| Concept | What It Is | Guarantee | Used In |
|---|---|---|---|
| Quartz oscillator | Piezoelectric crystal at resonant frequency | ~32,768 Hz; drifts 15 ms–1 s/day | All computers |
| Cesium atomic clock | Electron hyperfine transitions at 9.19 GHz | Defines the SI second; <1 ns/day | UTC, GPS, time servers |
| NTP Wall Clock | OS clock disciplined via network sync | ±1–50 ms; can jump backward | Logging, display |
| Monotonic Clock | Counter that only increases | Measures elapsed time; never backward | Timeouts, benchmarks |
| Lamport Clock | Logical integer counter per process | `a→b ⟹ C(a)<C(b)`; total order | Paxos, debugging |
| Vector Clock | Integer vector per process | `a→b ⟺ V(a)<V(b)`; detects concurrency | Dynamo, Riak, CRDTs |
| TrueTime (Spanner) | GPS + atomic, bounded uncertainty interval | External consistency / linearizability | Google Spanner |
| Hybrid Logical Clock | (physical_time, logical_counter) pair | Close to wall clock + Lamport properties | CockroachDB, YugabyteDB |

---

# References

[1] Newton, I. *Philosophiæ Naturalis Principia Mathematica.* Royal Society, London, 1687.

[2] Einstein, A. "Zur Elektrodynamik bewegter Körper." *Annalen der Physik,* vol. 17, pp. 891–921, 1905. Available: [fourmilab.ch/etexts/einstein/specrel/www/](https://www.fourmilab.ch/etexts/einstein/specrel/www/)

[3] Ashby, N. "Relativity in the Global Positioning System." *Living Reviews in Relativity,* vol. 6, no. 1, 2003. doi:10.12942/lrr-2003-1

[4] Curie, P. & Curie, J. "Développement par compression de l'électricité polaire dans les cristaux hémièdres à faces inclinées." *Bulletin de la Société Minéralogique de France,* vol. 3, pp. 90–93, 1880.

[5] Kleppmann, M. *Designing Data-Intensive Applications.* O'Reilly Media, 2017. ISBN 978-1-4493-7332-0. Available: [dataintensive.net](https://dataintensive.net)

[6] Bureau International des Poids et Mesures (BIPM). *The International System of Units (SI), 9th Edition.* BIPM, 2019. Available: [bipm.org/en/publications/si-brochure](https://www.bipm.org/en/publications/si-brochure)

[7] Corbett, J. C. et al. "Spanner: Google's Globally Distributed Database." *Proceedings of OSDI '12,* pp. 251–264, 2012. Available: [usenix.org/system/files/conference/osdi12/osdi12-final-16.pdf](https://www.usenix.org/system/files/conference/osdi12/osdi12-final-16.pdf)

[8] Mills, D. L. *Network Time Protocol (Version 4): Protocol and Algorithms Specification.* RFC 5905, IETF, June 2010. Available: [rfc-editor.org/rfc/rfc5905](https://www.rfc-editor.org/rfc/rfc5905)

[9] Lamport, L. "Time, Clocks, and the Ordering of Events in a Distributed System." *Communications of the ACM,* vol. 21, no. 7, pp. 558–565, July 1978. Available: [lamport.azurewebsites.net/pubs/time-clocks.pdf](https://lamport.azurewebsites.net/pubs/time-clocks.pdf)

[10] Fidge, C. J. "Timestamps in Message-Passing Systems That Preserve the Partial Ordering." *Proceedings of the 11th Australian Computer Science Conference,* vol. 10, no. 1, pp. 56–66, 1988.

[11] Mattern, F. "Virtual Time and Global States of Distributed Systems." In Cosnard, M. et al. (eds.), *Proceedings of the Workshop on Parallel and Distributed Algorithms,* pp. 215–226, Elsevier, 1989.

[12] DeCandia, G. et al. "Dynamo: Amazon's Highly Available Key-value Store." *Proceedings of SOSP '07,* pp. 205–220, 2007. Available: [dl.acm.org/doi/10.1145/1294261.1294281](https://dl.acm.org/doi/10.1145/1294261.1294281)

[13] Kulkarni, S. et al. "Logical Physical Clocks and Consistent Snapshots in Globally Distributed Databases." *Proceedings of OPODIS 2014.* arXiv:1409.7349. Available: [arxiv.org/abs/1409.7349](https://arxiv.org/abs/1409.7349)

[14] Essen, L. & Parry, J. V. L. "An Atomic Standard of Frequency and Time Interval: A Caesium Resonator." *Nature,* vol. 176, pp. 280–282, 1955. doi:10.1038/176280a0

[15] Vig, J. R. *Quartz Crystal Resonators and Oscillators for Frequency Control and Timing: A Tutorial.* U.S. Army CECOM, SLCET-TR-88-1 (Rev. 8.5.1), 2014. Available: [ieee-uffc.org/download/vig-tutorial/](https://ieee-uffc.org/download/vig-tutorial/)

[16] Marrison, W. A. "The Evolution of the Quartz Crystal Clock." *Bell System Technical Journal,* vol. 27, no. 3, pp. 510–588, 1948.

[17] Tanenbaum, A. S. & Van Steen, M. *Distributed Systems: Principles and Paradigms,* 3rd ed. Pearson, 2017. Available: [distributed-systems.net](https://www.distributed-systems.net/index.php/books/ds4/)

[18] Ludlow, A. D. et al. "Optical Atomic Clocks." *Reviews of Modern Physics,* vol. 87, pp. 637–701, 2015. doi:10.1103/RevModPhys.87.637