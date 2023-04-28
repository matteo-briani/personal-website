---
title: Debug SPI invalid argument
layout: post
categories: embedded
---

![Olimex and 433 mhz tranceiver]({{site.baseurl}}/assets/images/olimex433.jpg)


## The issue

One day in the lab, the requirement is a rapid prototyping, trasmitting data over the 433Mhz spectrum; nothing more nothing less.
The hardware is choosen with the following strict rule: "it has to be something in some drawer in the lab".
So, we found ourself working with the E07-M1101D tranceiver module (based on the well known cc1101 ic), and the Olimex STMP157-SOM with its relative evaluation board (based on the STM32MP157).

Once wired up the transceiver to the board and fix the permissions to the `/dev/spidev0.0` device, we are ready to attempt our first transmission over the 433 Mhz spectrum.

Without much of a thinking, we end up using [a Python package](https://github.com/fphammerle/python-cc1101) to transmit our first bits.
Full of hopes, we launch the cli command:

```
printf '\x01\x02\x03' | cc1101-transmit -f 433920000 -r 1000
```

aaaaand...nothing, we are welcomed with a burp from the Linux kernel in the vest of a Python exception
```
Traceback (most recent call last):
  File "/home/olimex/boh/bin/cc1101-transmit", line 8, in <module>
    sys.exit(_transmit())
  File "/home/olimex/boh/lib/python3.9/site-packages/cc1101/_cli.py", line 157, in _transmit
    with cc1101.CC1101(lock_spi_device=True) as transceiver:
  File "/home/olimex/boh/lib/python3.9/site-packages/cc1101/__init__.py", line 563, in __enter__
    self._reset()
  File "/home/olimex/boh/lib/python3.9/site-packages/cc1101/__init__.py", line 270, in _reset
    self._command_strobe(StrobeAddress.SRES)
  File "/home/olimex/boh/lib/python3.9/site-packages/cc1101/__init__.py", line 250, in _command_strobe
    response = self._spi.xfer([register | self._WRITE_SINGLE_BYTE])
OSError: [Errno 22] Invalid argument
```

## Dear SPI, are you actually working?

Ok, first things first, we are using the stock Olinuxino image and we feel kind of blind about all the peripherals setups. Should we trust the device tree, the kernel config, etc. 
We need more proofs.

We want to check that the SPI is working properly before digging deeper in the software.
Luckily, there is a very fast way to proceed.
With the help of the `spidev-test` utility (a copy extracted from the Linux kernel can be found [here](https://github.com/rm-hull/spidev-test)), we remove the transceiver, connect the MOSI and MISO pins with a jumper and launch the command

```
./spidev_test -v
```

The command output is quite reassuring:

```
spi mode: 0x4
bits per word: 8
max speed: 500000 Hz (500 KHz)
TX | FF FF FF FF FF FF 40 00 00 00 00 95 FF FF FF FF FF FF FF FF FF FF FF FF FF FF FF FF FF FF F0 0D  | ......@....?..................?.
RX | FF FF FF FF FF FF 40 00 00 00 00 95 FF FF FF FF FF FF FF FF FF FF FF FF FF FF FF FF FF FF F0 0D  | ......@....?..................?.
```
TX and RX are equals; we can conclude that the SPI is working properly, so the problem lies somewhere else. 
No excuses now, let's find out.

## strace, can you please help me out? 

Let's run the failing command with `strace` (don't mind the extra step of the virtual environment activation, it really does not matter):
```
strace -fv bash -c 'source /home/olimex/boh/bin/activate && printf '\x01\x02\x03' | cc1101-transmit  -f 433920000 -r 1000'
```

Digging the output of `strace` in the quest of finding `EINVAL`, we end up with the following lines

```
[pid   788] read(0, "x01x02x03", 8192)  = 9
[pid   788] read(0, "", 8183)           = 0
[pid   788] openat(AT_FDCWD, "/dev/spidev0.0", O_RDWR|O_LARGEFILE) = 3
[pid   788] ioctl(3, SPI_IOC_RD_MODE, 0xbeb14dd7) = 0
[pid   788] ioctl(3, SPI_IOC_RD_BITS_PER_WORD, 0xbeb14dd7) = 0
[pid   788] ioctl(3, SPI_IOC_RD_MAX_SPEED_HZ, 0xbeb14de0) = 0
[pid   788] flock(3, LOCK_EX|LOCK_NB)   = 0
[pid   788] ioctl(3, SPI_IOC_WR_MAX_SPEED_HZ, 0xbeb15db8) = 0
[pid   788] ioctl(3, SPI_IOC_MESSAGE(32), 0xbeb14b38) = -1 EINVAL (Invalid argument)
[pid   788] close(3)                    = 0
```

Well, well, well, pretty easy to see right? Mmm...no, actually no, debugging `ioctl` messages is really not my free time hobby. 
However, we collected a valuable information: at the beginning of our SPI exchange, we passed an invalid argument to the SPI device.  
Let's stress this, the invalid argument refers to the SPI device, not our CC1101 ic!
So, since we are sure that our dead SPI interface is working - remember the test with `spidev_test` - we are left to check what is actually happening within the invoked code.

## Good habit: read the source!

Take another look at the Python trace-back that we posted in the beginning, we see this line:
```
with cc1101.CC1101(lock_spi_device=True) as transceiver:
```
which is a Python context manager.  It calls the method `__enter__`  of the class `cc1101.CC1101` and assign its result to the variable `transceiver`. At the end of the `with` block, it will call the method `__exit__`.

So, it looks a good place to start looking. If we are lucky, around that line, there could be some hint about our `EINVAL`.

Aaaaaaaand....here it is, at the specific commit, at time of writing, the method `__enter__` is the following 

```python
    def __enter__(self) -> CC1101:
        # https://docs.python.org/3/reference/datamodel.html#object.__enter__
        try:
            self._spi.open(self._spi_bus, self._spi_chip_select)
        except PermissionError as exc:
            raise PermissionError(
                f"Could not access {self._spi_device_path}"
                "\nVerify that the current user has both read and write access."
                "\nOn some systems, like Raspberry Pi OS / Raspbian,"
                "\n\tsudo usermod -a -G spi $USER"
                "\nfollowed by a re-login grants sufficient permissions."
            ) from exc
        if self._lock_spi_device:
            # advisory, exclusive, non-blocking
            # lock removed in __exit__ by SpiDev.close()
            fcntl.flock(self._spi.fileno(), fcntl.LOCK_EX | fcntl.LOCK_NB)
        try:
            self._spi.max_speed_hz = 55700  # empirical
            self._reset()
            self._verify_chip()
            self._configure_defaults()
            marcstate = self.get_main_radio_control_state_machine_state()
            if marcstate != MainRadioControlStateMachineState.IDLE:
                raise ValueError(f"expected marcstate idle (actual: {marcstate.name})")
        except:
            self._spi.close()
            raise
        return self
```

Right before the call to `self.reset()`, a very-very-very much appreciated comment state that the `max_speed_hz` is set to the empirical value of `55700`.
Mmm...our `spidev_test` is using a completely different speed: `500000`.
What would happen if we tweak that speed to our "proven" value?

Manually change the code to 
```
self._spi.max_speed_hz = 500000
```
and run the command again:

```
printf '\x01\x02\x03' | cc1101-transmit -f 433920000 -r 1000
```
The output is now
```
CC1101(marcstate=idle, base_frequency=433.92MHz, symbol_rate=1.00kBaud, modulation_format=ASK_OOK, sync_mode=TRANSMIT_16_MATCH_16_BITS, preamble_length=4B, sync_word=0xd391, packet_length≤255B, output_power=(0xc6,0))
transmitting 0x09783031783032783033 (b'\tx01x02x03')
```

Eureka! It works!!!

To be fair, the actual proof of the transmission has been done by monitoring the 433.92MHz spectrum with a software defined radio - SDR.
