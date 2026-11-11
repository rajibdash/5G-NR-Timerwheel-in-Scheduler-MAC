# 5G-NR-Timerwheel-in-Scheduler-MAC
Investigation and understanding to evolve with new APIs

## Starting point
Designing a timer wheel for a 5G NR User Plane Controller (UPC) or Layer 2 (L2) Scheduler (MAC/RLC) requires deterministic O(1) complexity, 
zero dynamic memory allocation, and sub-millisecond timing precision. In 5G NR, slot durations shrink down to 500 µs (30 kHz SCS) or 125 µs 
(120 kHz SCS), making heap-based \(O(\log N)\) timers unusable due to cache misses and scheduling jitter.
