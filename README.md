# 5G NR Timer Wheel in User-Plane and MAC Schedulers

A timer wheel can be designed as a **circular array of time buckets**, where each bucket contains multiple typed events. Advancing time selects a bucket; expiring an event changes protocol state or makes work eligible. The wheel does **not** itself select a UE, assign PRBs, or guarantee a radio transmission.

This is an engineering investigation and proposed design, not an Ericsson source-code disclosure. It separates standardized behavior, publicly documented Ericsson capabilities, inspected open-source implementations, and design recommendations.

## 1. Evidence and scope

| Evidence category | What can be established | What cannot be inferred |
|---|---|---|
| 3GPP/ETSI specifications [S1]-[S7] | Protocol timers, timing relationships, QoS parameters, and procedure ownership | A vendor's internal data structure, thread model, or timer API |
| Ericsson public material [E1]-[E3] | Priority scheduling for time-critical traffic, latency/reliability objectives, and flexible edge user-plane deployment | That Ericsson implements these features with a particular timer wheel, bucket size, event enumeration, or allocation policy |
| Inspected srsRAN source [I1]-[I2] | An actual NR scheduler HARQ timeout wheel and a separate circular resource-allocation calendar | Ericsson implementation details |
| FD.io VPP, Linux, and DPDK [I3]-[I5] | Concrete networking timer designs and their tradeoffs | NR protocol compliance or Ericsson product behavior |
| Design in this README | A possible implementation with explicit assumptions and failure handling | A production-ready implementation or measured performance guarantee |

**Ericsson limitation:** no publicly verifiable Ericsson gNB/UPC timer-wheel implementation was identified in the sources reviewed. The cited Ericsson articles describe capabilities, not scheduler source code. Their goals motivate this design; they do not prove its internals. A patent, if added later, must be identified as a proposed invention rather than evidence of deployed code.

**Research provenance:** public web research was conducted on 2026-11-11. Ericsson and ETSI references were located through indexed web results; direct page/PDF retrieval was unavailable in this environment. The implementation references were read directly through GitHub. Protocol descriptions below are summaries, not substitutes for the cited specifications. Release 18 references are a baseline, not a claim that they are the latest releases.

### What does "user-plane scheduler" mean here?

There are different scheduling domains:

```text
Data network -- N6 -- UPF -- N3 -- gNB CU-UP -- F1-U -- gNB DU -- radio -- UE
                     |             |                   |
                 forwarding,     SDAP/PDCP          RLC/MAC/PHY
                 shaping, QoS    processing         radio scheduling
```

This is a simplified split-gNB architecture; integrated deployments need not have the same process boundaries.

- A **UPF packet scheduler** may pace or shape traffic and enforce policy. It does not own NR HARQ, RLC, or PDCP timers.
- A **RAN user-plane scheduler/controller** may schedule bearer processing, queue service, and packet deadlines. PDCP and RLC timer ownership follows the hosting entity, not the name of the scheduler.
- A **DU MAC scheduler** selects radio opportunities using UE state, HARQ, channel information, available control/data resources, QoS, and configured timing.

"UPC" is not used here as a standardized 3GPP network-function name or a verified Ericsson component boundary.

## 2. Timing foundations and realistic complexity

For the normal cyclic-prefix slot structure, the slot duration is `1 ms / 2^mu`, with subcarrier spacing `15 * 2^mu kHz` [S1].

| Numerology `mu` | SCS | Slot duration |
|---|---|---|
| 0 | 15 kHz | 1 ms |
| 1 | 30 kHz | 500 us |
| 2 | 60 kHz | 250 us |
| 3 | 120 kHz | 125 us |

These are examples, not the full set of NR numerologies. Shorter-than-slot allocations exist. A slot-granularity wheel alone cannot express every symbol-level deadline; use exact radio-time conversion, a finer calendar, or a separate high-resolution mechanism where required. Do not treat all OFDM symbols as equal-duration wall-clock intervals.

Important corrections to common assumptions:

- Wheel insertion and direct unlinking can be **O(1)** in a fixed-horizon design. Processing a bucket containing `k` timers is **O(k)**, plus callback work. A burst can still miss a slot deadline.
- Heap-based **O(log N)** timers are not inherently unusable in NR. Timer count, cache layout, churn, precision, and worst-case latency determine suitability. DPDK's documented `rte_timer` implementation uses a **per-lcore skiplist**, not a wheel [I5].
- Preallocation and avoiding hot-path allocation are useful real-time design choices, not requirements imposed by 3GPP.
- A wheel reduces timer-management work; it does not eliminate OS preemption, synchronization, PHY-interface deadlines, or execution-time variability.

## 3. Circular buffer versus timer wheel

A packet FIFO ring stores entries in arrival order. A timer wheel is instead an array of **bucket heads**, indexed by expiry time. Several events can share a bucket, and events can be cancelled out of arrival order. It must not overwrite live events when the cursor wraps.

Define:

```text
epoch        = fixed origin of the chosen time domain
tick_period  = Delta
W            = number of buckets
cursor       = last fully processed absolute tick
due_tick     = ceil((absolute_deadline - epoch) / Delta)
bucket_index = due_tick % W
horizon      = W * Delta
```

Use integer time arithmetic with checked range/overflow handling. A power-of-two `W` permits `due_tick & (W - 1)`, but is not essential.

```text
                 circular array of bucket heads
           +-----+-----+-----+-----+-----+-----+
index      |  0  |  1  |  2  | ... | W-2 | W-1 |
           +--|--+-----+--|--+-----+-----+-----+
              |         |
              v         v
          event A    event B --> event C
                     HARQ       DRX boundary

Each event carries its full deadline and typed identity.
The bucket number alone is not a timestamp.
```

An intrusive doubly linked list, or an index-based equivalent in a preallocated pool, supports constant-time removal by handle. One list node belongs to only one bucket or ready queue at a time. A UE with several independent timers needs separate nodes.

### A bounded single-wheel example

Assume `Delta = 125 us`, `W = 64`, and `cursor = 100`. The horizon is 8 ms.

| Absolute deadline tick | Bucket | Action |
|---|---|---|
| 106 | 42 | Insert; process when the cursor reaches 106 |
| 107 | 43 | Insert; may contain several different event types |
| 164 | 36 | Valid: exactly one revolution ahead; bucket 36 was already drained at tick 100 |
| 170 | 42 | Outside this wheel's horizon; use the explicit overflow policy |
| 100 or earlier | Not inserted | Already due; put on the current ready path |

Ticks 106 and 170 hash to the same bucket, but must not expire together.

For a bounded wheel, accept only `cursor < due_tick <= cursor + W`. Reject longer delays with a visible error, or place them in a separately designed overflow structure. Merely storing an absolute deadline does not make an arbitrary-delay timer safe in a wheel that dispatches every entry on its next bucket visit.

### Longer timers: choose a deliberate extension

| Approach | How it works | Tradeoff |
|---|---|---|
| Bounded wheel | Covers one revolution; longer delays are rejected or routed elsewhere | Simple, predictable membership operations; limited range |
| Hashed wheel with rounds | Keeps longer timers in a bucket with remaining revolutions; decrements on visits | Repeated inspection of long-lived timers increases maintenance cost |
| Hierarchical wheel | Uses coarse outer levels, then promotes timers into finer levels as expiry approaches | Broad range; promotion bursts and exact-boundary rules need care |
| Fine wheel plus coarse timer service | Keeps radio-sensitive work local and sends long maintenance timers to another service | Requires explicit ownership and clock conversion at the boundary |

For a rounds-based variant, with a just-processed cursor and positive integer delay `d`, one common convention is `rounds = floor((d - 1) / W)`: decrement a positive count on bucket visits and expire when it is already zero. Test this convention at `W`, `W + 1`, and `2W`; mixing conventions causes a full-revolution error.

Hierarchical wheels are not all identical. Modern Linux's ordinary timer wheel deliberately uses coarser levels without classic recascading; its timeout slack is not automatically suitable for radio deadlines [I4]. Varghese and Lauck provide the foundational comparison of wheel variants [T1].

## 4. Typed event model and lifecycle

Keep timing infrastructure generic; put protocol behavior in typed handlers. An illustrative record contains:

| Field | Purpose |
|---|---|
| `event_type` | Selects a validated handler; unknown types are reported as errors |
| `absolute_deadline`, `due_tick` | Preserve timing intent and detect lateness or bucket aliasing |
| `owner_shard`, `cell_id` | Identify the thread and cell that may mutate the event |
| `context_id`, `context_epoch` | Identify a UE, bearer, flow, or procedure without trusting a recycled identifier |
| `timer_id`, `arm_generation` | Distinguish timer instances and invalidate obsolete cancellation/dispatch tokens |
| Typed payload | For example, HARQ process ID and direction, bearer ID, or flow ID; avoid an untyped pointer contract |
| Intrusive links and lifecycle state | Track membership in a bucket, detached batch, ready queue, or execution |

Use a tagged union/variant for payloads and validate the event/payload combination. Do not store packet payloads in wheel buckets.

### Proposed API contracts

```text
arm(type, context, payload, deadline) -> handle or explicit error
cancel(handle)                     -> cancelled / stale / already executing
rearm(handle, new_deadline)         -> replacement handle or explicit error
advance_to(absolute_time)           -> expiry work plus lateness/overrun status
```

Suggested lifecycle:

```text
FREE -> ARMED -> READY -> EXECUTING -> FREE
          |        |
          +--------+---- cancellation invalidates this arm

Rearm: invalidate/unlink the old arm, then create a new generation.
```

Specify whether failed rearm leaves the previous timer armed. A useful contract is **validate capacity, range, and payload before replacing the old arm**, so failure preserves it.

Cancellation must account for detached expiry batches and ready queues, not just linked bucket nodes. A ready token should capture the arm generation and context epoch; compare them again immediately before dispatch. Invalidated tokens must not execute. Generation checks do not replace safe memory lifetime: do not reclaim/reuse the backing record while an execution or queued reference can still access it. Prevent generation wrap from making a stale handle valid.

### Single-owner processing algorithm

The following is conceptual pseudocode for the bounded wheel, not a complete implementation:

```text
arm(event, deadline):
    validate owner, context, event type, payload, capacity, and time range
    compute due_tick using upward rounding
    if due_tick > cursor + W:
        return OUT_OF_HORIZON and report the failure
    acquire a record; assign a new generation and immutable arm identity
    if due_tick <= cursor:
        enqueue a ready token for this arm
    else:
        link the record into bucket[due_tick % W]
    return handle

advance_to(now):
    target = floor((now - epoch) / Delta)
    reject backward time or an invalid clock/radio-time mapping
    for each tick from cursor + 1 through target:
        detach that tick's complete bucket
        advance cursor to that tick
        for each detached record:
            verify ownership and full due_tick
            report an invariant violation if due_tick is not this tick
            enqueue a ready token for the arm
    dispatch ready tokens in the scheduler's defined phase order
    before each dispatch, validate generation, context epoch, and state
    record deadline lateness; transition out of READY before invoking handler
```

The pseudocode assumes bucket extraction is completed before handlers run and that detached membership remains cancellable or invalidatable. Dispatch order across expired ticks must preserve required per-context ordering.

Do not recursively invoke a handler that arms an already-due timer. Queue it and apply a bounded same-tick dispatch policy; otherwise one callback can monopolize the scheduler. Make overflow, pool exhaustion, invalid input, and dispatch overrun observable rather than silently dropping work.

## 5. User-plane scheduling perspective

The principal question is **when a flow or packet becomes eligible, urgent, or no longer useful**. The ready-queue policy still decides which eligible flow receives service.

| Candidate event type | Owner and trigger | Expiry action and constraints |
|---|---|---|
| `FLOW_PACING_ELIGIBLE` | Packet scheduler; a flow's next permitted send time | Move the flow to an eligible queue; transmission still depends on resources |
| `SHAPER_RECHECK` | Shaping scheduler; insufficient tokens/credit | Recompute credit from elapsed time, then reconsider eligibility; policing need not use timers |
| `HOL_URGENCY_CHECK` | RAN bearer/flow scheduler; head-of-line packet approaches a local deadline | Update urgency or scheduler weight; not an automatic standardized discard |
| `LOCAL_PACKET_DEADLINE` | Queue owner; explicitly configured packet-lifetime policy | Apply that policy and notify/account for expiry; do not confuse this with PDCP `discardTimer` |
| `PDCP_DISCARD` | Transmitting PDCP entity; finite configured `discardTimer` | Perform PDCP SDU discard and applicable lower-layer notification per TS 38.323 [S5] |
| `PDCP_REORDERING_EXPIRE` | Receiving PDCP entity; configured `t-Reordering` | Execute the PDCP receive-state procedure, not generic FIFO flushing [S5] |
| `RLC_POLL_RETRANSMIT` | Transmitting AM RLC entity; `t-PollRetransmit` | Execute polling/retransmission procedure using current RLC state [S4] |
| `RLC_REASSEMBLY_EXPIRE` | Receiving UM/AM RLC entity; `t-Reassembly` | Run mode-specific receive/reassembly actions; AM and UM are not interchangeable [S4] |
| `RLC_STATUS_ALLOWED` | Receiving AM RLC entity; `t-StatusProhibit` | End status prohibition; report only when the protocol's other conditions permit [S4] |
| `MAINTENANCE_SAMPLE` | Implementation-specific accounting service | Update coarse statistics outside the radio-critical dispatch budget |

The first four and last rows are design examples, not standardized timer names. PDCP/RLC timers belong to their entities even when a shared service implements their storage.

**Core SR & Uplink Timer Wheel Events**

In a telecom baseband scheduler (such as Ericsson's LTE/5G MAC scheduler), the timer wheel manages the time-bounded states between receiving a PUCCH SR indication and receiving the subsequent PUSCH data or Buffer Status Report (BSR):
* _SR Prohibit / Re-trigger Window (sr-ProhibitTimer)_: Tracks the cooldown period after a UE transmits an SR before another SR opportunity is evaluated or expected.
* _UL Grant Response / PUSCH Reception Timeout_: Tracks the expected arrival slot (k2 offset) of the PUSCH transmission after the scheduler issues a DCI Format 0_0 / 0_1 UL grant in response to an SR. If no PUSCH or BSR arrives by that tick, the wheel triggers a grant retry or marks the SR attempt as failed.
* _Pending SR Aging / Expiration_: Tracks how long an unserved SR has remained in the scheduling queue. If a UE cannot be scheduled immediately due to PDCCH CCE or PRB congestion, the wheel triggers a priority boost or drops the stale SR state after a maximum latency threshold.
* _Periodic BSR & Retransmission BSR Timers (periodicBSR-Timer, retxBSR-Timer)_: Tracks when a UE is expected to send updated buffer reports once an initial SR-triggered grant has been fulfilled, ensuring the scheduler does not lose track of remaining uplink buffer data.
* _UE Inactivity & DRX / SR Masking Timers (tInactivityTimer, drx-InactivityTimer)_: Tracks transitions between active scheduling, DRX sleep cycles (where SR opportunities are masked or restricted), and RRC idle release when no further SRs or data arrive.
* _HARQ Round-Trip & Re-scheduling Timers_: Tracks the slot offset for UL HARQ feedback and synchronous/asynchronous retransmission grants if the initial SR-triggered PUSCH fails CRC decoding.

### Packet delay budget is not a local discard timeout

The 5G QoS Packet Delay Budget concerns the path between the UE and the N6-terminating UPF, not just residence time in a MAC queue [S6, clause 5.7.3.4]. Apply the applicable access-network/core-network budget rules and QoS configuration.

A possible **local planning estimate**, not a 3GPP formula, is:

```text
remaining_local_budget =
    assigned_access_budget
    - measured_or_estimated_elapsed_access_delay
    - reserved_remaining_processing_and_radio_time
```

Timestamp provenance matters: a packet's arrival at a local queue is not necessarily the start of the PDB measurement. Different hosts' clocks cannot be subtracted without a valid synchronization model. An exhausted estimate should trigger a documented policy, not an unannounced "success" or unconditional packet drop.

Useful scale reductions:

- Keep a timer for a flow's **earliest relevant deadline** instead of every packet where semantics permit. Re-evaluate on head change and completion.
- A FIFO head represents the earliest deadline only when deadlines are monotonic in that queue. Otherwise maintain an earliest-deadline structure or separate queues.
- Periodic token refill need not generate a timer every tick: compute accrued credit lazily from elapsed time, and arm a wakeup only for blocked work.
- Coalesce wakeups only within permitted slack. Independent PDCP SDU discard obligations must remain correctly represented.

## 6. MAC scheduling perspective

The principal question is **which protocol transitions and future radio opportunities must be considered at a particular slot or symbol boundary**.

| Candidate event type | Ownership and semantics | Scheduler reaction |
|---|---|---|
| `HARQ_FEEDBACK_WATCHDOG` | gNB implementation-specific timeout for missing DL HARQ-ACK or UL decoding/CRC indication | Resolve current HARQ state under a documented policy; timeout is not proof of NACK |
| `HARQ_RETX_AGE_LIMIT` | Optional gNB policy for a pending retransmission that remains unscheduled | Report starvation and apply policy; not a universal 3GPP HARQ timer |
| `DRX_BOUNDARY_UPDATE` | gNB prediction/mirror of UE DRX state [S2, clause 5.7] | Recompute expected monitoring eligibility using all Active Time conditions |
| `RA_PROCEDURE_DEADLINE` | gNB random-access procedure context [S2, clause 5.1] | Expire local procedure state or schedule needed work within the applicable window |
| `UL_ALIGNMENT_GUARD` | gNB tracking of UE uplink-alignment validity [S2, clause 5.2] | Re-evaluate UL eligibility and timing-advance action; obey procedure-specific exceptions |
| `PERIODIC_RESOURCE_OCCASION` | Scheduler calendar for configured CSI/SRS/SR or other periodic resources | Account for configured occasions and collisions; the timer is not a resource reservation |
| `CONFIG_ACTIVATION_BOUNDARY` | A configuration procedure with an explicitly defined effective radio time | Apply the correct configuration version; do not invent activation timing |

**UE timers are not automatically gNB timers.** `periodicBSR-Timer`, `retxBSR-Timer`, and `sr-ProhibitTimer` execute in the UE MAC procedures [S2, clauses 5.4.4-5.4.5]. A gNB may track expected behavior, but it does not run the UE's BSR state machine on the UE's behalf. Similarly, `timeAlignmentTimer` and DRX timers have standardized UE-side behavior; gNB tracking is a separate implementation concern.

DRX includes `drx-onDurationTimer`, `drx-InactivityTimer`, and per-process `drx-HARQ-RTT-TimerDL/UL` and `drx-RetransmissionTimerDL/UL`. An RTT-timer interval does **not by itself** require the UE to stay awake; retransmission timers contribute to Active Time. Expiry of one timer does not necessarily mean sleep because other Active Time conditions may remain. Start/stop rules, symbols/slots/sub-millisecond units, and RRC configuration must be mapped exactly [S2, S7].

### A radio calendar is related to, but different from, a timeout wheel

For a same-numerology illustration:

```text
PDCCH at slot n ---- K0 ----> PDSCH at slot n + K0
PDSCH at slot m ---- K1 ----> HARQ-ACK opportunity at slot m + K1
PDCCH at slot n ---- K2 ----> PUSCH at slot n + K2
```

K0/K2 allocation rules are in TS 38.214; HARQ-ACK timing rules are in TS 38.213 [S3]. Mixed numerologies, processing constraints, TDD symbol direction, and configured tables require the exact specification mappings; the drawing is not a universal conversion formula.

Maintain two distinct structures where useful:

| Structure | Contents | Safety requirement |
|---|---|---|
| Future-slot resource calendar | PRB/symbol occupancy, PUCCH/PUSCH/PDSCH allocations, scheduling results | Validate the absolute slot tag and horizon before reusing a ring entry |
| Timer wheel | State transitions, feedback watchdogs, deadline/eligibility events | Validate the deadline and event lifetime; never infer a grant from expiry |

Known K0/K1/K2 resources must be reserved when the scheduling decision is committed, not first requested by a callback at the target slot. The scheduler/PHY preparation deadline can precede the over-the-air slot; trigger processing early enough for that pipeline.

### Suggested slot-processing integration

This ordering is a design choice, not a prescribed Ericsson sequence:

```text
slot indication and timestamped PHY/control messages
    -> ingest messages and apply context/configuration updates
    -> resolve feedback and invalidate superseded watchdogs
    -> advance the appropriate time domain and collect due events
    -> apply state transitions and refresh eligibility
    -> schedule retransmissions/new data under radio and QoS constraints
    -> commit allocations and deliver results before the PHY deadline
```

Define equal-time precedence explicitly. For example, a valid ACK received within the permitted interface deadline should supersede its watchdog; late feedback after HARQ-process reuse must not complete a new transport block. Correlate feedback with process/transport-block identity and transmission occasion, not just a recycled UE identifier.

## 7. Public implementation evidence and Ericsson relevance

### srsRAN: an explicit NR HARQ timeout wheel

At inspected revision `4bf1543936d062686d64c10724d2f27a9854f065` [I1]:

- `cell_harq_repository` stores `harq_timeout_wheel` as a vector of intrusive doubly linked HARQ lists.
- `alloc_harq()` computes `slot_timeout` and inserts into `harq_timeout_wheel[slot_timeout.to_uint() % harq_timeout_wheel.size()]`.
- `slot_indication()` selects the current slot's bucket and handles its entries.
- `dealloc_harq()` unlinks the HARQ from its timeout bucket. Retransmission handling also updates bucket membership.
- `handle_harq_ack_timeout()` handles state-dependent timeout outcomes and notifies after deallocation to avoid reentrancy.

This directly demonstrates the circular-array-plus-event-list pattern in an NR scheduler. Its timeout policy and sizing assumptions are implementation-specific; copying the modulo expression without those assumptions is unsafe.

The same project separately implements a `cell_resource_allocator` with a circular pool of per-slot resource grids [I2]. That is evidence for the distinction between a resource calendar and an event timeout wheel.

### Other networking implementations

FD.io VPP's timer template describes configurable wheel geometries, multiple timer IDs per object, start/stop handles, and batched expired-handle callbacks [I3]. These are useful patterns for typed scheduler events, but not a ready-made NR scheduler or proof of zero-allocation behavior.

Linux supplies a concrete multi-level timeout wheel with deliberate granularity/slack decisions [I4]. DPDK supplies a contrasting ordered skiplist timer service whose precision depends on how frequently the application calls `rte_timer_manage()` [I5]. Neither should be relabeled as an Ericsson implementation.

### What the Ericsson publications contribute

| Public source | Supported takeaway | Design implication proposed here |
|---|---|---|
| Advanced connectivity solutions for 5G time-critical communication [E1] | Priority scheduling supports differentiated time-critical traffic | Keep expiry/eligibility separate from the policy that prioritizes ready traffic |
| Overview: time-critical communication with 5G NR [E2] | Low latency and reliability must be considered together | Measure deadline misses and late delivery, not only average timer latency |
| Edge use cases need a 5G and beyond user plane [E3] | User-plane deployment must fit different edge use cases | Preserve UPF/CU-UP/DU ownership and budget boundaries rather than assuming one global wheel |

The right claim is **a design aligned with publicly described scheduling goals**, not "Ericsson uses this implementation." No internal Ericsson APIs, event names, wheel dimensions, or proprietary algorithms are asserted here.

## 8. Production engineering constraints

### Clocks, numerology, and discontinuities

Use an extended absolute radio-tick counter rather than raw `(SFN, slot)` as timer identity. NR's 10-bit SFN repeats every 10.24 seconds; modulo bucket indices cannot distinguish epochs. Define initialization, synchronization loss, cell restart, and stale-message handling.

For mixed numerologies, choose per-cell/numerology wheels or a common sufficiently fine time base with explicit conversion. A BWP/numerology change must not reinterpret a pending "10 slots" using the new slot duration. Preserve its intended absolute deadline or explicitly recompute it according to procedure rules.

Use monotonic time for elapsed-time packet timers. Use a defined mapping to radio time for radio events. Wall-clock/NTP adjustments must not silently move protocol deadlines.

If processing jumps from tick `t` to `t + n`, visiting only the last bucket loses expiries. Process crossed buckets, or use a correctly designed skip/fast-forward mechanism. Large gaps need an explicit recovery policy: expired state transitions may require chronological handling, but scheduling a radio transmission in the past is impossible and must be reported.

### Ownership, cancellation, and context lifetime

Prefer one mutating owner per cell/shard, with bounded command queues for other cores. Cross-core `arm`/`cancel` requests need acceptance/ordering semantics; enqueueing a cancellation is not proof it has already taken effect.

On UE release, bearer removal, handover, or cell stop:

1. Stop or reject new commands for that context epoch.
2. Invalidate timer arms and unlink their records.
3. Invalidate/drain ready tokens and account for executing handlers.
4. Reclaim context storage only after outstanding references are safe.

Use context epochs across migration and identifier reuse. A raw `rnti` or pointer is not a sufficient lifetime check. Single-owner execution simplifies races but does not remove queued stale work.

### Capacity, workload, and overload

For a fixed wheel, approximate memory as:

```text
memory = W * bucket_head_size
       + max_simultaneous_timer_records * record_size
       + ready_queues + command_queues + context_indexes
```

Illustration only: 5,000 UEs, 16 tracked processes per direction, and two directions could require 160,000 simultaneous HARQ records. At 64 bytes each, that is 10.24 MB before all other timers and queues. Actual process counts and record layouts depend on configuration.

For a scheduler tick, the relevant execution budget is approximately:

```text
time_domain_advance
+ bucket_extraction
+ due_event_state_updates
+ scheduling_policy_and_resource_allocation
+ PHY_submission
< available_pipeline_budget
```

The available budget is not automatically a whole slot. Include queueing and preemption in the measured bound.

Size for correlated expiry bursts: synchronized DRX boundaries, many UEs losing feedback, and mass context teardown are not uniformly distributed traffic. A wheel never guarantees constant total tick cost.

Reserve capacity for critical events, measure pool/queue high-water marks, and define admission or recovery when capacity is exhausted. Separate maintenance callbacks from radio-critical state transitions. Do not silently defer a hard radio deadline to satisfy a callback cap. Jitter soft maintenance tasks only when their semantics allow it; never randomize standardized radio occasions.

For periodic tasks, normally compute the next expiry from the prior **scheduled** expiry rather than callback completion to avoid drift. Define whether missed occurrences are coalesced, counted, or replayed; replaying obsolete radio allocations is not valid.

### Observability and focused verification

Expose active timers by type, bucket occupancy, expiry batch size, insert/cancel/rearm counts, stale-token rejection, pool/queue exhaustion, skipped ticks, callback duration, and dispatch lateness (`dispatch_time - intended_deadline`). Track maxima and tail percentiles, not just averages.

Before deploying an implementation, verify:

| Scenario | Required property |
|---|---|
| Deadline between ticks | No early expiry; upward rounding adds less than one tick before dispatch delay |
| Exact boundary, current tick, already-past deadline | A documented immediate-ready policy; no accidental full-wheel delay |
| `W - 1`, `W`, `W + 1`, and multiple revolutions | Correct horizon rejection/overflow/rounds behavior |
| Many event types in one bucket | No lost entries; deterministic required ordering; observable overload |
| Cancel/rearm before extraction and after READY | At most one execution of a live arm; obsolete generations do not execute |
| Handler rearms itself or another due event | No unbounded recursion or unobserved same-tick starvation |
| ACK and watchdog at the same boundary | Timestamp/phase rules select the intended outcome |
| UE/HARQ reuse, bearer release, migration | No stale handler accesses a new or freed context |
| Skipped ticks, SFN wrap, restart, numerology change | No lost expiry or timestamp reinterpretation |
| Long-running periodic operation | No unintended phase drift; missed-occurrence policy is respected |
| Pool/command/ready queue exhaustion | Visible failure and defined recovery, not silent event loss |

## 9. References

### Standards

- **[S1]** [3GPP TS 38.211 / ETSI TS 138 211 V18.5.0](https://www.etsi.org/deliver/etsi_ts/138200_138299/138211/18.05.00_60/ts_138211v180500p.pdf): physical channels and modulation; clause 4 covers numerology and frame/slot structure.
- **[S2]** [3GPP TS 38.321 / ETSI TS 138 321 V18.5.0](https://www.etsi.org/deliver/etsi_ts/138300_138399/138321/18.05.00_60/ts_138321v180500p.pdf): MAC; clauses 5.1, 5.2, 5.4.4, 5.4.5, and 5.7 cover random access, UL alignment, SR, BSR, and DRX.
- **[S3]** [TS 38.213 official version directory](https://www.etsi.org/deliver/etsi_ts/138200_138299/138213/) and [TS 38.214 V18.5.0](https://www.etsi.org/deliver/etsi_ts/138200_138299/138214/18.05.00_60/ts_138214v180500p.pdf): control procedures/HARQ-ACK timing and data-channel allocation/processing timing. Select matching releases.
- **[S4]** [3GPP TS 38.322 / ETSI TS 138 322 V18.2.0](https://www.etsi.org/deliver/etsi_ts/138300_138399/138322/18.02.00_60/ts_138322v180200p.pdf): RLC; data-transfer procedures and protocol timers.
- **[S5]** [TS 38.323 official version directory](https://www.etsi.org/deliver/etsi_ts/138300_138399/138323/): PDCP; SDU discard, receive/reordering procedures, and timer definitions. Select the deployed release.
- **[S6]** [3GPP TS 23.501 / ETSI TS 123 501 V18.10.0](https://www.etsi.org/deliver/etsi_ts/123500_123599/123501/18.10.00_60/ts_123501v181000p.pdf): 5GS architecture and QoS; particularly clause 5.7.3.4 on PDB.
- **[S7]** [TS 38.331 official version directory](https://www.etsi.org/deliver/etsi_ts/138300_138399/138331/): RRC configuration, including DRX and protocol timer values/units. Match the MAC/RLC/PDCP release and supported features.

### Ericsson public material

- **[E1]** [Advanced connectivity solutions for 5G time-critical communication](https://www.ericsson.com/en/reports-and-papers/ericsson-technology-review/articles/advanced-connectivity-solutions-for-5g-time-critical-communication), Ericsson Technology Review.
- **[E2]** [Overview: time-critical communication with 5G NR](https://www.ericsson.com/en/blog/2021/2/time-critical-communication--5g-nr), Ericsson.
- **[E3]** [Edge use cases need a 5G and beyond user plane](https://www.ericsson.com/en/blog/2022/6/edge-use-cases-need-a-5g-and-beyond-user-plane), Ericsson.

### Inspected public implementation references

- **[I1]** srsRAN Project, revision `4bf1543936d062686d64c10724d2f27a9854f065`: [`cell_harq_manager.h`](https://github.com/srsran/srsRAN_Project/blob/4bf1543936d062686d64c10724d2f27a9854f065/lib/scheduler/cell/cell_harq_manager.h) and [`cell_harq_manager.cpp`](https://github.com/srsran/srsRAN_Project/blob/4bf1543936d062686d64c10724d2f27a9854f065/lib/scheduler/cell/cell_harq_manager.cpp). See `harq_timeout_wheel`, `alloc_harq`, `slot_indication`, `dealloc_harq`, and `handle_harq_ack_timeout`.
- **[I2]** srsRAN Project, same revision: [`resource_grid.h`](https://github.com/srsran/srsRAN_Project/blob/4bf1543936d062686d64c10724d2f27a9854f065/lib/scheduler/cell/resource_grid.h), particularly `cell_resource_allocator`.
- **[I3]** FD.io VPP, revision `b2514c7f1c52fcdaafcda10336658c9ce2b50457`: [`tw_timer_template.h`](https://github.com/FDio/vpp/blob/b2514c7f1c52fcdaafcda10336658c9ce2b50457/src/vppinfra/tw_timer_template.h), wheel geometry, per-object timer IDs, handles, and expiry callback API.
- **[I4]** Linux, revision `067c5c24cef23d913b67eaaf71ae67e7169a6620`: [`kernel/time/timer.c`](https://github.com/torvalds/linux/blob/067c5c24cef23d913b67eaaf71ae67e7169a6620/kernel/time/timer.c), opening design commentary on levels, granularity, and non-recascading behavior.
- **[I5]** DPDK, revision `68209bbd21e4724056350b3ee5d3c29fd6c48584`: [`timer_lib.rst`](https://github.com/DPDK/dpdk/blob/68209bbd21e4724056350b3ee5d3c29fd6c48584/doc/guides/prog_guide/timer_lib.rst), documented per-lcore skiplist implementation and timer-management polling contract.

### Foundational algorithm

- **[T1]** George Varghese and Tony Lauck, *Hashed and Hierarchical Timing Wheels: Efficient Data Structures for Implementing a Timer Facility*, IEEE/ACM Transactions on Networking, 1997. [DOI: 10.1109/90.650142](https://doi.org/10.1109/90.650142); [university-hosted paper](https://www.cs.columbia.edu/~nahum/w6998/papers/ton97-timing-wheels.pdf). Foundational theory, not evidence about a specific vendor.
