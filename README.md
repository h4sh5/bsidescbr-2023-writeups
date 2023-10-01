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

<img src=circuit_labels.png>

There are a number of hardware interface challenges that require hooking up the GPIO utility pins on the side of the board (e.g. for I2C, SPI and serial protocols). Only the utility pins needed to be connected to for the CTF challenge (and obviously the USB-C port for power and access via a computer serial terminal); the rest of the ports (e.g. JTAG, serial programming, etc) do not need to be touched.

The bpod comes with pre-existing tools to sniff all protocols required to complete the challenges under the Tools menu. If you have special hardware like the Flipper Zero, that can also be used as I believe it supports i2c, spi and serial as well.

In case you overwrite the ESP32's bootloader for some reason, or interrupt the flashing process, or somehow brick the entire thing, you can get an ISP programmer of some sort (like from [Jaycar](https://www.jaycar.com.au/duinotech-isp-programmer-for-arduino-and-avr/p/XC4627)) to still be able to program the chip over the ISP programming port.

Soldering on a header provided at the Hardware Village using one of their provided soldering stations make it a lot easier to hook up the bpod to other things (like another bpod!)

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



### Cheesy Strings I

Challenge description:

```
I don't want to string you along for too long.
```


You can either use the `bpod.bin` file that the updater script dropped, or dump the flash using `esptool.py` (https://pypi.org/project/esptool/) by running something like `esptool.py -p /dev/ttyUSB0 -b 460800 read_flash 0 0x400000 flash.bin` (found this on a related medium article about [reverse engineering esp32 flash dumps](https://olof-astrand.medium.com/reverse-engineering-of-esp32-flash-dumps-with-ghidra-or-ida-pro-8c7c58871e68))


```
$ strings bpod.bin | rg cybears
cybears{h0w_l0ng_1s_a_p1ec3_0f_str1ng_ch33s3}
```

### Serial Hacker

Challenge description:

```
Are you a serial hacker?
```

You just need to hook up two bpods with pins SERIAL_TX1, SERIAL_RX1 and GND, as described on the bpod's Tools -> uartterm app. Then you can use the uartterm app to sniff out the flag.

> UART (Universal Asynchronous Receiver-Transmitter) is a computer hardware device and protocol for asynchronous serial communication for a bus with a clock. This means that baud rates are configurable (faster means more chance for errors), as well as things like parity bits. For a fun, long read (not right now!) on serial consoles, see http://www.catb.org/~esr/faqs/things-every-hacker-once-knew/

The TX pin of a bpod should go to the RX pin of the other, and vice versa, The GND pin should go to another GND.

Then, change the baud rate to 9600 and you'll see the flag repeat itself in your uartterm:

```
==== Reading ====

cybears{u_r_a_s3r1al_h4ck3r}cybears{u_r_a_s3r1al_h4ck3r}cybears{u_r_a_s3r1al_h4ck3r}cybears{u_r_a_s3r1al_h4ck3r}
```

flag:
`cybears{u_r_a_s3r1al_h4ck3r}`


### I can see you

 Challenge description:

```
I too see you.
```

So.. "I too see you" sounds like "I 2 c you" which is a hint that this is the I2C challenge.




### A Place Called Vertigo

Challenge description:

```
Cybears lost their iPod, with their favourite 'pre-bundled no-opt-out artist' U2 on it! One of their favourite tracks had a hidden watermark applied - but dont worry, it's encrypted with a key. Unfortunately the AI found the key on pastebin:

key: db928fb0b0081a3e9c225d51aa1b688e

Hope no one finds that file!

Flag format is .cybears{XXXXX}.
```

There is a music file in the firmware somewhere, with the U2 song:

`rg -a WAVE bpod.bin`  (using ripgrep)

shows that there's some data with `RIFFd....WAVEfmt "Vfdata@` 

If we look up "wav watermark", we find this repo https://github.com/swesterfeld/audiowmark

which looks like they have a key very similar to the one in the challenge description (same length in hex)

So let's look at what a WAV header looks like:

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



