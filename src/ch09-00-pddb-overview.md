# The Plausibly Deniable DataBase (PDDB) Overview

From the standpoint of Xous, the Plausibly Deniable DataBase (PDDB) is the filesystem layer on top of a raw disk. It plays the role that `FAT` or `ext4` might play in other OSes, combined with `LUKS` or `VeraCrypt`.

The PDDB can be accessed either through a native API, or through Rust's `std::fs::File` layer. `std::fs::File` enables applications that are "naive" to deniability to run. However, deniability-aware applications would use native API calls to create a more seamless user experience.

![dictionary to key mapping example](images/betrusted-pddb-architecture.png)

The PDDB is structured as a `key:value` store divided into `dictionaries` that features multiple overlay views. Each overlay view is called a `Basis` (plural Bases). A Basis has the following properties:

 - The current view is the union of all open Bases
 - In case of namespace conflicts (two keys with the same name in a dictionary):
   - For reads, the value in the most recently unlocked Basis is returned
   - For writes, the value updates an existing key (if one exists) in the most recently unlocked Basis; otherwise, a new key is created in the most recently unlocked Basis.
   - In all cases, the API supports specifically naming a target Basis. This overrides the defaults specified above
 - The default Basis is named `.System`, and it is created when the PDDB is formatted. The PDDB is considered mounted if the `.System` Basis can be found.
 - When a Basis is locked, its data is indistinguishable from free space and hence plausibly deniable.
 - A Basis is unlocked by a name and password combo. If either are lost or forgotten, the Basis is equivalent to having been deleted.

The PDDB documentation is structured into several chapters. Please browse them using the bookmarks on the left hand side.
