# `std` API

⚠ The `std` API is only accurate on real hardware and Renode emulation. `std` on hosted mode will use the `std` implementation of the host.

The PDDB is also accessible from `std::fs::File`, as [described here](https://xobs.io/experimental-rust-filesystem-and-path-support-for-xous/):

```rust,noplayground,ignore
fn main() {
    // Create an example file under the `sys.rtc` key
    let mut file = std::fs::File::create("sys.rtc:test").unwrap();
    file.write_all(&[1, 2, 3, 4]).unwrap();
    // Close the file.
    core::mem::drop(file);

    // Open the example file and ensure our data is present
    let mut file = std::fs::File::open("sys.rtc:test").unwrap();
    let mut v = vec![];
    file.read_to_end(&mut v).unwrap();
    assert_eq!(&v, &[1, 2, 3, 4]);
    // Close the file.
    core::mem::drop(file);

    // Remove the test file
    std::fs::remove_file("sys.rtc:test").unwrap();
}
```

The example above shows the process of creating, reading from, and deleting a file.

The main difference from a Unix-like or Windows interface is that the path separator on Xous is `:`.

At this time, PDDB has no restrictions on dict or key names, meaning it's possible to create a key with a `:` in the name. The standard library functions won't be able to disambiguate these paths at this time, and so this character will likely be made illegal in key names. This is somewhat remeniscent of the Windows Registry where \ is a path separator that is illegal in path names yet is allowed in key names.

## Path Conventions

A PDDB Path may be a dict, a dict + a key, a basis + dict, or a basis + dict + key.
In the following examples, the given Basis, Dict, and Key are as follows:

* Basis: `.System`
* Dict: `wlan.networks`
* Key: `Home Wifi`

A canonical path looks like:

![path format](images/betrusted-pddb-path-format.png)

### Examples

* `:Home Wifi` -- A basis named "Home Wifi"
* `:.System:` -- A basis named ".System"
* `wlan.networks` -- A dict named "wlan.networks" in the default basis
* `wlan.networks:recent` -- A dict named "wlan.networks:recent", which may be considered a path, in the default basis. This also describes a key called "recent" in the dict "wlan.networks", depending on whether
* `:.System:wlan.networks` -- A dict named "wlan.networks" in the basis ".System"
* `:.System:wlan.networks:recent` -- a fully-qualified path, describing a key "recent" in the dict "wlan.networks" in the basis ".System". Also describes a dict "wlan.networks:recent" in the basis ".System" when
* `:` -- The root, which lists every basis. Files cannot be created here. "Directories" can be
            created and destroyed, which corresponds to creating and destroying bases.
* `::` -- An empty basis is a synonym for all bases, so this corresponds to listing all dicts in the root of the default basis.
*  -- An empty string corresponds to listing all dicts in root the union basis.

### Corner cases

* `: :` -- A basis named " ". Legal, but questionable
* ` ` -- A dict named " " in the default basis. Legal, but questionable.
* `: ` -- Also a dict named " " in the default basis.
* ` : ` -- A key named " " in a dict called " ". Legal.
* `baz:` -- A dict named "baz" in the default basis with an extra ":" following. Legal.
* `baz:foo:` -- Currently illegal, but may become equal to `baz:foo` in the future.
* `:::` -- An key named ":" in an empty dict in the default basis. Illegal.
* `::::` -- An key named "::" in an empty dict in the default basis. Illegal.
* `::foo` -- A key "foo" in the default basis.
* `:lorem.ipsum:foo:baz` -- A key called "foo:baz" in the basis "lorem.ipsum". May also describe a dict "foo:baz" in the basis "lorem.ipsum" if treated as a directory.
* `:bar:lorem.ipsum:foo:baz` -- A key called "baz" in the dict "lorem.ipsum:foo" in
            the basis "bar", or a dict called "lorem.ipsum:foo:baz". Legal.

Any reference to "default basis" depends on whether the operation is a "read" or a "write":

* "Read" operations come from a union, with the most-recently-added basis taking precedence
* "Write" operations go to the most-recently-added basis that contains the key. If the key does not exist and "Create" was specified, then the file is created in the most-recently-added basis.

## API Status

As of v0.9.9, there is basic support for `std::fs` and `std::path`. As an example, the following features work:

- Opening and closing files
- Reading from and writing to files
- Seeking within an open file
- Creating new files
- Deleting files
- Listing directories
- Creating directories
- Deleting directories

There are a lot of features that do not work, or do not currently make sense. We'll add these features if there is demand:

- Copying files
- Truncating currently-open files
- Duplicating file descriptors
- Getting creation/access/modification times on files
- Symlinks
- Readonly files
- Permissions
- Renaming files

Additionally, the PDDB native API supports callbacks to notify senders of various file events such as deletions and updates. The spec for the native callback API is still a work in progress.

