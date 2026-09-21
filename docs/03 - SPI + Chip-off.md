# Extracting the Firmware from SPI

While attempting to extract the firmware, I ran into some issues that hopefully you can avoid.

A simple chip clip and CH341a programmer was unable to interact with the flash in any meaningful way. This was likely due to the way the chip's power rail is set up. This means that soldering the chip off the board is the next reasonable step towards firmware extraction.

Using a SOP8 to DIP8 chip adapter, we can completely isolate the chip and pull the firmware off with `IMSProg`

<img width="3072" height="4080" alt="SPI Adapter" src="https://github.com/user-attachments/assets/3f8a025c-22a6-4884-a97f-ab67cd014e1d" />

**IMSProg Settings**

<img width="978" height="683" alt="IMSProg_N300" src="https://github.com/user-attachments/assets/26ba69a1-1703-42e5-8691-817292de034b" />

We save the extracted firmware to `N300_firmware.bin`
____

**Binwalk**

A quick check with binwalk shows us that the firmware is successfully recognized

<img width="1167" height="548" alt="binwalk_id_N300" src="https://github.com/user-attachments/assets/8a233d5b-a611-41eb-9171-94d241f3495e" />
