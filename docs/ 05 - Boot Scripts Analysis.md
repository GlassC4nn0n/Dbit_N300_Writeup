# Boot Scripts Analysis

**Location:** `etc/init.d/rcS`, `etc/init.d/rcS_32M`, `bin/startup.sh`, `bin/init.sh`, `bin/post_startup.sh`

## Summary

Traced the full userspace boot sequence from first init script through to
web server startup, mapping the device's runtime environment, service
initialization, and identifying two notable conditional behaviors around
telnet access.

## Boot Chain Overview


## `rcS` vs `rcS_32M` — Diff

| | `rcS` | `rcS_32M` |
|---|---|---|
| Web asset extraction | `#flash extr /web` (commented out) | `flash extr /web` (active) |
| Web server | `boa` (foreground) | `webs &` (backgrounded) |
| Everything else | identical | identical |

`_32M` likely corresponds to a 32MB-flash SKU where web UI assets
are stored in a separate flash region and unpacked at boot, vs. baked into
the rootfs on other variants. **Note:** `webs` binary was not found in this
extracted image.

## Key Behaviors Identified

### 1. Deterministic reset-to-defaults on config corruption (`startup.sh`)
`startup.sh` validates several flash config sections (`test-hwconf`,
`test-bluetoothhwconf`, `test-customerhwconf`, `test-dsconf`, `test-csconf`)
and silently invokes `flash default*` to reset any that fail validation.
No user notification, no fail-safe halt — flagged as relevant context for
[telnet backdoor finding], since a corrupted config could possibly
force a device back into default state.

### 2. WPS PIN generation (`startup.sh`)
```sh
eval `$GETMIB HW_WSC_PIN`
if [ "$HW_WSC_PIN" = "" ]; then
    $TOOL gen-pin
fi
```
Generates a WPS PIN via `flash gen-pin` if unset. **Algorithm not yet
reversed** — flagged as a follow-up (see [Open Questions](#open-questions)).
WPS PIN predictability is a well-documented vulnerability class across
embedded routers generally.

### 3. `init.sh` is a pass-through wrapper
```sh
sysconf init $*
```
The entire file. All actual network/config logic for `init.sh gw all`
lives in the compiled `sysconf` binary, not in any shell script — meaning
static script review alone cannot fully characterize this stage; binary
reversing of `sysconf` is required. **Not yet completed.**

### 4. Conditional telnetd (rcS / rcS_32M)
Two independent triggers for `telnetd -p 2356`:
```sh
# Trigger A: MIB flag not explicitly disabled
if [ "$(flash get HW_TELNETD_ENABLE | awk -F '=' '{print $2}')" != "0" ]; then ...

# Trigger B: hardcoded default MAC match
if [ "$(macaddr | awk '{print toupper($1)}')" == "00:E0:4C:81:96:C9" ]; then ...
```
See dedicated finding doc: [`0X-telnet-access.md`](0X-telnet-access.md).
**Important:** default value of `HW_TELNETD_ENABLE` was not confirmed via
static analysis — this finding should not be overstated as a guaranteed
live backdoor. See that doc's caveats section.

## Open Questions
- [ ] Locate/reverse `sysconf` — resolves `init.sh gw all` logic, potentially
      the TLS key/cert consumer question, and MAC/config handling
- [ ] Reverse `flash gen-pin` — is the WPS PIN MAC-derived/predictable?
- [ ] Confirm default value of `HW_TELNETD_ENABLE`
- [ ] Locate `webs` binary or confirm this firmware variant doesn't use
      `rcS_32M` in practice
- [ ] Carve/extract the `flash extr /web` region if `rcS_32M` is relevant
