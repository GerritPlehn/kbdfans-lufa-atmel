# KBD67 atmel-dfu bootloader

Replacing the lufa-ms bootloader on kbdfans boards with atmel-dfu 

## Introduction

This article will describe the process I went through to get the atmel-dfu bootloader back on my kbd67 lite r3 ISO hotswap.
I assume the process is identical or very similar to the other kbdfans PCBs that are being shipped with the lufa-ms bootloader, like kbd67 mkiirgb v3/v4 and kbd75rgb.

Why replace the lufa-ms bootloader in the first place? The method of dragging a hex file to the emulated USB drive may be convenient for some, however both for me and many other users it is very unreliable, which also lead the QMK devs to advise kbdfans against using it. Kbdfans however did it anyways, and I'm personally fed up with it. So let's fix it.

Primary resource I used for research: [ISP Flashing Guide](https://docs.qmk.fm/#/isp_flashing_guide) from the QMK docs and a bit of help from the [QMK discord](https://discord.gg/Uq7gcHh).

Of course: if you decide to follow along, you do so at your own risk. It's not rocket-science however!

### Environment

The exact versions of the tools may not be important, however for transparency and repeatability, these versions were used while doing this modification:

- host OS: arch 5.19.4
- avrdude: 7.0
- qmk cli: 1.1.1

### Materials

- [KBD67 MarKII RBG ISO PCB](https://kbdfans.com/collections/pcb/products/kbd67-lite-rgb-iso-pcb)
- Pro Micro
- jumper cables

## Steps

### Preparing the ISP flasher

To replace the bootloader we need to use the serial programming interface on the ATmega32u4. There are a few ways to do this, I chose to go with a Pro Micro as an ISP flasher.

- Download [Pro Micro ISP hex](https://github.com/qmk/qmk_firmware/blob/master/util/pro_micro_ISP_B6_10.hex) 
- connect Pro Micro to host
- make sure the voltage between VCC and GND is 5V. If it is 3.3V, check for a jumper to change the voltage to 5V.
- verify the device is connected

  ```bash
  ls /dev/ttyACM*
  # /dev/ttyACM0
  ```
- put Pro Micro into bootloader mode by shorting `RST` and `GND` twice quickly
- flash the ISP firmware

  ```bash 
  avrdude -p atmega32u4 -P /dev/ttyACM0 -c avr109 -U flash:w:pro_micro_ISP_B6_10.hex
  ```

If everything worked fine, the last few lines should look like this:

```
avrdude: 5756 bytes of flash verified

avrdude done.  Thank you.
```
- unplug the Pro Micro from the host

### Connecting Pro Micro to keyboard


**Attention**: on my unit the pad that has `MOSI` silk-screened next to it is actually connected to pin 5 `UGnd` and not to pin 10 as one would expect. So be aware of that and make sure to verify the pinout with the help of the [32u4 datasheet](https://ww1.microchip.com/downloads/en/DeviceDoc/Atmel-7766-8-bit-AVR-ATmega16U4-32U4_Datasheet.pdf)

- Make sure the keyboard is unplugged!
- Use jumper wires to hook up the Pro Micro according to the [guide section on the qmk docs](https://docs.qmk.fm/#/isp_flashing_guide?id=pro-micro-as-isp).


### Flashing the bootloader

- Download the bootloader hex from the [qmk repo](https://github.com/qmk/qmk_firmware/blob/master/util/bootloader_atmega32u4_1.0.0.hex)
- Plug in the Pro Micro into the host
- verify it is connected correctly as previously
- flash the bootloader

  ```bash
  avrdude -c avrisp -P /dev/ttyACM0 -p atmega32u4 -U flash:w:bootloader_atmega32u4_1.0.0.hex:i -U lfuse:w:0x5E:m -U hfuse:w:0x99:m -U efuse:w:0xF3:m
  ```

That's it! If everything worked fine, your output should look like this:

```
avrdude: AVR device initialized and ready to accept instructions

Reading | ################################################## | 100% 0.01s

avrdude: Device signature = 0x1e9587 (probably m32u4)
avrdude: NOTE: "flash" memory has been specified, an erase cycle will be performed
         To disable this feature, specify the -D option.
avrdude: erasing chip
avrdude: reading input file "bootloader_atmega32u4_1.0.0.hex"
avrdude: writing flash (32768 bytes):

Writing | ################################################## | 100% 0.00s

avrdude: 32768 bytes of flash written
avrdude: verifying flash memory against bootloader_atmega32u4_1.0.0.hex:

Reading | ################################################## | 100% 0.00s

avrdude: 32768 bytes of flash verified
avrdude: reading input file "0x5E"
avrdude: writing lfuse (1 bytes):

Writing | ################################################## | 100% 0.00s

avrdude: 1 bytes of lfuse written
avrdude: verifying lfuse memory against 0x5E:

Reading | ################################################## | 100% 0.00s

avrdude: 1 bytes of lfuse verified
avrdude: reading input file "0x99"
avrdude: writing hfuse (1 bytes):

Writing | ################################################## | 100% 0.00s

avrdude: 1 bytes of hfuse written
avrdude: verifying hfuse memory against 0x99:

Reading | ################################################## | 100% 0.00s

avrdude: 1 bytes of hfuse verified
avrdude: reading input file "0xF3"
avrdude: writing efuse (1 bytes):

Writing | ################################################## | 100% 0.00s

avrdude: 1 bytes of efuse written
avrdude: verifying efuse memory against 0xF3:

Reading | ################################################## | 100% 0.00s

avrdude: 1 bytes of efuse verified

avrdude done.  Thank you.
```

## Flashing QMK

Last step is to get QMK back onto the board again. Luckily now we don't need to bother with the cumbersome lufa bootloader anymore and we can as we are used to.

- Clone the QMK repo
- change the bootloader in `keyboards/kbdfans/kbd67/mkiirgb_iso/rules.mk`
  ```diff
  - BOOTLOADER = lufa-ms
  - BOOTLOADER_SIZE = 6144
  + BOOTLOADER = atmel-dfu
  ```
- build and flash firmware
  ```bash
  make kbdfans/kbd67/mkiirgb_iso:via:flash
  ```
