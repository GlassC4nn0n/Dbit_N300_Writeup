# Finding: Hardcoded, Unencrypted Private Key / Self-Signed Certificate Pair

**CWE:** CWE-321 (Use of Hard-coded Cryptographic Key), CWE-798 (Use of Hard-coded Credentials)
**Location:** `/etc/privateKey.key`, `/etc/certificate.crt`

## Summary

The firmware ships an unencrypted RSA-2048 private key and a matching
self-signed certificate, both stored in plaintext under `/etc`. The
certificate's subject (`CN=192.168.1.254`, a default gateway address) is
consistent with a locally-generated HTTPS management-interface certificate,
though **no runtime consumer of these specific files was identified during
static analysis**. This finding is reported as confirmed material with a
plausible but unproven purpose.

## Discovery

Located via extension search of the extracted rootfs:

```bash
find squashfs-root/etc -iname "*.crt" -o -iname "*.key" -o -iname "*.pem" -o -iname "*.cer"
```

<img width="1445" height="73" alt="Crypto_Locations" src="https://github.com/user-attachments/assets/16679cea-7db4-4cbb-ad6e-d4e41b557b4e" />

```
$ cat privateKey.key 
-----BEGIN PRIVATE KEY-----
MIIEvAIBA
<SNIP>
...
YKlQ0kxMhBAp6G7HPCWFtA==
-----END PRIVATE KEY-----
```

```
$ cat certificate.crt 
-----BEGIN CERTIFICATE-----
MIID1TC
<SNIP>
...
hnFIu6MSyV9JKlt6TU02AlCjEVvHRVhzPA==
-----END CERTIFICATE-----

```

## Analysis

### Certificate fields

```bash
openssl x509 -in certificate.crt -noout -text
```

| Field | Value |
|---|---|
| Subject / Issuer | Identical (self-signed) |
| CN | `192.168.1.254` |
| Subject email | `<CENSORED>@163.com` |
| O / OU | `RS` / `WN` |
| Location | C=CN, ST=JS (Jiangsu), L=SZ (Suzhou) |
| Validity | 2013-12-19 → 2014-12-19 (expired) |
| Basic Constraints | `CA:TRUE` |
| Signature algorithm | SHA1withRSA |
| Key size | RSA-2048 |

### Key/certificate pairing confirmed

Verified the private key and certificate share the same RSA modulus,
confirming they are a genuine matching pair:

```bash
openssl rsa -in privateKey.key -noout -modulus | openssl md5
# MD5(stdin)= 7310d7e2fa7116c2279146568674b674

openssl x509 -in certificate.crt -noout -modulus | openssl md5
# MD5(stdin)= 7310d7e2fa7116c2279146568674b674
```

Modulus MD5 hashes match.

### Key is unencrypted

No passphrase or `Proc-Type: 4,ENCRYPTED` header present — any party with
access to the firmware image has immediate access to the raw private key.

## Attempted Consumer Identification

Static analysis was used to try to identify which process, if any, loads
these files at runtime. All of the following came back negative:

- Recursive filename search (colon and plain grep, `--binary-files=text`) — no matches
- Search for common TLS config directives (`SSLCertificate`, `cert_file`,
  `SSL_CTX_use`, etc.) — no matches
- Known cert directory conventions (`/usr/local/ssl`, `/etc/ssl`) — empty
- Symlinks pointing at either file — none found
- Exhaustive `strings` sweep across every executable in the filesystem — no matches
- `boa` (primary web server referenced in `rcS`) — confirmed via an explicit
  compiled-in string (`"CPE does not support SSL! URL should not start with
  'https://'!"`) that this binary **does not support SSL at all**
- `webs` (web server referenced in `rcS_32M`) — binary not present in this
  extracted image
- `cwmpClient` (TR-069 client) — does implement a TLS client-certificate
  mechanism, but references different filenames (`/etc/client.pem`,
  `/etc/cacert.pem`, not the files in question), and is compiled without
  OpenSSL support (`"OpenSSL not installed: recompile with -DWITH_OPENSSL"`,
  no crypto library linked per `rabin2 -l`), making that code path inert
  regardless
- `sysconf` (primary config binary invoked via `init.sh gw all`) — no
  cert/key-related strings found

**Conclusion of this thread:** no consumer was identified through static
analysis. This does not mean the files are unused — it means definitive
attribution requires dynamic analysis (see below).

## Impact

If this key pair is used as hypothesized (a local HTTPS management
interface certificate):

- The key is very likely **identical across every device shipped with this
  firmware image**, since there is no per-device generation step visible in
  the boot scripts.
- Anyone with the public firmware image possesses the private key, enabling
  passive decryption of captured HTTPS traffic to the device's web UI, or
  active MITM without a browser certificate warning (since it is the
  device's own legitimate certificate).

If the key pair is unused/orphaned SDK cruft, the practical runtime impact
is lower, but its presence still reflects poor secrets hygiene in the build
process — an unencrypted private key should not ship in a production
filesystem image regardless of whether it is currently wired up to a
service.

## Recommendation

- Do not ship static, shared private keys in firmware images.
- Generate unique key material per device at first boot or during
  manufacturing provisioning.
- Remove unused cryptographic material from production build images.

## Status / Next Steps

- [x] Key and certificate located
- [x] Key/certificate pairing confirmed
- [ ] Runtime consumer confirmed — **unresolved via static analysis**
- [ ] Dynamic analysis (emulation + `strace`/`inotify` on a running
      instance) to observe which process, if any, opens these files
- [ ] NVRAM/MIB variable audit (`apmib`) for a config value pointing at
      either path, in case the path is assembled at runtime rather than
      hardcoded
- [ ] Check whether this same key/cert (by modulus or serial number) appears
      in other firmware images from the same vendor/ODM, or shows up in
      Shodan/Censys TLS scan data, to establish scope of real-world exposure
