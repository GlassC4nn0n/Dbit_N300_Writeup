# Finding: Static Root Password Hash Deployed at Every Boot

**CWE:** CWE-798 (Use of Hard-coded Credentials), CWE-256 (Plaintext Storage
of a Password, N/A here since hashed — noted for completeness only)
**Location:** `etc/shadow.sample`, `etc/passwd`, `etc/passwd_orig`

## Summary

The firmware ships a non-empty, non-locked root password hash in
`etc/shadow.sample`, which is unconditionally copied to `/var/shadow` on
every boot (`cp /etc/shadow.sample /var/shadow` in `rcS`/`rcS_32M`). A
second, differently-salted root hash exists in `etc/passwd_orig`, suggesting
a build-time pipeline that sets/rotates this value per firmware build or
customer SKU rather than a single hash reused verbatim across all vendor
products. This is the credential that would authenticate any login —
telnet, console, or otherwise — against every device running this firmware
image, unless something later in the boot process overrides it.

## Discovery

```bash
cat squashfs-root/etc/shadow.sample
```
```
root:$1$KE...0$TFJ...GzV.:14587:0:99999:7:::
nobody:*:14495:0:99999:7:::
```

```bash
cat squashfs-root/etc/passwd_orig
```
```
root:$1$o...6$7uB...Kqy.:0:0:root:/:/bin/sh
nobody:x:0:0:nobody:/:/dev/null
```

## Analysis

Both hashes use `$1$` (MD5crypt) format: `$1$salt$hash`.

| File | Live at runtime? |
|---|---|
| `shadow.sample` | **Yes** — copied to `/var/shadow` unconditionally on every boot |
| `passwd_orig` | Not directly — appears to be a build-time template/backup, not referenced by any boot script found so far |

The two files use different salts, so the ciphertexts differ regardless of
whether the underlying plaintext password is the same. This points to a
provisioning/build pipeline that generates or modifies the root hash at some
stage, rather than one static hash hardcoded identically across the entire
codebase.

**The hash that matters for real-world impact is the one in
`shadow.sample`**, since that is confirmed live at every boot via the boot
script chain.

## Impact

Contingent on cracking success:

- If cracked, this directly determines the real-world exploitability of the
  [conditional telnet access finding](./02-conditional-telnet-access.md):
  telnet's own exposure is conditional and unconfirmed, but *if* it or any
  other login path is reachable, this password is what an attacker would
  need to actually get in.

## Recommendation

- Do not ship a static, shared root password hash in firmware images.
- Require a mandatory password change/randomized initial credential at
  first boot or during manufacturing provisioning.
- Lock the root account (`!` or `*` in the hash field) by default when no
  interactive login is required for normal operation.
- Migrate away from MD5crypt (`$1$`) to a modern KDF (e.g., `$6$`/SHA-512
  crypt, or better, bcrypt/argon2 where the platform supports it).

## Open Questions

- [ ] If cracked, cross-reference against the telnet-access finding to state
      a concrete, combined attack scenario rather than two separate findings
- [ ] Check whether `passwd_orig`'s hash (if cracked to a different
      plaintext) represents an older/superseded default vs. current
