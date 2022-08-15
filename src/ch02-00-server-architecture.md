# Server Architecture

This chapter is written for kernel maintainers.

Application programmers should see [caller idioms](ch07-02-caller-idioms.md) and [messages](ch07-00-messages.md) for details on how to use servers.

## What is the difference between a Thread and a Server?

A Server is a messaging construct, whereas threads refer to execution flow. You can have a thread without a Server, and you can have multiple Servers referenced in a thread.

Threads in Xous are conventional: just a PC + stack that runs in a given process space. A single process can have up to 32 threads, and each thread will run until their time slice is up; or they yield their time slice by either explicitly yielding, or blocking on something (such as a blocking message).

Blocking messages dovetails into the concept of servers: in Xous, server is basically just a 128-bit ID number that servers as a mailbox for incoming messages. There is a limit of 128 servers in Xous across all processes and threads. Within that limit, one can allocate all the servers they want for a given thread, although it's not terribly useful to do that.

Messages specify a 128-bit server ID as a recipient; and, the typical idiom (although it doesn't have to be this way) is for a thread to wake up, initialize, allocate a server ID, and then wait for a message to arrive in its inbox.

If no message arrives, the thread consumes zero time.

Once a message arrives, the thread will be unblocked to handle the message when its quantum comes up. The thread may receive a quantum through the normal round-robin pre-emptive scheduler, but it could also receive a quantum in the case that a blocking message is sent from another thread. What happens then is the sender yields the remainder of its time to the receiving server, so that the message may be handled immediately.

As a counter-example, a "valid" but *not recommended* way to communicate between threads in Xous is to do something like:

```rust,noplayground,ignore
let sem = Arc::new(AtomicBool::new(false));
thread::spawn(
    let sem = sem.clone();
    move || {
        loop {
            if sem.load(Ordering::SeqCst) {
                // do something useful here

                // do it just once
                sem.store(false, Ordering::SeqCst);
            } else {
                // we could be nice and yield our quantum to another thread
                xous::yield_slice();
                // but even if you forget to yield, eventually, the pre-emptive
                // scheduler will stop polling and allow another thread to run.
            }
        }
    }
);

// Later in the parent thread, use this to trigger the child thread:
sem.store(true, Ordering::SeqCst);
```

The "bad example" above starts a thread that just polls the `sem` variable until it is `true` before a one-shot execution of the thing it's supposed to do.

The problem with this construction is that it will constantly run the CPU and always take a quantum of time to poll `sem`. This is very inefficient; it actually burns more battery, and has a material impact on user experience to do it this way.

A more "Xous" way to do this would be:

```rust,noplayground,ignore
let sid = xous::create_server().unwrap();
let conn = xous::connect(sid).unwrap();
thread::spawn(
    // sid automatically clones here
    move || {
        loop {
            let msg = xous::receive_message(sid).unwrap();
            // typically one would decode an opcode from the message body, so you can dispatch more than one function.
            let _opcode: Option<ActionOp> = FromPrimitive::from_usize(msg.body.id());
            // but in this case, we have exactly one thing, so:

            // do something useful here

            // indicate that we got our thing done
            xous::return_scalar(msg.sender, retval).unwrap();
        }
    }
);

// Later in the parent thread, use this to trigger the child thread:
xous::send_message(conn,
    Message::new_blocking_scalar(0 /* this is the opcode field */,
    0, 0, 0, 0) // up to 4 "scalar" arguments can be sent as well
).unwrap();
```

The above `send_message()` would yield the remaining quantum of time for the parent thread, and dispatch into the child thread. The child would receive the message, do "something useful" and return a value to the caller. Assuming there was still time left in the quantum, this `return_scalar` would return execution back to the parent thread!

If the message was not blocking, the parent thread would continue executing until its quantum is completed, and then the child thread would handle the message only then. Assuming the child  thread can handle the response very quickly, it would yield the remainder of its quantum once it completed doing "something useful" and it returned to the top of its loop where it calls `receive_message()`, and found its input queue to be empty.

Thus, Xous is carefully coded such that everything blocks if it's not being used, using the idiom above. Crates like `crossbeam` are implemented using `condvar` which intenally uses servers and messages to ensure that blocking waits are efficient.

So, in general, if the CPU load bar is pegged to 100% and nothing is "going on" (perhaps just a spin-wait), it's considered a bug.
