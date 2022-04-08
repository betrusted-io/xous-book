# Synchronization Primitives

Synchronization primitives are provided via the Ticktimer Server. This includes mutexes, process sleeping, and condvars.

## Thread Sleeping

Thread sleeping is a primitive that is implemented by the ticktimer
server.

This takes advantage of the fact that a sender will suspend a thread
until a `BlockingScalar` message is responded to.

In order to suspend a thread, simply send a `BlockingScalar` message
to the ticktimer server with an `id` of `1` and an `arg1` indicating
the number of milliseconds to sleep.

If you need  to sleep for more than 49 days, simply send multiple messages.

## Mutex

Mutexes allow for multiple threads to safely access the same data.
Xous Mutexes have two paths: A fast path, and a slow path. Non-contended
Mutexes traverse the fast path and will not need a context switch.
Contended Mutexes will automatically fall back to the slow path.

The core of a Mutex is a single `AtomicUsize`. This value is 0 when
the Mutex is unlocked, and nonzero when it is locked.

### Mutex: Locking

Locking a Mutex involves a simple `try_lock()` operation. In this
operation, atomic instructions are used to replace the value `0`
with the value `1`, failing if this is the case.

```rust,noplayground,ignore
pub unsafe fn try_lock(&self) -> bool {
    self.locked.compare_exchange(0, 1, SeqCst, SeqCst).is_ok()
}
```

If the Mutex is locked, then the current thread will call `yield_slice()`
which hands execution to another thread in the current process in the hope
that the other thread will release its mutex.

This currently occurs three times.

If the lock still cannot be locked, then it is "poisoned". Instead of
swapping `0` for `1`, the thread does an atomic Add of `1` to the current
value. If the resulting value is `1` then the lock was successfully
obtained and execution may continue as normal.

However, if the value is not `1` then the process falls back to the Slow Path. This involves sending a `BlockingScalar` to the ticktimer
server with an `id` of `6` and `arg1` set to the address of the Mutex.

### Mutex: Unlocking

Unlocking a Mutex on the Fast Path simply involves subtracting `1` from
the Mutex. If the previous value was `1` then there were no other
threads waiting on the Mutex.

Otherwise, send a `BlockingScalar` to the ticktimer server with
an `id` of `7` and `arg1` set to the address of the Mutex.

## Condvar

Condvar is Rust's name for "conditional variables". Broadly speaking,
they are instances where one thread takes an area of memory and says
"Wake me up sometime in the future." A different thread can then say
"Wake up one other thread that's waiting on this object." Or it
can say "Wake up all other threads that are waiting on this object."

### Condvar: Waiting for a Condition

To suspend a thread until a condition occurs, or until a timeout hits,
allocate an area of memory for the condvar. Then send a `BlockingScalar`
message to the ticktimer server with an `id` of `8`. Set `arg1` to the
address of the condvar.

In order to add a timeout, set `arg2` to the number of milliseconds to
wait for. Times longer than 49 days are not supported, so multiple
calls will be required. If no timeout is required, pass `0` for `arg2`.

### Condvar: Signaling Wakeups

To wake up another thread, send a `BlockingScalar` message to the ticktimer
server with an `id` of `9`, and set `arg1` to the address of the condvar.
`arg2` should contain the number of blocked threads to wake up.
In order to wake only one thread, pass `1`.