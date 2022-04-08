# Synchronizing

## Scalar Pattern
A scalar synchronizing call has the following characterisics:

- Up to 4 `u32`-sized arguments
- Caller blocks until the callee returns
- Callee may return up to 2 `u32`-sized values

```rust,noplayground
// api.rs
pub(crate) enum Opcode {
    LightsSync,
    // ... and other ops
}
```

```rust,noplayground,ignore
// lib.rs:
impl MyService {
    // ... new(), etc.

    /// Tell the main loop to set the state of lights. This blocks until we get a confirmation code,
    /// which in this case was the last state of the lights.
    pub fn set_lights_sync(&self, state: bool) -> Result<bool, xous::Error> {
        match send_message(self.conn,
            Message::new_blocking_scalar(Opcode::LightsSync.to_usize().unwrap()),
                if state {1} else {0},
                0,
                0,
                0
            )
        ) {
            // match to `xous::Result::Scalar2(val1, val2)` for the case of two values returned
            Ok(xous::Result::Scalar1(last_state)) => {
                if last_state == 1 {
                    Ok(true)
                } else {
                    Ok(false)
                }
            }
            _ => {
                Err(xous::Error::InternalError)
            }
        }
    }
}
```

```rust,noplayground,ignore
// main.rs:
fn xmain() -> ! {
    // ... preamble
    loop {
        let msg = xous::receive_message(sid).unwrap();
        match FromPrimitive::from_usize(msg.body.id()) {
            Some(Opcode::LightsSync) => xous::msg_blocking_scalar_unpack!(msg, state, _, _, _, {
                let last_state = lights_current_state();
                if state == 1 {
                    turn_lights_on();
                } else {
                    turn_lights_off();
                }
                if last_state {
                    // alternative form is `xous::return_scalar2(msg.sender, val1, val2)`
                    xous::return_scalar(msg.sender, 1).expect("couldn't return last state");
                } else {
                    xous::return_scalar(msg.sender, 0).expect("couldn't return last state");
                }
            }),
            // .. other match statements
        }
    }
    // ... postamble
}
```

## Memory Pattern
A memory synchronizing call has the following characterisics:

- Messages are sent in blocks rounded up to the nearest 4096-byte page size
- Caller blocks until the data is returned
- Callee returns data by overwriting the same page(s) of memory that were sent

This example also shows how to do a memory message without `rkyv`. This is useful
for situations that can't have an `rkyv` dependency, or if you just prefer to do
things in a low-level fashion.

```rust,noplayground,ignore
// api.rs
pub(crate) enum Opcode {
    // use `rkyv` to serialize a memory message and send
    PushDataRkyv,
    // example of explicit serialization
    PushDataExplicit,
    // ... and other ops
}
#[derive(rkyv::Archive, rkyv::Serialize, rkyv::Deserialize)]
pub struct CompoundData {
    pub data: [u8; 1000],
    pub len: u16,
    pub description: xous_ipc::String::<128>,
}
/// The minimum size serialized is always one page (4096 bytes). Even if we made this smaller,
/// a full 4096 bytes are always allocated and cleared. `rkyv` simply hides this detail. So,
/// for raw implementations let's just expose this directly.
pub struct RawData {
    raw: [u8; 4096],
}
```

```rust,noplayground,ignore
// lib.rs:
impl MyService {
    // ... new(), etc.

    /// Send some `data` to the server. It'll get there when it gets there.
    /// This example uses `rkyv` to serialize data into a compound structure.
    pub fn push_and_get_data_rkyv(&self, data: &mut [u8], desc: &str) -> Result<(), xous::Error> {
        let mut rec = CompoundData {
            data: [0u8; 1000],
            len: 0,
            description: xous_ipc::String::new(),
        };
        if data.len() > rec.data.len() {
            return Err(xous::Error::OutOfMemory);
        }
        for (&s, d) in data.iter().zip(rec.data.iter_mut()) {
            *d = s;
        }
        rec.len = data.len() as u16;
        rec.description.append(desc).ok(); // overflows are silently truncated

        // now convert it into a Xous::Buffer, which can then be lent to the server
        let mut buf = Buffer::into_buf(rec).or(Err(xous::Error::InternalError))?;
        buf.lend_mut(self.conn, Opcode::PushDataRkyv.to_u32().unwrap()).map(|_| ())?;

        let response = buf.as_flat::<CompoundData, _>().unwrap();
        if response.data.len() > data.len() || response.data.len() > response.data.len() {
            Err(xous::Error::OutOfMemory)
        } else {
            // copy the data back
            for (&s, d) in response.data[..response.len as usize].iter().zip(data.iter_mut()) {
                *d = s;
            }
            Ok(())
        }
    }

    /// Send 32 bytes of `data` to a server. This example uses explicit serialization into a raw buffer.
    pub fn push_data_manual(&self, data: &mut [u8; 32]) -> Result<(), xous::Error> {
        // RawData can be sized smaller, but all IPC memory messages are rounded up to the nearest page
        // The sizing here reflects that explicitly. Using `rkyv` does not change this, it just hides it.
        let mut request = RawData { raw: [0u8; 4096] };
        for (&s, d) in data.iter().zip(request.raw.iter_mut()) {
            *d = s;
        }
        let buf = unsafe {
            xous::MemoryRange::new(
                &mut request as *mut RawData as usize,
                core::mem::size_of::<RawData>(),
            )
            .unwrap()
        };
        let response = xous::send_message(
            self.conn,
            xous::Message::new_lend_mut(
                Opcode::PushDataExplicit.to_usize().unwrap(),
                buf,
                None, // valid and offset are not used in explicit implementations
                None, // and are thus free to bind to other applications
            ),
        );
        match response {
            Ok(xous::Result::MemoryReturned(_offset, _valid)) => {
                // contrived example just copies whatever comes back from the server
                let response = buf.as_slice::<u8>();
                for (&s, d) in response.iter().zip(data.iter_mut()) {
                    *d = s;
                }
                Ok(())
            }
            Ok(_) => Err(xous::Error::InternalError), // wrong return type
            Err(e) => Err(e)
        }
    }
}
```

```rust,noplayground,ignore
// main.rs:
fn xmain() -> ! {
    // ... preamble
    let mut storage = Vec::<CompoundData>::new();
    let mut raw_data = [0u8; 32];
    loop {
        let mut msg = xous::receive_message(sid).unwrap();
        match FromPrimitive::from_usize(msg.body.id()) {
            /// This will get processed whenever the server gets scheduled, which has no strict
            /// relationship to the caller's state. However, messages are guaranteed
            /// to be processed in-order.
            Some(Opcode::PushDataRkyv) => {
                let mut buffer = unsafe {
                    Buffer::from_memory_message_mut(msg.body.memory_message_mut().unwrap())
                };
                let mut data = buffer.to_original::<PushDataRkyv, _>().unwrap();
                storage.push(data);
                // A contrived return value.
                data.len = 1;
                data.data[0] = 42;
                // Note that you can stick *any* `rkyv`-derived struct
                // into the buffer as a return "value". We just happen to re-use
                // the same structure defintion here for expedience
                // However, it's up to the recipient to know the returned type,
                // and to deserialize it correctly. Nothing prevents type mismatches
                // across IPC boundaries!
                buffer.replace(data).expect("couldn't serialize return");
                // `msg` goes out of scope at this point, triggering `Drop` and thus unblocking the caller
            },
            Some(Opcode::PushDataExplicit) => {
                let body = msg.body.memory_message_mut().expect("incorrect message type received");
                let mut data = body.buf.as_slice_mut::<u8>();
                for (&s, d) in data.iter().zip(raw_data.iter_mut()) {
                    *d = s;
                }
                // Very contrived example of "returning" data. Just poke something into the first byte.
                data[0] = 42;
                // there is no `replace()` because `data` is the original message memory: this is
                // unlike the previous example where `to_original()` creates a copy of the data.

                // `msg` goes out of scope at this point, triggering `Drop` and thus unblocking the caller
            }
            // .. other match statements
        }
    }
    // ... postamble
}
```
