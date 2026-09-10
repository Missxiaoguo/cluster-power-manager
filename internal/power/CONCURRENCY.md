# Power package concurrency guidance

Each host owns one mutex that serializes changes to its pools, CPU membership,
and pool power profiles. Every pool belonging to the host uses that same mutex.

For operations covered by this mutex, the locking rules are:

1. An entry point that mutates pool membership or a pool power profile acquires
   the host mutex exactly once and holds it for the complete operation.
2. Internal mutation helpers assume the host mutex is already held and never
   acquire it themselves. This includes `setCpus`, `moveCpus`, `setPool`,
   `doSetPool`, and `consolidate`.
3. A locking entry point must not call another locking entry point while holding
   the mutex. It calls the corresponding internal helper instead.
4. The mutex remains held while a multi-CPU operation applies the power profile
   so another mutation cannot observe or modify partially updated membership.

Acquiring the mutex twice in the same goroutine deadlocks because Go mutexes are
not reentrant. Keeping acquisition at exported boundaries makes the lock owner
clear and keeps internal call chains from accidentally acquiring it again.

These rules serialize host-scoped pool mutations. Read-side synchronization is
outside the scope of this guidance.
