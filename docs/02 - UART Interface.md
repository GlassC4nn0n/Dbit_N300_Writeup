The UART interface on the top of the board is the easiest place to start. 


<img width="3072" height="4080" alt="Flash Chip   UART" src="https://github.com/user-attachments/assets/e9efc59c-3f6c-43db-af4e-f8804ad8c28e" />


There are two main approaches when determining the baud rate: Use a signal analyzer and determine the exact baud rate, or guess common baud rates. I will walk through both.

**Calculating Baud Rates**

1. Hook up a logic analyzer to the UART interface, start your analysis software, and turn on the target device
2. Stop recording signals, go back to the start of the recording and measure the length of the signal. In our case, this is 26 microseconds
3. Do the following calculation: `1/26 x 1000` which equals `38.461`
4. This is very close to the common baud rate of `38400`, so we will use that when reading from UART

**Guessing Baud Rates**

Embedded systems usually use similar baud rates across different devices. If you don't have the patience for logic analysis, you can simply choose between these common baud rates and will likely find the correct one:

4800, 9600, 19200, 38400, 57600, 115200, 230400, 460800, and 921600
_______

**Connecting via UART**

We use the following command to open a terminal on the target

`sudo picocom -b 38400 /dev/ttyACM0 --logfile N300.log`

The terminal is, unfortunately, non-interactive. However, the bootlog does give us some useful information about the system.
_________

# Findings
_The full bootlog has been added to this directory for further analysis_

**Bootloader**

```
RealTek(RTL8196E)at 2025.08.12-13:39+0800 v3.4.11E [16bit](380MHz)
```

**Linux Version**

```
Linux version 3.10.90 (jenkins@muserver) (gcc version 4.6.4 (Realtek RSDK-4.6.4 Build 2080) ) #en32 Tue Aug 12 13:42:09 CST 2025
```
**Filesystem Structure**

```
squashfs: version 4.0 (2009/01/31) Phillip Lougher
```

**Partitions**

```
Creating 4 MTD partitions on "flash_bank_1":  
0x000000000000-0x000000210000 : "boot+cfg+linux"  
0x000000210000-0x0000003c0000 : "rootfs"  
0x0000003c0000-0x0000003e0000 : "cwmp transfer"  
0x0000003e0000-0x000000400000 : "cwmp notification"
```

**Boa Webserver**

```
boa: server version Boa  
boa: server built Aug 1type:3, enable:0, percent0 
```
