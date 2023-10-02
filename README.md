# Bsides Canberra 2023 Badge challenges writeup

## Introduction

Bsides Canberra 2023 comes with a very cool badge - the revolutionary bPod, complete with scrollwheel controls akin to the original iPod. Of course, it also comes with a nice bunch of CTF challenges.

You can find the bPod website here https://bpod.bsidescbr.com.au/update.html

There is a firmware updater script available online (link found in the QR code in the update settings of the bPod itself). The script tries to connect to your bPod, identifies its chipset (a diverse range of ESP32 chipsets with various settings were used, smartly by the creator to circumvent chip supply chain shortages by diversifying. Huuuuge kudos to Peter who made it; everything has been open sourced here on this Gitlab https://gitlab.com/pjranki/bpod)

.. ,then decodes and decompresses the embedded base64 firmware data and flashes it over serial (usb-C connector) to the device. It also dumps the firmware files used to flash it into the current directory.

We are going to work with the firmware file that the [bPodUpdater.py](https://bpod.bsidescbr.com.au/static/bPodUpdater.py) script taken from the [bpod website](https://bpod.bsidescbr.com.au/update.html) dumped onto the disk when it runs (`bpod.bin`). It also drops `partition-table.bin` and `bootloader.bin` but they've been largely useless for the CTF.

> Note that the script may have been updated since the time of writing, and we may have different chipsets; therefore your binary offsets may not be the same as mine. A different static snapshot of the updater script is taken [here on waybackmachine](https://web.archive.org/web/20231001122022/https://bpod.bsidescbr.com.au/static/bPodUpdater.py)


## Ports and Soldering

The ports on the back of the board are labeled below for your convenience:

<img src=circuit_labels.png height=700>

There are a number of hardware interface challenges that require hooking up the GPIO utility pins on the side of the board (e.g. for I2C, SPI and serial protocols). Only the utility pins needed to be connected to for the CTF challenge (and obviously the USB-C port for power and access via a computer serial terminal); the rest of the ports (e.g. JTAG, serial programming, etc) do not need to be touched.

The bpod comes with pre-existing tools to sniff all protocols required to complete the challenges under the Tools menu. If you have special hardware like the Flipper Zero, that can also be used as I believe it supports i2c, spi and serial as well.

In case you overwrite the ESP32's bootloader for some reason, or interrupt the flashing process, or somehow brick the entire thing, you can get an ISP programmer of some sort (like from [Jaycar](https://www.jaycar.com.au/duinotech-isp-programmer-for-arduino-and-avr/p/XC4627)) to still be able to program the chip over the ISP programming port.

Soldering on a header strip provided at the Hardware Village using one of their provided soldering stations make it a lot easier to hook up the bpod to other things (like another bpod!)

Similar pin header strips can be bought easily. For example, this [40 pin header strip at Jaycar](https://www.jaycar.com.au/40-pin-header-terminal-strip/p/HM3212. You can just manually cut/snap it down to the number of pins you want (in this case, 12).

Here's my relatively nice and clean soldering job. It took less than 10 minutes for me to finish.

<img height=200 src=solder_angle.png>

<img height=200 src=solder_side.png>


## USB serial console on a computer terminal

You can use your computer's terminal to interact with the serial terminal by using `screen /dev/<port> 115200` (like `screen /dev/ttyACM0 115200` on Linux). That will set the baud rate to 115200, but anything else should also do (like 9600)

The crazy cool thing is that the serial terminal on your screen will mirror exactly what you're seeing on the bpod's display (not to the pixel, but semantically).

This means things like serial terminal (UART), I2C and SPI messages sniffed from the bpod's apps can be directly copied to your computer into tools like CyberChef.

## Challenges

- [Cheesey Strings I](#cheesy-strings-i)
- [Serial Hacker](#serial-hacker)
- [I can see you](#i-can-see-you)
- [I spy with my little eye](#i-spy-with-my-little-eye)
- [A Place Called Vertigo](#a-place-called-vertigo)
- [Blinky Bill](#blinky-bill)
- [Cheesy Strings II](#cheesy-strings-ii)

You can also find most of the author's quick walkthroughs of these challenges (with quite little detail) here https://gitlab.com/pjranki/bpod/-/tree/main/firmware/ctf/chals/badge?ref_type=heads


## Cheesy Strings I

Challenge description:

```
I don't want to string you along for too long.
```


You can either use the `bpod.bin` file that the updater script dropped, or dump the flash using `esptool.py` (https://pypi.org/project/esptool/) by running something like `esptool.py -p /dev/ttyUSB0 -b 460800 read_flash 0 0x400000 flash.bin` (found this on a related medium article about [reverse engineering esp32 flash dumps](https://olof-astrand.medium.com/reverse-engineering-of-esp32-flash-dumps-with-ghidra-or-ida-pro-8c7c58871e68))


```
$ strings bpod.bin | rg cybears
cybears{h0w_l0ng_1s_a_p1ec3_0f_str1ng_ch33s3}
```

## Serial Hacker

Challenge description:

```
Are you a serial hacker?
```

You just need to hook up two bpods with pins SERIAL_TX1 (transmit), SERIAL_RX1 (receive) and GND (ground), as described on the bpod's Tools -> uartterm app. Then you can use the uartterm app to sniff out the flag.

> UART (Universal Asynchronous Receiver-Transmitter) is a computer hardware device and protocol for asynchronous serial communication for a bus with a clock. This means that baud rates are configurable (faster means more chance for errors), as well as things like parity bits. For a fun, long read (not right now!) on serial consoles, see http://www.catb.org/~esr/faqs/things-every-hacker-once-knew/

The TX pin of a bpod should go to the RX pin of the other, and vice versa, The GND pin should go to another GND.

Then, change the baud rate to 9600 and you'll see the flag repeat itself in your uartterm. If you don't, reboot the bpod so it starts sending messages again.

```
==== Reading ====

cybears{u_r_a_s3r1al_h4ck3r}cybears{u_r_a_s3r1al_h4ck3r}cybears{u_r_a_s3r1al_h4ck3r}cybears{u_r_a_s3r1al_h4ck3r}
```

flag:
`cybears{u_r_a_s3r1al_h4ck3r}`


## I can see you

Challenge description:

```
I too see you.
```

So.. "I too see you" sounds like "I 2 c you" which is a hint that this is the [I2C](https://en.wikipedia.org/wiki/I%C2%B2C) (I squared C, or IIC) challenge.

I2C, or Inter-Integrated Circuit, is a synchronous, multi-master/multi-slave (controller/target), packet switched, single-ended, serial communication bus.

This is the most straight forward hardware interface challenge; there are no configurations to set, no complex pin wiring. Just SDA to SDA (Serial Data Line) and SCK to SCK (Serial Clock Line) on both sides, as well and GND (ground).

<img src=i2c_pins.png height=500>

After hooking them up, just open the i2csniff app on one bpod, and reboot the other bpod. It will start spitting out ASCII-looking values:

<img src=i2c_flag.png height=500>

`S66W+63+79+62+65+61+72+73+7B+69+5F+32+5F+63+5F+79+30+75+7D+s`

Decoding it <a target=_blank href="https://cyberchef.org/#recipe=From_Hex('Auto')&input=UzY2Vys2Mys3OSs2Mis2NSs2MSs3Mis3Mys3Qis2OSs1RiszMis1Ris2Mys1Ris3OSszMCs3NSs3RCtz">from hex using CyberChef</a>

(we can ignore the first S66W start header byte)

`cybears{i_2_c_y0u}`


## I spy with my little eye

Challenge description:

```
I spy with my little eye.
```

Sounds like SPI to me! This is probably the most painful hardware challenge to hook up. As described by the notes in most bpod tools, it's a "best effort" to decode SPI packets, so there might be a lot of noise. It takes 5 wires.

According to wikipedia, these are the pins:

```
    SCLK : Serial Clock (clock signal from main)

    MOSI : Main Out Sub In (data output from main)

    MISO : Main In Sub Out (data output from sub)
    __   ___________
    CS : Chip Select (active low signal from main to address subs and initiate transmission)
```

The diagram on the bpod app (when accessed via the USB console) is like this:

```
    ==== Diagram ====
      bPod              other
    o [GND]--------------[GND]
    o [IO5]---------------[SO]
    o [IO6]---------------[SI]
    o [IO7]---------------[CS]
    o
    o [SCK]--------------[SCK]
    o
    o
    o
```

And it tells you to NOT use the SPI port from one to the other, but rather use the IO5-7 pins, otherwise it could stop the display from working. Sometimes even when you don't touch the pins, one or both of your bpod's screens will stop working anyway. It's okay, just use the USB-C cable and `screen` (or any other serial terminal, like the one in Arduino IDE) to control the bpod.

So, hooking up the clock, SO (MISO) and SI (MOSI) isn't that hard. After a couple tries, we realized that the i2c clock pin also works for the clock; the SPI clock pin can sometimes be a bit unreliable. Strange.

The tricky bit is the CS. As wikipedia mentioned, Chip Select is an active low signal, which means that when you give it a high voltage (1), it interprets it as a 0, to address subs and initiate transmission. So for this purpose we want an always high signal pin, such as 3V.

So an actual working wiring configuration is, strangely enough:

```
      bPod                other bPod
    o [GND]---------------[GND]
    o [IO5]---------------[MISO]
    o [IO6]---------------[MOSI]
    o [IO7]---------------[3V]
    o
    o [I2C_SCK]-----------[SPI_SCK]
```

(Maybe the I2C_SCK is being used for the sniffer's input? It does say in spisniff notes to NOT use the marked SPI pins on the bpod, so maybe that's why a different SCK has to be used)

<img src=spi_wires.png height=500>

After it's hooked up, you can reboot the target bpod, and keep sniffing with the spisniff app through the noise. Eventually it spits out something semi-ASCII-hex-looking:

```
63:63+79:79+
61:61+72:72+73:73+7B:7B+69:69+5F:5F+73:73+70:70+31:31+7D:7D+63:63+79:79+62:62+65:65+61:61+72:72+73:73+7B:7B+69:69+5F:5F+70:70+31:31+7D:7D+
```

Which is hex encoded:

```
ccyyaarrss{{ii__sspp11}}ccyybbeeaarrss{{ii__pp11}}
```

Cleaning it up:

`cybears{i_sp1}`


## A Place Called Vertigo

Challenge description:

```
Cybears lost their iPod, with their favourite 'pre-bundled no-opt-out artist' U2 on it! One of their favourite tracks had a hidden watermark applied - but dont worry, it's encrypted with a key. Unfortunately the AI found the key on pastebin:

key: db928fb0b0081a3e9c225d51aa1b688e

Hope no one finds that file!

Flag format is .cybears{XXXXX}.
```

<img src=music.png height=400>

There is a music file in the firmware somewhere, with the U2 song. Let's grep for common music file headers, such as WAV (I am using `ripgrep`, which is a faster grep):

`rg -a WAV bpod.bin`

shows that there's some data with `RIFFd....WAVEfmt "Vfdata@` 

If we look up "wav watermark", we find this repo https://github.com/swesterfeld/audiowmark

which looks like they have a key very similar to the one in the challenge description (same length in hex)

So let's look at what a WAV file header looks like:

https://stackoverflow.com/questions/28137559/can-someone-explain-wavwave-file-headers/28137825#28137825

looks like from bytes 5-8 (the 4 bytes after "RIFF"), we have the size of the file. 

<img src=wav_size.png>


that value in hex is `64 92 09 00` (which in big endian is 1,687,292,160, aka way too big), and in little endian `00 09 92 64` = 627,300 (not too big, 600KB ish)

So let's extract that out. From the hex editor we can see that the address of RIFF (the start of the WAV file) is at 0x16088. 

So we can use 0x16088 as the offset (`skip` argument in `dd`) and 0x099264 as the size (`count` argument in `dd`) to carve out this the wav: (the `$((NUM))` syntax is the POSIX compliant way to evaluate a number into decimal form, be in hex, octal or else)

`dd if=bpod.bin of=u2.wav.bin bs=1 skip=$((0x16088)) count=$((0x099264))`


we can tell that we have succeeded by running `wc` and `file` to verify our carved file.


```
$ wc -c u2.wav.bin ; file u2.wav.bin
  627300 u2.wav.bin
u2.wav.bin: RIFF (little-endian) data, WAVE audio, Microsoft PCM, 24 bit, mono 22050 Hz
```

Now in a separate terminal, we clone the repo https://github.com/swesterfeld/audiowmark, cd into it, build the docker image with `docker build -t audiowmark .`, then back in the same directory as the file we can make a key file for testing:

`docker run -v .:/data --rm -i audiowmark gen-key test.key`

```
$ cat test.key
# watermarking key for audiowmark

key 7da7ae4d0c459ddbd6b8caacb3450e52
```

we can see that their key file format is very simple. Replacing the fake generated key with the real one provided by the challenge description (db928fb0b0081a3e9c225d51aa1b688e) and calling that file `real.key`:

```
$ cat real.key
# watermarking key for audiowmark

key db928fb0b0081a3e9c225d51aa1b688e
```

Finally, we can run `audiowmark get` in our built docker container to get the flag out:

```
$ docker run -v .:/data --rm -i audiowmark get --key real.key u2.wav.bin
pattern  0:00 7b7d637962656172737b6d752421637d 1.394 0.460 CLIP-B
pattern  0:00 edb09016892b6f72189f419357bce22e 0.825 0.814 CLIP-A
pattern  0:00 e87b6771b491885c6b459bf10b4a20ab 0.753 0.803 CLIP-A
pattern  0:00 f9b26a18ad3f97eacb0e8e87f78199ca 0.725 0.795 CLIP-A
pattern  0:00 a05754890a0b7ace703b09f7e0d4146f 0.737 0.808 CLIP-A
```

Hex decoding:

```
echo 7b7d637962656172737b6d752421637d|xxd -r -p; echo
{}cybears{mu$!c}
```

Yay! We got the flag `cybears{mu$!c}`

The CTF scoreboard theme is also pretty cool

<img src=place_called_vertigo_solved.png>

## Blinky Bill

**(Unsolved)**

Challenge description:

```
Head towards the light, I dare you.
```


The lights blink through the LEDs like a stream of binary from right to left, *apparently*. It's a giant pain in the ass, so you likely want to record it with your phone to step through it.

I'm not doing that.

## Cheesy strings II

**(Unsolved)**

Challenge description:

```
Dig deep to find the vault of cheesy goodness.
```

Running binwalk, we find some AES stuff:

```
 binwalk --dd='.*' bpod.bin

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
764528        0xBAA70         AES S-Box
764784        0xBAB70         AES Inverse S-Box
```

The AES S-Box starts with:

```
xxd _bpod.bin.extracted/BAA70 |head
00000000: 637c 777b f26b 6fc5 3001 672b fed7 ab76  c|w{.ko.0.g+...v
00000010: ca82 c97d fa59 47f0 add4 a2af 9ca4 72c0  ...}.YG.......r.
00000020: b7fd 9326 363f f7cc 34a5 e5f1 71d8 3115  ...&6?..4...q.1.
00000030: 04c7 23c3 1896 059a 0712 80e2 eb27 b275  ..#..........'.u
00000040: 0983 2c1a 1b6e 5aa0 523b d6b3 29e3 2f84  ..,..nZ.R;..)./.
00000050: 53d1 00ed 20fc b15b 6acb be39 4a4c 58cf  S... ..[j..9JLX.
```


It's time to reverse engineer the firmware, painfully... Looking up "reverse engineering esp32 firmware", we find these resources:

- https://olof-astrand.medium.com/reverse-engineering-of-esp32-flash-dumps-with-ghidra-or-ida-pro-8c7c58871e68
- https://blog.rop.la/en/reversing/2022/05/02/an-easy-way-to-reconstruct-an-esp32-app-image-to-elf.html
- https://github.com/tenable/esp32_image_parser

We can use https://github.com/tenable/esp32_image_parser to convert bpod.bin into an ELF file, and load it into [Ghidra](https://github.com/NationalSecurityAgency/ghidra/releases) with https://github.com/yath/ghidra-xtensa (a Ghidra plugin you need to install)

Saving this script as `image2elf.py` in the same directory as the git cloned `esp32_image_parser`:

```py
#!/usr/bin/env python3
import sys
from esp32_image_parser import *

print ("{} {}".format(sys.argv[1], sys.argv[2]))

input_file = sys.argv[1]
output_file = sys.argv[2]

image2elf(input_file, output_file, True)
print('written to', output_file)
```

Then running it with

```
python3 image2elf.py bpod.bin bpod.bin.elf
```

After loading it into Ghidra and the analysis is complete, we can search memory for the start of the AES S-Box we found (starting with the string `c|w{`)

You can hit `S` to search a pattern in memory. (Or go to Search -> Memory on the top bar)

We can hit `L` after highlighting the S-box address in the Listing window to give it a label, like `aes_sbox`

And we do the same with the bytes of the Inverse S-box and mark it as well.

Now we use one of the most powerful functionalities of RE tools like Ghidra and IDA - reference finding. If we find a reference that uses `aes_sbox`, it's most likely going to be related to AES functions.

To do so we highlight the `aes_sbox` label in the Listing menu and hit Cmd-Shift-F (mac) or Ctrl-Shift-F (win/linux), depending on your OS.

We find a function with DATA reference to it, and they point to a function at 400aeb64 (your offset might be different; refer to the [bpod.bin](bpod.bin) uploaded to this repo for hopefully the same offsets as me. It's still going to be the same function anyway).

<img src=sbox_refs.png>

The function there just returns the value at an index of the sbox, so I renamed it (via the `L` keyboard shortcut once again) to `get_sbox_index`

<img src=sbox_find_refs.png>

Doing a reference find again, we find a lot more references to the `get_sbox_index`. This is a great sign - meaning that the function is used by many others, showing us that we are on the right track to find the encryption-related functions.

<img src=get_sbox_refs.png>

I should note here that [AES](https://en.wikipedia.org/wiki/Advanced_Encryption_Standard) is a symmetric key encryption algorithm. That means there is one key, and one IV (Initialization Vector) to encrypt and decrypt data. Since this is a firmware image of an embedded device, and there is no sign of public/private encryption (like RSA), we can assume that it has to get the key from somewhere to decrypt the encrypted data; which means that **the key is most likely stored in the same firmware**.

I basically rename/label found functions by the order that I found them, with the prefix being what I think they relate to or what they do. For example, anything functions I find by reference to AES I just name `aes1`, `aes2`, `aes3` .. Feel free to rename them whatever makes sense based on what you think they do.

So if we can follow down this rabbit hole by doing the find refs and then set label "loop", we should be able to find data (`DAT_*`) references in Ghidra for the encryption key as well as the encrypted data.

This is as far as I've gotten; I have not yet solved this challenge, but may update this writeup when I do.

