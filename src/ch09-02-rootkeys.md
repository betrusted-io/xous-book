# Deriving The PDDB's Keys

This chapter examines the cryptopraphic material used to encrypt the PDDB, and traces its origin all the way back to the hardware root of trust. It assumes you are familiar with the general structure of the PDDB.

## Basis Keys

A Basis within the PDDB holds a virtual filesystem that is unionized with the other Bases. A Basis is protected with a `name` and `password` combination. Neither the `name` nor the `password`, nor a hash or salt for a password, is stored within the PDDB, as such records would be a sidechannel revealing the existence of a secret Basis. Thus the confidentiality of a Basis is derived entirely from the strength of the `password`, but there is a generic, per-device salt (perhaps more accurately called a "pepper") that means brute force attackers must prepare hash tables unique to each device.

A Basis is defined by two keys:
  - A 256-bit page table key, used to derive an AES cipher run in ECB
  - A 256-bit data key, used to derive an AES-GCM-SIV cipher

The cryptographic matter pertaining specifically to the PDDB is stored in raw FLASH in a header with the following structure:

```rust,noplayground,ignore
#[repr(C)]
pub(crate) struct StaticCryptoData {
    /// a version number for the block
    pub(crate) version: u32,
    /// aes-256 key of the system basis page table, encrypted with the User0 root key, and wrapped using NIST SP800-38F
    pub(crate) system_key_pt: [u8; WRAPPED_AES_KEYSIZE],
    /// aes-256 key of the system basis, encrypted with the User0 root key, and wrapped using NIST SP800-38F
    pub(crate) system_key: [u8; WRAPPED_AES_KEYSIZE],
    /// a pool of fixed data used for salting. The first 32 bytes are further subdivided for use in the HKDF.
    pub(crate) salt_base: [u8; 4096 - WRAPPED_AES_KEYSIZE * 2 - size_of::<u32>()],
}
```

The structure is sized to be exactly one page of memory, with the "remaining" data filled with TRNG-derived salt. The version number is considered a "hint", as it is not signature protected and there are no anti-rollback measures.

The PDDB has a default Basis called `.System`, which has its page table and data keys stored as wrapped keys by the device's root enclave. It is the only Basis treated in this manner. All other Bases are derived from the `name` and `password` of the basis, as hashed by `salt_base`. Any Basis that is not the `.System` Basis is referred to as a "secret Basis".

### Secret Basis Key Derivation

A secret Basis key derivation is performed using the folowing algorithm, implemented in Rust but presented here in Python for clarity:

```python
for name, pw in basis_credentials.items():
    # Basis names are limited to 64 bytes encoded as UTF-8.
    # Copy the Basis name into a 64-byte array that is initialized with all 0's
    bname_copy = [0]*64
    i = 0
    for c in list(name.encode('utf-8')):
        bname_copy[i] = c
        i += 1

    # Passwords are limited to 72 bytes encoded as UTF-8. They are
    # always null-terminated, so a 73-byte 0-array is prepared.
    plaintext_pw = [0]*73
    pw_len = 0
    for c in list(pw.encode('utf-8')):
        plaintext_pw[pw_len] = c
        pw_len += 1
    pw_len += 1 # For the null termination

    # Hash the 64-byte basis name, 73-byte password and the `salt_base` using SHA512/256.
    hasher = SHA512.new(truncate="256")
    hasher.update(salt_base[32:]) # from byte 32 until the end of the salt region (couple kiB)
    hasher.update(bytes(bname_copy))
    hasher.update(bytes(plaintext_pw))
    derived_salt = hasher.digest()

    # Use the first 16 bytes of the derived salt and the null-terminated plaintext password
    # to drive a standard `bcrypt` with a work factor of 7. We can only do 7 because the
    # target hardware is a single-issue, in-order 100MHz RV32-IMAC.
    bcrypter = bcrypt.BCrypt()
    hashed_pw = bcrypter.crypt_raw(plaintext_pw[:pw_len], derived_salt[:16], 7)

    # Derive a key for the page table, using HKDF/SHA256, plus the first 32 bytes of salt,
    # and an info word of "pddb page table key"
    hkdf = HKDF(algorithm=hashes.SHA256(), length=32, salt=pddb_salt[:32], info=b"pddb page table key")
    pt_key = hkdf.derive(hashed_pw)

    # Derive a key for the data pages, using KHDF/SHA256, plus teh first 32 bytes of salt,
    # and an info word of "pddb data key"
    hkdf = HKDF(algorithm=hashes.SHA256(), length=32, salt=pddb_salt[:32], info=b"pddb data key")
    data_key = hkdf.derive(hashed_pw)

    keys[name] = [pt_key, data_key]
```

### System Basis Key Derivation

The System basis keys for the PDDB are wrapped by the Precursor's on-board `root-keys` block. They are wrapped using the `User0` key (also referred to as the `user root key`). Please see the Xous wiki for more information on the [layout of the root key block](https://github.com/betrusted-io/betrusted-wiki/wiki/Secure-Boot-and-KEYROM-Layout#key-rom-format).

```python
# Pseudocode for accessing the root keys from the key block.
# The user key is at offset 40, 32 bytes long.
user_key_enc = get_key(40, keyrom, 32)
# The pepper is at offset 248, 16 bytes long.
pepper = get_key(248, keyrom, 16)
# The password type is XOR'd in to the pepper, to make it less convenient
# for pre-computed rainbow tables to re-use their work across various passwords.
pepper[0] = pepper[0] ^ 1 # encodes the "boot" password type into the pepper

# The unlock PIN is up to 72 bytes long, encoded as UTF-8. Prepare a null-terminated
# version of the password. Here the pseudocode refers to the "unlock PIN" as "boot_pw"
boot_pw_array = [0] * 73
pw_len = 0
for b in bytes(boot_pw.encode('utf-8')):
    boot_pw_array[pw_len] = b
    pw_len += 1
pw_len += 1 # null terminate, so even the null password is one character long

# Use bcrypt on the password + pepper to derive a key. See above for notes on
# the use of a work factor of 7.
bcrypter = bcrypt.BCrypt()
hashed_pw = bcrypter.crypt_raw(boot_pw_array[:pw_len], pepper, 7)

# Expand the derived 24-byte password to 32 bytes with SHA512/256.
hasher = SHA512.new(truncate="256")
hasher.update(hashed_pw)
user_pw = hasher.digest()

# XOR the derived key with the encrypted, stored user_key to get the plaintext user_key
user_key = []
for (a, b) in zip(user_key_enc, user_pw):
    user_key += [a ^ b]

# Derive an anti-rollback user state. This takes the user_key and hashes it
# repeatedly, a maximum of 255 times. Every time we need to version the system
# with anti-rollback, we increase a counter stored at offset 254 in the keyrom
# and subtract this from 255. This means newer versions can still access older
# versions by applying N extra hashes (where N is the number of versions older)
# that need to be accessed, while making it impossible for an application holding
# the current key to guess what the next key might be in the anti-rollback sequence.
rollback_limit = 255 - int.from_bytes(keyrom[254 * 4 : 254 * 4 + 4], 'little')
for i in range(rollback_limit):
    hasher = SHA512.new(truncate="256")
    hasher.update(bytes(user_key))
    user_key = hasher.digest()
# user_key now contains the actual key that is used to wrap the PDDB system keys.

# Access the wrapped system keys from the StaticCryptoData structure (as defined above)
wrapped_key_pt = static_crypto_data[4:4+40]
wrapped_key_data = static_crypto_data[4+40:4+40+40]

# Extract the .System key by unwrapping the system keys with AES-KWP, per NIST SP800-38F
# (or identically RFC 5649).
key_pt = aes_key_unwrap_with_padding(bytes(user_key), bytes(wrapped_key_pt))
key_data = aes_key_unwrap_with_padding(bytes(user_key), bytes(wrapped_key_data))

# The key unwrapping method will fail with very high probability if the provided
# user password is incorrect. Thus the results from the key unwrap must be checked
# and handled for the case of an incorrect boot password.
```

The application of the page table and data keys are discussed in the chapter on the [Basis Internal Structure](ch09-01-basis.md). Note that the AES-GCM-SIV for the data keys does require AAD, which includes the device-specific DNA as well as the version number of the PDDB.
