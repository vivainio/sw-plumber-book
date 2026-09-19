---
icon: lucide/shield
---

# Linux kernel keyrings

Linux provides building blocks for keeping encryption keys, controlling
access to them, and restricting what programs can do. **Kernel keyrings** provide a way to retain keys in the kernel and grant
applications access through explicit permissions.

Kernel keyrings date back to Linux 2.6. They are often unfamiliar despite
being available independently of desktop password managers.

## Session keyrings: access inherited by a process family

A keyring contains references to keys. A key has a type, description,
payload, identifier, and access permissions. The kernel stores the payload
and checks permissions when a program requests it.

Processes can subscribe to a **session keyring**. Children inherit the
subscription across `fork()`, and it survives `execve()`. A launcher can
create a fresh keyring before executing an application, making its keys
available to that application's descendants.
[session-keyring(7)](https://man7.org/linux/man-pages/man7/session-keyring.7.html)

The similarly named **process keyring** is unsuitable for this inheritance
pattern: it does not survive an ordinary fork/exec chain in the same way.
The user keyring has a different purpose again: sharing keys across a UID.
Choose the anchor intentionally rather than relying on the login's defaults.
[Kernel key retention service](https://docs.kernel.org/security/keys/core.html)

This facility needs no GUI or D-Bus daemon. It is separate from GNOME
Keyring and the Secret Service API used by desktop applications.

## Ownership and possession are different permissions

A key has permission classes for its **possessor**, owner, group, and other
processes. Possession follows accessible links from the process's keyring
subscriptions. A process can possess a key without receiving access merely
because its UID matches the key's owner. Conversely, owner permissions can
grant access to another process under the same account.

For session-specific access, restrict both the keyring and its keys. Remove
owner/group/other grants that would bypass possession. Avoid linking the
session's keys into a shared user keyring. Knowing a numeric key ID alone
does not bypass permission checks.
[keyrings(7)](https://man7.org/linux/man-pages/man7/keyrings.7.html)

The available rights include view, read, write, search, link, and set
attributes. A launcher may need broader rights during setup than readers
need afterward. In particular, granting link or permission-changing rights
to every descendant lets those descendants broaden access.
[keyctl_setperm(3)](https://man7.org/linux/man-pages/man3/keyctl_setperm.3.html)

## User keyrings: keys shared by your normal account

Each UID has a **user keyring**, addressed as `@u` by `keyctl` or
`KEY_SPEC_USER_KEYRING` (`-4`) by the API. Programs running as your normal
local user can use it across independent shells and application launches.
No dedicated service account or common launcher is required. Search `@u`
explicitly: automatic key lookup does not always include it, although PAM
commonly links it into a login's session keyring.
[user-keyring(7)](https://man7.org/linux/man-pages/man7/user-keyring.7.html)

The **key type** `user` and the **user keyring** are separate concepts.
`user` describes a payload that userspace can read; `@u` selects where its
reference is stored. A key of type `user` can also be stored in `@s`.

Sharing a ring does not override the permissions of the keys inside it.
For access based on your UID, give the key owner the necessary rights and
remove possessor, group, and other grants. With ordinary process credentials,
the owner check matches your UID; more precisely it uses the caller's
filesystem UID. The kernel already knows this identity, so retrieving the
key needs no additional application password. This authorizes access using
an existing OS identity; it does not authenticate a remote party.

Here is a disposable example using the `keyutils` CLI. Run it as your normal
user. It prints demonstration data, never a real credential:

```bash
user_demo_id=$(printf '%s' 'example-only' | keyctl padd user plumbers:uid-demo @u)
keyctl setperm "$user_demo_id" 0x003f0000

# This also works in another ordinary shell under the same UID.
keyctl pipe "$(keyctl search @u user plumbers:uid-demo)"
printf '\n'

# Remove this demonstration key when finished.
keyctl revoke "$user_demo_id"
keyctl unlink "$user_demo_id" @u
```

`0x003f0000` grants all six rights to the owner only, including permission
to manage and revoke the example key. Other permission classes grant
nothing. This deliberately trusts all programs running as you; it does
not distinguish a trusted editor from an untrusted script under your
account. Avoid changing permissions on the entire shared `@u` ring merely
to configure one application's key.
[keyctl_setperm(3)](https://man7.org/linux/man-pages/man3/keyctl_setperm.3.html)

| Anchor | Intended sharing |
| --- | --- |
| `@u`: user keyring | Programs with the same UID, subject to key permissions |
| `@s`: session keyring | Processes sharing an inherited or explicitly joined subscription |
| `@us`: user session keyring | Per-UID fallback when a process lacks its own session keyring |

`@us` is distinct from `@u`; it provides a default session anchor rather
than another name for the user keyring.
[user-session-keyring(7)](https://man7.org/linux/man-pages/man7/user-session-keyring.7.html)

The user keyring is held only in kernel memory; it is not saved to disk.
Rebooting loses it. Closing one terminal usually leaves it available to
other running processes under your account. After the keyring is gone,
applications must load their keys again to decrypt existing data. Root or
an administrator controlling the host is outside this protection.


## Try the inheritance with a disposable value

On Ubuntu or Debian, install the `keyutils` package for the `keyctl` CLI.
Use a disposable value for this exercise; the final command prints it.

```bash
# Start a child shell with a fresh anonymous session keyring.
keyctl session - bash --noprofile --norc

# These commands now run inside that shell.
keyctl setperm @s 0x3f000000
demo_key_id=$(printf '%s' 'example-only' | keyctl padd user demo:token @s)
keyctl setperm "$demo_key_id" 0x0b000000

# A further child can find and read the key.
bash --noprofile --norc -c 'keyctl pipe "$(keyctl search @s user demo:token)"'
printf '\n'

exit
```

`@s` selects the caller's session keyring. The ring's `0x3f000000` grants
all six rights to possessors only; this is convenient for the exercise,
not a final application policy. The key's `0x0b000000` grants possessors
view, read, and search, with no owner/group/other rights. `padd` reads the
payload from stdin rather than a command argument.
[keyctl(1)](https://man7.org/linux/man-pages/man1/keyctl.1.html)

From an unrelated shell, searching that shell's `@s` will not find this key.
To test permissions rather than merely a different search path, try
`keyctl pipe KEY_ID` there using the numeric ID printed or retained in the
first shell. With the permissions above and no shared subscription, direct
reading should fail too. These tests establish keyring access behavior;
they do not establish isolation from process debugging or application control.

## Calling it from Python

The CLI is optional. Python can use `ctypes` with `libkeyutils.so.1`, supplied
by `libkeyutils1` on Debian/Ubuntu. The relevant sequence is:

```text
keyctl_join_session_keyring(NULL)     create and join an anonymous keyring
keyctl_setperm(ring, permissions)    restrict access before adding secrets
add_key("user", name, bytes, ..., ring)
keyctl_setperm(key, permissions)     restrict the payload's access
execve(server, argv, environment)   replace launcher; retain subscription
```

`NULL` requests a fresh anonymous ring. In a real binding, declare each
function's argument and return types and check errors using `errno`;
default `ctypes` conversions are insufficient for pointers and sizes.
[keyctl_join_session_keyring(3)](https://man7.org/linux/man-pages/man3/keyctl_join_session_keyring.3.html)

Readers search the current session keyring and call `keyctl_read` into a
buffer of the appropriate size. A read copies the payload into application
memory: it is no longer protected solely by kernel key permissions.
Python also does not promise reliable erasure of all copies of a `bytes`
object. [keyctl_read(3)](https://man7.org/linux/man-pages/man3/keyctl_read.3.html)

## Key storage and encryption are separate operations

A kernel keyring does not automatically encrypt files. An application reads
an authorized key and uses a cryptographic library to encrypt or decrypt
its data. The ciphertext can live on disk while the key is retained only
for the application's lifetime.

Use **authenticated encryption** so modifications to ciphertext are detected
as well as its contents concealed. Follow the library's nonce requirements;
for algorithms that require unique nonces, reusing a nonce with the same key
can defeat security. A human password also needs an appropriate password
key derivation function before use as an encryption key.

For structured data such as JSON, encrypting the complete document hides
field names and values. Encrypting individual values permits independent
updates but exposes the structure. Associate each ciphertext with its field
name using authenticated associated data when the format supports it, so
entries cannot be silently swapped. These are application format choices;
the kernel keyring stores the key used by that format.

## Lifetime and limits

A keyring's lifetime follows references, not the terminal window that
started the application. Children can retain access after the launcher
exits. Keys can expire or be revoked, and unreferenced objects are garbage
collected. Additional links can extend retention.
[keyrings(7)](https://man7.org/linux/man-pages/man7/keyrings.7.html)

Revoking a key prevents subsequent reads; it cannot erase copies already
retrieved by an application. Losing the only decryption key also makes
stored ciphertext unusable. Recovery and unlocking belong in the storage
design.

Keyring permissions govern the key API. They do not prevent an authorized
reader from copying a key or sharing decrypted data. They also do not
replace protection against process inspection: Linux's ptrace checks and
Yama policy determine whether another process can inspect application
memory. Host administrator access remains outside this protection.
[Yama documentation](https://docs.kernel.org/admin-guide/LSM/Yama.html)

## Availability and neighboring mechanisms

Kernel key support depends on `CONFIG_KEYS`; it is not guaranteed by the
word “Linux” alone. A sandbox can also deny the relevant system calls.
Probe operations on the actual host and fail clearly when unavailable.
[Kernel key retention service](https://docs.kernel.org/security/keys/core.html)

Other primitives answer different questions. [Namespaces](../containers/index.md)
control a process's view of resources. Landlock lets applications restrict
their own access, with restrictions inherited by descendants; it does not
serve as a credential store. These mechanisms can complement a keyring,
provided the application defines which processes it trusts and which
resources each process should reach.
[Landlock documentation](https://docs.kernel.org/userspace-api/landlock.html)
