# DBIT N300 Router Firmware Security Analysis 

## Summary

Firmware pulled from the N300 via chip-off extraction has been statically analyzed, revealing three significant findings including a hardcoded/expired private key pair, static default credentials, and conditionally-enabled debug telnet access.

## Device & Firmware Profile

- **Vendor/Model:** DBIT N300 
- **Firmware file:** `N300_firm.bin`
- **Base platform:** Realtek SDK-derived embedded Linux (`rlx-linux`)
- **Architecture:** MIPS/ARM
- **Extraction method:** Chip-off → IMSProg → FAT + unblob → SquashFS

## Methodology

Full walkthrough available in [`docs/`](docs/). Brief overview:

1. UART interface reconnaissance 
2. Pull device firmware via chip-off with IMSProg
3. Firmware extraction (FAT/unblob → SquashFS)
4. Boot chain analysis (`rcS`, `rcS_32M`, `startup.sh`, `init.sh`, `post_startup.sh`)
5. Credential/key material discovery (`/etc/`)
6. Static binary analysis (`rabin2`, `strings`) on network/config binaries
7. Dynamic analysis (EMBA)

**Tools used:** FAT, `openssl`, `strings`, `grep`, hashcat, EMBA

## Disclaimer

This research was conducted for educational purposes. Do not use this information against devices you do not
own or have explicit authorization to test.
