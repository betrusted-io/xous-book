# Deferred Response

Deferred response is a variant of synchronous messaging. In this case, the caller blocks, but the callee is free to process new messages (typically to help compute results that evnetually unblock the caller).

As of Xous 0.9.7, the trick to deferred response is different between `scalar` and `memory` messages.

- For `scalar` messages, one needs to store the `msg.sender` field (a `usize`) and delay the `xous::return_scalar(sender, value)` call until the appropriate time.
- For `memory` messages, one needs to store the entire `MessageEnvelope`, so that it does not go out of scope. `memory` messages automatically call the appropriate syscall (`return_memory_offset_valid` for `lend` and `lend_mut`, `unmap_memory` for `send`) in their `Drop` trait implementation.

Future versions of Xous may or may not implement a `Drop` method which automatically returns `scalar` messages when they go out of scope, this is a topic of active discussion. However, as is the case with all the other idioms, the pattern is different from `scalar` and `memory` types, so, regardless, they will be treated with separate examples.

## Scalar Pattern

This is very close to the thing that's actually implemented for synchronizing all the servers during a suspend/resume event.

```rust,noplayground
// api.rs
pub(crate) enum Opcode {
    WaitUntilReady,
    TheBigEvent,
    // ... and other ops
}
```

```rust,noplayground,ignore
// lib.rs:
impl MyService {
    // ... new(), etc.

    /// This will wait until the main loop decides it's ready to unblock us.
    pub fn wait_until_ready(&self) -> Result<(), xous::Error> {
        send_message(self.conn,
            Message::new_blocking_scalar(Opcode::WaitUntilReady.to_usize().unwrap()),
                0, 0, 0, 0
            )
        ).map(|_| ())
    }
}
```

```rust,noplayground,ignore
// main.rs:
fn xmain() -> ! {
    // ... preamble
    let mut waiting = Vec::<MessageSender>::new();
    loop {
        let msg = xous::receive_message(sid).unwrap();
        match FromPrimitive::from_usize(msg.body.id()) {
            Some(Opcode::WaitUntilReady) => xous::msg_blocking_scalar_unpack!(msg, _, _, _, _, {
                // store the message sender;
                // the sender continues to block because `xous::return_scalar()` has not been called
                waiting.push(msg.sender);
                // execution continues on here
            }),
            // .. this loop is still available to do things, even though the callers are blocked ..
            // stuff happens until something triggers TheBigEvent:
            Some(Opcode::TheBigEvent) => {
                for sender in waiting.drain(..) {
                    // the argument is arbitrary. `return_scalar2` can also be used.
                    xous::return_scalar(sender, 1).expect("couldn't unblock sender");
                }
                // All the waiting processes are now unblocked.
            }
            // .. other match statements
        }
    }
    // ... postamble
}
```

## Memory Pattern

```rust,noplayground,ignore
// api.rs
pub(crate) enum Opcode {
    GetDeferredData,
    Event,
    // ... and other ops
}
#[derive(rkyv::Archive, rkyv::Serialize, rkyv::Deserialize)]
pub struct DeferredData {
    pub spec: u32,
    pub description: xous_ipc::String::<128>,
}
```

```rust,noplayground,ignore
// lib.rs:
impl MyService {
    // ... new(), etc.

    pub fn get_data_blocking(&self, spec: u32) -> Result<String, xous::Error> {
        let mut rec = DeferredData {
            spec,
            description: xous_ipc::String::new(),
        };

        let mut buf = Buffer::into_buf(rec).or(Err(xous::Error::InternalError))?;
        buf.lend_mut(self.conn, Opcode::GetDeferredData.to_u32().unwrap()).map(|_| ())?;

        let response = buf.as_flat::<DeferredData, _>().unwrap();
        Ok(String::new(response.description.as_str().unwrap_or("UTF-8 error")))
    }
}
```

```rust,noplayground,ignore
// main.rs:
fn xmain() -> ! {
    // ... preamble
    // if you are sure there will never be multiple deferred messages, you can just use an
    // Option<MessageEnvelope> and .take() to remove it from scope, instead of Vec and .drain()
    let mut storage = Vec::<xous::MessageEnvelope>::new();
    let mut spec: u32 = 0;
    loop {
        let mut msg = xous::receive_message(sid).unwrap();
        match FromPrimitive::from_usize(msg.body.id()) {
            /// This will get processed whenever the server gets scheduled, which has no strict
            /// relationship to the caller's state. However, messages are guaranteed
            /// to be processed in-order.
            Some(Opcode::GetDeferredData) => {
                spec += {
                    // any incoming arguments are processed in a block like this to ensure
                    // that the `msg` has no ownership interference with `spec`.
                    let buffer = unsafe {
                        Buffer::from_memory_message(msg.body.memory_message().unwrap())
                    };
                    let data = buffer.to_original::<DeferredData, _>().unwrap();
                    data.spec
                };
                storage.push(msg);
                // `msg` is now pushed into the scope of `storage`, which prevents `Drop`
                // from being called, thus continuing to block the caller.
            },
            // ... other processing happens, perhaps guided by the value in `spec`
            Some(Opcode::Event) => {
                let result = DeferredData {
                    spec: 0,
                    description: xous_ipc::String::from_str("something happened!"),
                };
                // `drain()` takes `msg` back out of the scope of `storage`, and
                // unless it is bound to another variable outside of this scope,
                // it will Drop and unblock the caller at the end of this block.
                for mut sender in storage.drain(..) {
                    let mut response = unsafe {
                        Buffer::from_memory_message_mut(sender.body.memory_message_mut().unwrap())
                    };
                    response.replace(result).unwrap();
                }
            }
            // .. other match statements
        }
    }
    // ... postamble
}
```
