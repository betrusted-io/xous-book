# Messages and Message Passing

Messages form the basis of interproces communication on Xous. A process exists in isolation and can only communicate to the outside world by sending messages. The limited API provided by the kernel means that almost all interactions are provided by userspace Servers, which must be communicated with using Messages.

## Connecting to and Disconnecting from Servers

To connect to a server you must supply it an `Server ID`. A `Server ID` is a 16-byte value of some sort that is shared in a universal namespace. If you know a Server's `Server ID` then you can connect to that Server.

There are a few well-known `Server ID`s. These include bare minimum IDs that are required by any process to do anything useful. They are:

* *b"xous-log-server "*: Output log messages to the console, as well as basic `println!()` support
* *b"ticktimer-server"*: Used for `sleep()` as well as time-based Mutexes
* *b"xous-name-server"*: A central nameserver that is used for connecting to all other servers

To connect to a Server, call `xous::connect()`. For example, to connect to the `ticktimer-server`, call:

```cs
let connection_id = xous::connect(xous::SID::from_bytes(b"ticktimer-server").unwrap())?;
```

This will provide you a Connection to that server. If the Server is not available, the call will block until it is created. To fail if the server does not exist, use `try_connect()` instead of `connect()`.

## Connection Limitations

Connections are limited on a per-process basis. Each process may only establish a connection to at most 32 servers. When this number is exceeded, `xous::connect()` will return `Error::OutOfMemory`.

If you call `xous::connect()` twice with the same `Server ID`, then you will get the same `connection_id`.

## Disconnecting

To disconnect from a server, call `unsafe { xous::disconnect(connection_id)};`. This function is `unsafe` because you can copy connection IDs, so it is up to you to ensure that they are no longer in use when disconnecting.

For example, if you `connect()` to a Server and spawn a thread with that connection ID, you should only call `disconnect()` once that thread has finished with the connection. Similarly, if you `Copy` the connection ID to the thread, you must make sure that **both** uses of the Connection ID are destroyed prior to disposing of the connection.

Because of this, it is recommended that you use an `ARC<CID>` in order to ensure that the connection is only closed when it is no longer in use.

Furthermore, recall that subsequent calls to `connect()` with the same argument will reuse the `connection_id`. Because of this, it is vital that you only call `disconnect()` when you are certain that all instances are finished with the connection.

## Message Overview

Messages come in five kinds: Scalar, BlockingScalar, Borrow, MutableBorrow, and Send. `Scalar` and `Send` messages are nonblocking and return immediately, while the others wait for the Server to respond.

`Borrow`, `MutableBorrow`, and `Send` all detach memory from the client and send it to the server.

## Scalar and BlockingScalar Messages

These messages allow for sending four `usize`s of data plus one `usize` of command. This can be used to send short updates to the Server. `Scalar` messages return to the client immediately, meaning the Server will receive the message after a short delay.

`BlockingScalar` messages will pause the current thread and switch to the Server immediately. If the message is handled quickly, the Server can respond to the message and switch back to the Client before its quantum expires.

`BlockingScalar` messages can return one or two `usize`s worth of data by returning `Result::Scalar1(usize)` or `Result::Scalar2(usize, usize)`.

As an example of what can be done, the ticktimer server uses `BlockingScalar` messages to implement `msleep()` by delaying the response until a timer expires.

## Borrow, MutableBorrow, and Send Messages

These messages allow for sending memory from one process to another. Memory must be page-sized and aligned, but may be any memory available to a process. For example, a hardware process may want to reserve all MMIO peripherals in the system and then share them with processes as desired.

The memory message types allow for one `usize` worth of tag data which can be used to describe what the message is used for.

Furthermore, messages may also contain two advisory fields: `offset` and `valid`. These fields may be used to define an offset with the memory block where interesting data occurs. Similarly, the `valid` field could be used to define how large the data is.

When memory is passed via `MutableBorrow` then the memory is mapped into the Server's address space as writable. Addtionally, the `offset` and `valid` fields become writable and may be updated in the server. As an example, if a Server implemented `bzero()` to clear a memory range to zero, then it might clear the contents of the buffer, then set both `offset` and `valid` to 0.

Internally, the `MutableBorrow` is updated by passing the new fields to `ReturnMemory()` where it gets updated in the client.