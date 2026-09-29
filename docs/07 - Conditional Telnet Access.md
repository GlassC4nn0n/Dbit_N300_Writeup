# Finding: Conditional Debug Telnet Access

**CWE:** CWE-489 (Active Debug Code), CWE-1188 (Insecure Default Initialization of Resource)
**Location:** `etc/init.d/rcS`, `etc/init.d/rcS_32M`

## Summary

Both boot scripts contain logic that will start an unauthenticated-at-launch
`telnetd` service on a non-standard port (2356) under two independent
conditions: an NVRAM/MIB flag not being explicitly set to disable it, and a
hardcoded default MAC address match. **The actual real-world trigger
conditions for either path were not confirmed during static analysis** — this
finding documents reachable code, not a proven live vulnerability.

## Discovery

Located while reviewing the boot script chain (`rcS` / `rcS_32M`):

```sh
# enable telnet hw setting to start telnet
if [ "$(flash get HW_TELNETD_ENABLE | awk -F '=' '{print $2}')" != "0" ]; then
    if [ "$(ps | grep telnetd | grep -v grep)" == "" ]; then
        echo "start telnetd because of HW_TELNETD_ENABLE"
        /bin/telnetd -p 2356
    fi
fi

# check whether the mac address is the default value
# if so, start telnetd
if [ "$(macaddr | awk '{print toupper($1)}')" == "00:E0:4C:81:96:C9" ]; then
    if [ "$(ps | grep telnetd | grep -v grep)" == "" ]; then
        echo "start telnetd because of default mac addr"
        /bin/telnetd -p 2356
    else
        echo "found telnetd, no need start twice"
    fi
fi
```
Identical in both `rcS` and `rcS_32M`.

## Analysis

### Trigger A — MIB flag `HW_TELNETD_ENABLE`

Telnet starts unless this flash-stored variable is explicitly `"0"`. Written
naively, this is an **opt-out** design (telnet enabled by default unless
something disables it), rather than opt-in. However:

- The actual **default value** of `HW_TELNETD_ENABLE` as shipped on
  production units was not located during static analysis (searched for the
  variable across `flash`, `sysconf`, and the rest of the filesystem — no
  default-value definition found).
- It is possible this value is set to `"0"` during manufacturing/provisioning
  and only reverts to an enabling state under specific fault conditions
  (e.g., the config-reset-to-defaults behavior in `startup.sh` — see the
  boot-scripts doc). This was not confirmed either way.

### Trigger B — hardcoded default MAC address

```
00:E0:4C:81:96:C9
```
The OUI prefix (`00:E0:4C`) is a registered Realtek Semiconductor OUI,
confirmed present as boilerplate elsewhere in the filesystem (`udhcpd`,
`cwmpClient` both contain the bare OUI `00E04C`, though not the full MAC —
ruled out as a second hardcoded-identity instance). The full MAC
`00:E0:4C:81:96:C9` was found **only** in `rcS` / `rcS_32M`.

This condition most plausibly exists as a manufacturing/QA safety net —
auto-enabling debug access if a unit reaches a boot stage with an
unprogrammed/default MAC still set, which would itself indicate a
provisioning failure — rather than a condition expected to be true on
correctly-manufactured, shipped consumer units.

### Port and authentication

`telnetd -p 2356` runs on a non-standard port with no flags restricting
authentication. Root login would authenticate against whatever is present in
`/var/shadow` at boot time — see the separate default-credentials finding —
meaning if either trigger fires, the resulting telnet session's security is
only as strong as that static password hash.

## Caveats (Important)

This finding should **not** be represented as a confirmed, universally
exploitable backdoor. Specifically unconfirmed:

1. The real default value of `HW_TELNETD_ENABLE` on production firmware
2. Whether any shipped unit would ever actually present the hardcoded MAC
3. Whether `telnetd` genuinely binds and accepts connections when either
   condition is met, in a real or emulated running instance (dynamic
   verification was not completed — FirmAE emulation attempts were blocked
   by environment/dependency issues; EMBA's emulation profile was the
   planned follow-up)

## Impact (Conditional)

If Trigger A's default is confirmed to be enabling, or if manufacturing
provisioning fails and Trigger B's condition is met: unauthenticated network
exposure of a root-capable shell on port 2356, subject only to whatever
credential is present in `/var/shadow`.

## Recommendation

- Telnet debug services should not be reachable in production firmware
  builds under any default condition.
- If retained for manufacturing/QA purposes, gate behind a build-time flag
  stripped from production images, not a runtime MIB/config check.
- Remove hardcoded MAC-based conditionals entirely; a fallback keyed to a
  known "unprovisioned" identity is itself an information-disclosure risk
  once the firmware is public.

## Status / Next Steps

- [x] Conditional telnet-enable logic located and both trigger paths mapped
- [x] Hardcoded MAC confirmed unique to this logic (not duplicated elsewhere
      as a full identity)
- [ ] Confirm actual default value of `HW_TELNETD_ENABLE` (static: locate
      definition in `sysconf`/`flash`/default MIB table; or dynamic: query
      live/emulated instance)
- [ ] Confirm via emulation whether `telnetd` actually starts and accepts
      connections under a fresh/default boot
- [ ] Cross-reference with default-credentials finding once shadow hash is
      cracked, to assess realistic end-to-end impact if either trigger fires
