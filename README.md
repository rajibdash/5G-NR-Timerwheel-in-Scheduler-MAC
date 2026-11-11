# 5G NR Timerwheel in Scheduler/MAC
Investigation and own understanding to evolve with new APIs. for more efficient and revealing correct information by events.

## Starting point
Designing a timer wheel for a 5G NR User Plane Controller (UPC) or Layer 2 (L2) Scheduler (MAC/RLC) requires deterministic O(1) complexity, 
Zero dynamic memory allocation, and sub-millisecond timing precision (good learning about low latency objectives). In 5G NR, slot durations 
shrink down to 500 µs (30 kHz SCS) or 125 µs (120 kHz SCS), making heap-based \(O(\log N)\) timers unusable due to cache misses and scheduling 
jitter.
