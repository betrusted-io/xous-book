# Forwarding Messages

Because server IDs are used to protect APIs, there arises occasions where
servers need to be firewalled: a private server within a crate may implement
a range of powerful and dangerous APIs, of which only a small portion should
be revealed to external callers.

The general idiom in this case is to:

1. Create a process-private server that contains all the APIs. The server is not registered with `xous-names`; it is entirely a secret within the crate.
2. Create a process-public server that contains only the public APIs. This server is registered with `xous-names` and it may have a connection limit of `None`, e.g., anyone and everyone may connect to it.
3. Certain messages are forwarded from the process-public server to the process-private server.

In order to support this idiom, messages have a `.forward()` call. Usage is straightfoward:

```rust,noplayground,ignore
    // A private server that can do many powerful things
    let cm_sid = xous::create_server().expect("couldn't create connection manager server");
    let cm_cid = xous::connect(cm_sid).unwrap();
    thread::spawn({
        move || {
            connection_manager::connection_manager(cm_sid);
        }
    });

    loop {
        let mut msg = xous::receive_message(net_sid).unwrap();
        // .. other code

        // These messages are forwarded on to the private server
        // This one is a `lend` of a memory message
        Some(Opcode::SubscribeWifiStats) => {
            msg.forward(
                cm_cid,
                ConnectionManagerOpcode::SubscribeWifiStats as _)
            .expect("couldn't forward subscription request");
        }
        // This one is a `blocking_scalar` scalar message type
        Some(Opcode::UnsubWifiStats) => {
            msg.forward(
                cm_cid,
                connection_manager::ConnectionManagerOpcode::UnsubWifiStats as _)
            .expect("couldn't forward unsub request");
        },
    }
```

Other usage notes:
- Message types cannot be transformed across the forwarding boundary.
- You are allowed to inspect a Memory `msg` by unpacking it into a `Buffer`, but you must make sure the `Buffer` goes out of scope before calling `.forward()` (perhaps by putting the inspection operation within its own block, e.g. a pair of curly braces).

