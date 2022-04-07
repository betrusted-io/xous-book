# Non-Synchronizing Idioms
## Scalar Pattern
A scalar non-synchronizing call has the following characterisics:

- Up to 4 `u32`-sized arguments
- Caller does not block
- Callee does not return any result
- No guarantee of synchronization between caller and callee
  - Side effects may happen at an arbitrary time later
  - Messages are guaranteed to arrive in order

```rust,noplayground
// api.rs
pub(crate) enum Opcode {
    Lights,
    // ... and other ops
}
```

```rust,noplayground
// lib.rs:
impl MyService {
    // ... new(), etc.

    /// Tell the main loop to set the state of lights. When this call exits, all we know is
    /// a message is "en route" to the main loop, but we can't guarantee anything has happened.
    pub fn set_lights(&self, state: bool) -> Result<(), xous::Error> {
        send_message(self.conn,
            Message::new_scalar(Opcode::Lights.to_usize().unwrap()),
                if state {1} else {0},
                0,
                0,
                0
            )
        ).map(|_|())
    }
}
```

```rust,noplayground
// main.rs:
fn xmain() -> ! {
    // ... preamble
    loop {
        let msg = xous::receive_message(sid).unwrap();
        match FromPrimitive::from_usize(msg.body.id()) {
            /// This will get processed whenever the server gets scheduled, which has no strict
            /// relationship to the caller's state. However, messages are guaranteed
            /// to be processed in-order.
            Some(Opcode::Lights) => xous::msg_scalar_unpack!(msg, state, _, _, _, {
                if state == 1 {
                    turn_lights_on();
                } else {
                    turn_lights_off();
                }
            }),
            // .. other match statements
        }
    }
    // ... postamble
}
```

## Memory Pattern
A memory non-synchronizing call has the following characterisics:

- Messages are sent in blocks rounded up to the nearest 4096-byte page size
- Caller does not block
- Callee does not return any result
- No guarantee of synchronization between caller and callee
  - Side effects may happen at an arbitrary time later
  - Messages are guaranteed to arrive in order

```rust,noplayground
// api.rs
pub(crate) enum Opcode {
    // use `rkyv` to serialize a memory message and send
    PushDataRkyv,
    // example of explicit serialization
    PushDataExplicit,
    // ... and other ops
}
/// `rkyv` can be used as a convenience method to serialize data in complex structures.
/// Almost any type can be contained in the structure (enums, other structures), but the
/// type must also `derive` the `rkyv` Archive, Serialize, and Deserialize traits.
/// Thus one cannot simply seralize a `std::string::String`; it must be transcribed into
/// a `xous_ipc::String::<N>` type which has a defined allocation size of `N`.
#[derive(rkyv::Archive, rkyv::Serialize, rkyv::Deserialize)]
pub struct CompoundData {
    pub data: [u8; 1000],
    pub len: u16,
    pub description: xous_ipc::String::<128>,
}
```

```rust,noplayground
// lib.rs:
impl MyService {
    // ... new(), etc.

    /// Send some `data` to the server. It'll get there when it gets there.
    /// This example uses `rkyv` to serialize data into a compound structure.
    pub fn push_data_rkyv(&self, data: &[u8], desc: &str) -> Result<(), xous::Error> {
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
        data.len = data.len() as u16;
        rec.description.append(desc).ok(); // overflows are silently truncated

        // now consume `rec` and turn it into a Xous::Buffer, which can then be mapped into the
        // callee's memory space by `send`
        let buf = Buffer::into_buf(rec).or(Err(xous::Error::InternalError))?;
        buf.send(self.conn, Opcode::PushDataRkyv.to_u32().unwrap()).map(|_| ())
    }
}
```

```rust,noplayground
// main.rs:
fn xmain() -> ! {
    // ... preamble
    let mut storage = Vec::<CompoundData>::new();
    let mut raw_data = [0u8; 32];
    loop {
        let msg = xous::receive_message(sid).unwrap();
        match FromPrimitive::from_usize(msg.body.id()) {
            /// This will get processed whenever the server gets scheduled, which has no strict
            /// relationship to the caller's state. However, messages are guaranteed
            /// to be processed in-order.
            Some(Opcode::PushDataRkyv) => {
                let buffer = unsafe { Buffer::from_memory_message(msg.body.memory_message().unwrap()) };
                // `.to_original()` automatically makes a copy of the data into my process space.
                //    This adds overhead and time, but your original types are restored.
                // `.as_flat()` will use the data directly out of the messages' memory space without copying it,
                //    but it introduces some type complexity. We don't give an example here, but you may find
                //    one in the TRNG's `FillTrng` implementation, where we avoid making two copies of the
                //    data for a more performant implementation.
                let data = buffer.to_original::<PushDataRkyv, _>().unwrap();
                storage.push(data);
            }
            // .. other match statements
        }
    }
    // ... postamble
}
```