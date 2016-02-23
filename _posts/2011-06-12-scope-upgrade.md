---
layout: post
author: andrew
---

# Scope Upgrade

This page will demonstrate how it is possible to enable certain software upgrades on certain digital oscilloscopes by crafting your own hardware key. The hardware key consists of nothing more than an i2c eeprom device and a sim card connector. The information found here is based on [http://forum.tsebi.com/viewtopic.php?f=4&t=113](http://forum.tsebi.com/viewtopic.php?f=4&t=113) (Tektronix DPO 2024 hardware feature module memory upgrade).

## Parts

- 1 sim card connector - Ex: digikey [478-4366-1-ND](http://www.digikey.com/product-detail/en/009162006206175/478-4366-1-ND/1551089)
- 1 24c08 eeprom - Ex. digikey [AT24C08D-SSHM-TCT-ND](http://www.digikey.com/product-detail/en/AT24C08D-SSHM-T/AT24C08D-SSHM-TCT-ND/4743514) (other sizes will work too, the instructions here are relevant for the 24c08)
- Protoboard
- Wirewrap wire

## Schematic

![Schematic](/images/scope_upgrade_schematic.jpg)

## Instructions

1. Remove extraneous plastic from sim card connector, leaving only the metal contacts.
2. Wire up to 24c08 and connector according to attached schematic. Cut board to appropriate size. 
3. Choose two of the following options: DPO2EMBD, DPO2COMP, DPO2AUTO. The hardware should be capable of up to 4, but the forums indicate that the software only uses one bit (A1) for addressing (note: there are only two available ports on the device itself). I have not confirmed myself whether more than 2 modules is possible, largely because I have no use for the DPO2AUTO module.
4. Program the 24c08 chip over i2c (using bus pirate, avr, pic, w/e, just don't forget those pull-up resistors on the i2c bus when programming). Your choice of the NULL TERMINATED strings "DPO2EMBD", "DPO2COMP" and "DPO2AUTO" should start at either memory address "0x004" or "0x204" in the EEPROM (Here the last two bits of the first address byte indicate the values of A1 and A0 being "emulated" by the larger chip).

   For example, I used an avr to program the EEPROM by modifying the code from [http://www.nongnu.org/avr-libc/examples/twitest/twitest.c](http://www.nongnu.org/avr-libc/examples/twitest/twitest.c):

   ```c
   rv = ee24xx_write_bytes(0x004, 9, (uint8_t *)"DPO2COMP");
   rv = ee24xx_write_bytes(0x204, 9, (uint8_t *)"DPO2EMBD");
   ```

5. Remove any existing modules from the slots, or else the new module will cause bus contention (note that because we are using the larger eeprom, one module does the job of two!)
6. Place your new module into the scope and turn it on. You should now have the two software modules you chose unlocked (can be confirmed in the About menu box).
7. ???
8. Profit!

## Pics

![Board](/images/scope_upgrade_board.jpg)

![ScreenShot](/images/scope_upgrade_screenshot.jpg)

