---
layout: post
author: marcin
---

# IBM Thinkpad T30 Bios Password Reset

## Intro and Hardware

A friend of mine recently came upon an old IBM T30 Thinkpad at an auction for $40.  Bringing it home, he found that there was a power-on supervisor password.  This can't be reset by removing the battery, as the pswd is stored on an EEPROM on the motherboard..... So it came to me!

### The Buspirate

I recently purchased a [buspirate v3 from Seeedstudio](http://www.seeedstudio.com/depot/bus-pirate-v3-assembled-p-609.html?cPath=174&zenid=032aa3caf779c7cde4dc39b8ccffef65), (I forgot to buy the [Probes](http://www.seeedstudio.com/depot/bus-pirate-probe-kit-p-526.html?cPath=178_180&zenid=032aa3caf779c7cde4dc39b8ccffef65) though...) and decided to give it a quick test run...

[![BusPirate](/images/t30_bios_bus_pirate.jpg)](/images/t30_bios_bus_pirate.jpg)
 
A Buspirate is a USB based "Universal Electronic hacking tool to talk to electronic stuff" from the cool guys at [Dangerous Prototypes](http://dangerousprototypes.com/docs/Bus_Pirate). It does I2C, JTAG (slowly), SPI, 1-WIRE, UART, and any other protocol you'd like to bitbang.  It's completely scriptable from Python or Perl, or any other language you can get interface with a comport with.  It's also directly accessible from a Terminal client. Cool Beans!

### The Laptop

Looking up "T30 bios password reset" on google pointed me towards an eeprom chip located somewhere on the motherboard. In the case of the T30, it was under the RAM slot. (Click for larger image)

[![EEProm](/images/t30_bios_eeprom1.jpg)](/images/t30_bios_eeprom1.jpg) | [![EEProm Zoom](/images/t30_bios_eeprom2.jpg)](/images/t30_bios_eeprom2.jpg) | [![EEProm Zoom Closer](/images/t30_bios_eeprom3.jpg)](/images/t30_bios_eeprom3.jpg)

Upon closer inspection, it turns out to be an ATMEL AT24RF08C, funnily enough, the same eeprom used in the [SCOPE UPGRADE]({% post_url 2011-06-12-scope-upgrade %}) project.
Looking up the datasheet [online](http://www.datasheetarchive.com/AT24RF08C*-datasheet.html), we can bring out the pinout and addressing modes for this EEProm chip.

### Pinout

[![EEProm](/images/t30_bios_eeprom_pinout.png)](/images/t30_bios_eeprom_pinout.png) 

### Data Addressing

The AT24RF08 series features 1k bits of memory, organized into 8 x 128 bit (16 byte) blocks. 

The blocks are addressed with 3 bits, B2 B1 B0, followed by A1 A0, which are used to address the individual words. (A2 is held internally high).  The addressable space thus runs from 0x00->0x7F in each block.

In order to do a read operation over the i2c bus, some slight bit manipulation is involved, the AT24RF08 doesn't simply take a 0xDEVICEADDRESS 0XMEMORYADDRESS.  

The 0xDEVICEADDRESS actually contains part of the memory address you want to access including whether you are doing a read or write operation, and specifically the bytes denoting which memory bloc your target address resides on.  Because of this, the AT24RF08C actually responds to addresses 0xA8->0xAF on the i2C bus.
 
Our address of interest is 0x338 (where the password starts), or the 824'th bit.    To see which block it will be in we simply divide 824/128 =  6.43..
so we're in the 7th addressing block.  B2 B1 B0 = 1 1 0 .  

Our word address is simply 0x38.

Now, to READ the memory address at location 0x338, we first need to WRITE that address into the device.  Thus, our first operation is a WRITE. 

Datasheet | 1 | 0 | 1 | 0 | 1 | B2 | B1 | W*/R
----------|---|---|---|---|---|----|----|-----
We want	  | 1 | 0 | 1 | 0 | 1 | 1 | 1 | 0

```
RESULTING DEVICE ADDRESS -> 0xAE
RESULTING MEMORY ADDRESS -> 0x38
```

> NOTE: 
> In order for us to not overwrite the eeprom @ that location with some data, we need to send a STOPBIT over the bus before continuing on.

```
First i2C operation =>  STARTBIT 0xAE 0x38 STOPBIT
```

To READ the information out now, all we do is put the DEVICE ADDRESS onto the i2c bus, and let the clock run as many times as we need bytes.  The EEPROM auto-increments the memory address in the read operation as long as the clock runs, until a STOPBIT is received.

Datasheet | 1 | 0 | 1 | 0 | 1| X | X | W*/R
----------|---|---|---|---|---|----|----|-----
We want   | 1 | 0 | 1 | 0 | 1 |	1 | 1 | 1

> "X" denotes dont-cares. I chose 1's for whatever arbitrary reason I gave myself at the time.

```
RESULTING DEVICE ADDRESS -> 0xAF
```

```
Second i2C operation => STARTBIT 0xAF *toggle clock* STOPBIT
```

Thus, our COMPLETE I2C request will look like the following:

```
STARTBIT 0xAE 0x38 STOPBIT STARTBIT 0xAF *toggle clock* STOPBIT
```

# Hooking up and setting up the Hardware

1. First step is to power on the laptop (I used batter power as not to get any funny grounding issues). 
2. DO NOT plug the buspirate in at this point.
3. Wait until the password prompt appears, and then wait around 60 seconds for everything to settle down.  Connect the Buspirate according to the following diagram, starting with the GND pins. 

   > NOTE: On the buspirate, SCL is denoted by the CLK pin and SDA is denoted by the MOSI pin.

   [![How to connect your BusPirate](/images/t30_bios_bus_pirate_connnections.jpg)](/images/t30_bios_bus_pirate_connnections.jpg)

4. Plug in the buspirate and turn on your favourite Terminal Application.  I used MTTTY. My buspirate was being enumrated onto COM port 8, so I had to run MTTTY as administrator for it to be able to connect to a COMport past 4.
(CLICK FOR LARGER IMAGE)

# Using the buspirate to read I2C.

## Setting up the buspirate for i2C mode.

1. bring up the terminal

   [![Terminal](/images/t30_bios_terminal.jpg)](/images/t30_bios_terminal.jpg)

2. press 4 to get into I2C mode, then select (3) to select 100KHz mode.
3. You should now see READY on the terminal. See above picture for details.
4. We want to bring up the power supply first, so we type in a capital W, the buspirate should respond with

   ```
   Power Supplies ON.
   ```

5. Now we bring the pull-up resistors on with a capital P, the buspirate should respond with

   ```
   Pull-up resistors ON
   ```

   If anything else shows up, ex, Short detected, etc, check your wiring to ensure proper contacts are made, and that you hooked the buspirate up to the correct pins.

6. Now, we write the following:

   ```
  [0xAE 0x38][0xAF r:16]
   ```

   What does this do?
   The [ ] denote start and stop bits, and the r:16 tells the buspirate to toggle the clock 16 times. 

   The output should look something like the following (click to enlarge)

   [![Terminal Output](/images/t30_bios_terminal_after.jpg)](/images/t30_bios_terminal_after.jpg)

# Decyphering the Data - Keyboard Scancodes

Now that we have the data off the eeprom, we need to find out what it's encoded as.  Lucky for us, the password on the T30 is just stored as the correpsonding scancode, with no encryption. ( [See This Page for details](http://www.barcodeman.com/altek/mule/scandoc.php) ).  In my case, if you look at the above dump you see the following 

bits:  0x20 0x18 0x26 0x26 0x00 0x00  
The 0x00 can be thrown away as they are just padding.

Scancode(hex) | 0x20 | 0x18 | 0x26 | 0x26
--------------|------|------|------|-----
key number    | 33   | 25   | 39   | 39
102key equiv  | r    | w    | p    | p
84key equiv   | d    | o    | l    | l
  
I took the scancode hex dump from the eeprom, and cross referenced it with the corresponding key number (link above).  Then, looking at the 102 and 84 key keyboards, I came up with two possibilities... "rwpp" and "doll".

> **NOTE** This depends on the keyboard you have. If you are using an international (french/german/etc) your scancode->key lookups WILL DIFFER.

Trying the second password first...

[![Trying "doll"](/images/t30_bios_try_password.jpg)](/images/t30_bios_try_password.jpg)

**We have Bingo!**

[![Great Success!](/images/t30_bios_password_success.jpg)](/images/t30_bios_password_success.jpg)

We now have full access to the BIOS and can remove/change the power on password and supervisor password. A winner is you!

# Future Work

Funnily enough, now that I could get past POST, it actually turns out that this laptop also has the 188 error:  `BAD CRC2 CHECKSUM ERROR`.  I'm going to write up the fix for that later on as a continuation of this project.
