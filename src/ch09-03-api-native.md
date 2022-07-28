# Native API

The "native" API for the PDDB is a set of `xous`-specific calls that one can use to manage and access the PDDB.

All of the native API method signatures can be found in [`lib.rs`](https://github.com/betrusted-io/xous-core/blob/main/services/pddb/src/lib.rs) and the [frontend](https://github.com/betrusted-io/xous-core/blob/main/services/pddb/src/frontend/pddbkey.rs) module. Proper Rustdoc for these is on the list of things to do. This chapter treats the native API at an abstract level, with a focus on code examples rather than 100% coverage of every feature.

## The `Pddb` Object

Any native access to the PDDB goes through the `Pddb` object. You will need to add the `pddb` service to your `Cargo.toml` file, and then create a PDDB object like such:

```rust,noplayground,ignore
let pddb = pddb::Pddb::new();
```

The method you'll use the most is the `.get()` method on the `Pddb` object. It has a signature like this:

```rust,noplayground,ignore
pub fn get(
    &self,
    dict_name: &str,
    key_name: &str,
    basis_name: Option<&str>,
    create_dict: bool,
    create_key: bool,
    alloc_hint: Option<usize>,
    key_changed_cb: Option<impl Fn() + 'static + Send>
) -> Result<PddbKey>
```

`dict_name` and `key_name` are the names of the dictionary and key. They can be any valid UTF-8 string but you should avoid the `:` character as that is the path separator. Dictionary names can be up to 111 bytes long, and key names up to 95 bytes long.

`basis_name` is an optional Basis name, that can be any valid UTF-8 string that avoids `:`. Basis names can be up to 64 bytes long. If `basis_name` is `None`, then the Basis to use will be computed as follows:

- If the key exists in any Basis, the most recently open Basis is accessed for that key.
- If the key does not exist, then either it returns an error (depending on the flags), or it creates the key in the most recently opened Basis.

The System basis (available at `pddb::PDDB_DEFAULT_SYSTEM_BASIS`) is the fall-back as it is always mounted. Administrative operations, especially ones that deal with non-sensitive settings, should generally specify the system basis explicitly so that options don't mysteriously reset or disappear when secret Bases are mounted or unmounted.

`create_dict` specifies to create the dictionary, if it does not already exist.

`create_key` specifies to create the key, if it does not already exist.

`alloc_hint` is a hint to the system as to how much space it should allocate for the key. This parameter is only used when the key is created; later calls will ignore this. Generally, system performance will be much faster if you provide an `alloc_hint`, especially for small keys. Without the hint, the key size starts at 4 bytes, and every byte written deletes and re-allocates the key to grow to the next byte size. This is mitigated partially by buffering in the `Read` API, but if you have a sense of how big a key might be in advance, it's helpful to pass that on.

`key_changed_cb` is a callback meant to notify the key accessor that the key's status has changed. Typically this would be the result of someone locking or unlocking a secret Basis. As a `static + Send` closure, generally one would place a call to `xous::send_message()` inside the call, that would send a message to another server to handle this situation, similar to this:

```rust,noplayground,ignore
static SELF_CONN: AtomicU32 = AtomicU32::new(0); // 0 is never a valid CID

pub(crate) fn basis_change() {
    if SELF_CONN.load(Ordering::SeqCst) != 0 {
        xous::send_message(SELF_CONN.load(Ordering::SeqCst),
            Message::new_scalar(Opcode::BasisChange.to_usize().unwrap(), 0, 0, 0, 0)
        ).unwrap();
    }
}

// In the main loop, there would be some code similar to this:
fn main () {
    let xns = xous_names::XousNames::new().unwrap();
    let sid = xns.register_name("_My Server_", None).unwrap();
    let conn = xous::connect(sid).unwrap();
    SELF_CONN.store(conn, Ordering::SeqCst);
    // at this point SELF_CONN will be non-zero
}

// later on, a piece of code might refer to the function as follows:
fn myfunc() {
    let key = pddb.get(
        &dictname,
        &keyname,
        None,
        false, false, None,
        Some(crate::basis_change)
    ).unwrap();
    // if you want the callback to be None, you must specify the type of None as follows:
    let key2 = pddb.get(
        &dictname,
        &key2name,
        None,
        false, false, None,
        None::<fn()>
    ).unwrap();
}
```

## Access Examples

The general flow for writing to the PDDB is as follows:

1. Serialize your data into a `[u8]` slice
2. Fetch the key using the `pddb.get()` method
3. Use any `Write` trait from `std::io::Write` to write the data
4. Sync the PDDB (highly recommended, especially at this early stage)

The general flow for reading data from the PDDB is as follows:

1. Fetch the key using the `pddb.get()` method
2. Use any `Read` trait from `std::io::Read` to write the data
3. Deserialize the data into a `[u8]` slice

Below is an example of writing a key.

```rust,noplayground,ignore
use std::io::Write;

let pddb = pddb::Pddb::new();
let record = PasswordRecord {
    version: VAULT_PASSWORD_REC_VERSION,
    description,
    username,
    password,
    // ... other line items
};
let ser = serialize_password(&record);
let guid = self.gen_guid();
log::debug!("storing into guid: {}", guid);
pddb.borrow().get(
    VAULT_PASSWORD_DICT,
    &guid,
    None, true, true,
    Some(VAULT_ALLOC_HINT), Some(crate::basis_change)
)
.expect("error accessing the PDDB")
.write(&ser)
.expect("couldn't write data");
log::debug!("syncing...");
pddb.borrow().sync().ok();
```

And below is an example of reading a key.

```rust,noplayground,ignore
use std::io::Read;

let pddb = pddb::Pddb::new();
let mut record = pddb.borrow().get(
    VAULT_PASSWORD_DICT,
    &key,
    None,
    false, false, None,
    Some(crate::basis_change)
).expect("couldn't find key");
let mut data = Vec::<u8>::new();
match record.read_to_end(&mut data) {
    Ok(_len) => {
        if let Some(pw) = deserialize_password(data) {
            // pw now contains the deserialized data
        } else {
            // handle errors
        }
    }
    Err(e) => // handle errors
}
```

## Management

The Pddb object also has methods to help manage the PDDB, including:

- [`list_basis(..)`](https://github.com/betrusted-io/xous-core/blob/ac1f7465667aabb7bc7fa3e3e9ced8e980ea4a0c/services/pddb/src/lib.rs#L163)
- [`create_basis(..)`](https://github.com/betrusted-io/xous-core/blob/ac1f7465667aabb7bc7fa3e3e9ced8e980ea4a0c/services/pddb/src/lib.rs#L208)
- [`unlock_basis(..)`](https://github.com/betrusted-io/xous-core/blob/main/services/pddb/src/lib.rs#L230)
- [`lock_basis(..)`](https://github.com/betrusted-io/xous-core/blob/main/services/pddb/src/lib.rs#L252)
- [`delete_basis(..)`](https://github.com/betrusted-io/xous-core/blob/main/services/pddb/src/lib.rs#L274)
- [`delete_key(..)`](https://github.com/betrusted-io/xous-core/blob/main/services/pddb/src/lib.rs#L386)
- [`delete_dict(..)`](https://github.com/betrusted-io/xous-core/blob/main/services/pddb/src/lib.rs#L435)
- [`list_keys(..)`](https://github.com/betrusted-io/xous-core/blob/main/services/pddb/src/lib.rs#L498)
- [`list_dict(..)`](https://github.com/betrusted-io/xous-core/blob/main/services/pddb/src/lib.rs#L560)
- [`is_mounted_blocking(..)`](https://github.com/betrusted-io/xous-core/blob/ac1f7465667aabb7bc7fa3e3e9ced8e980ea4a0c/services/pddb/src/lib.rs#L143) - blocks until the PDDB is mounted

For non-blocking queries of PDDB mount status, there is an object called `PddbMountPoller` which has a method [`is_mounted_nonblocking()`](https://github.com/betrusted-io/xous-core/blob/ac1f7465667aabb7bc7fa3e3e9ced8e980ea4a0c/services/pddb/src/lib.rs#L44).

