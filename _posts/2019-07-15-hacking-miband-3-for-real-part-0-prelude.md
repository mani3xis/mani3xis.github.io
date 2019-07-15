---
layout: post
title: Hacking MiBand 3 for real, Part 0 - Prelude
categories: [hacking]
tags: [reverse engineering, miband]
---

I was a really happy Mi Band 3 user for more than a year. What has changed?
One of my family members bought a fully-blown smartwatch -- and that's fine, as I don't need all the bells and whistles.
But what really irks me, is the fact that he can develop his own apps for it.
Don't get me wrong -- I still love my watch, its simplicity, how the battery lasts long enough, the lack of mic, etc. All I want to do is to add some features to it (e.g. music controls), so let's make this happen!

Then I've asked myself if anyone has already done it. Well, I was really surprised once I've found
[both Yogesh Ojha's](https://medium.com/@yogeshojha/i-hacked-xiaomi-miband-3-and-here-is-how-i-did-it-43d68c272391)
[attempts](https://medium.com/@yogeshojha/i-hacked-miband-3-and-here-is-how-i-did-it-part-ii-reverse-engineering-to-upload-firmware-and-b28a05dfc308).
Personally, I wouldn't call it "real hacking" as he is just using built-in [GATT characteristics](https://www.bluetooth.com/specifications/gatt/characteristics/).
Nevertheless, his posts provided some useful information and really motivated me to do something more ambitious 🤓


What are we dealing with? The Hardware
--------------------------------------
So the primary question is: where to start? To be honest, I have no experience in reverse-engineering, but this is not a big issue -- kinda contrary -- I love challenges and learning new things 🤓
I've been crafting software for about 11 years now, so doing the opposite shouldn't be that hard. **Having the right mindset, all I can do now is just invite you to join my journey!**

I've decided to look into the guts of the watch to get a gist of what I'm dealing with.
But I ain't cuttin' my precious! Fortunately, there are [people](https://youtu.be/Zc1eOmhG6do?t=72) that (crack) opened [their](https://youtu.be/fhSvAN6aTdw?t=139) devices! [Lucky me!](https://youtu.be/5NV6Rdv1a3I)

So there is ARM inside. To be more specific: the main suspect is [DA14681 from Dialog](https://www.dialog-semiconductor.com/products/connectivity/bluetooth-low-energy/smartbond-da14680-and-da14681).
Its a Cortex-M0 based [SoC](https://en.wikipedia.org/wiki/System_on_a_chip) responsible for executing the software with BLE 4.2 support, integrated crypto engine and striking 128kB Data RAM!
Touch gestures are handled by [Azoteq IQS2661i0](https://www.azoteq.com/product/iqs266/) -- an [I2C](https://en.wikipedia.org/wiki/I%C2%B2C) capacitive trackpad controller with a self-capacitive wake-up.
On the first video I also saw a [GD25LQ32D](https://www.gigadevice.com/flash-memory/gd25lq32d/) 32Mbit serial flash.
Finding datasheets for those was hassle-free, so without further ado, let's move on!


What are we dealing with? The Software
--------------------------------------
Now, let's focus on the software side. Firmware images are being distributed with the official [Mi Fit APK](https://play.google.com/store/apps/details?id=com.xiaomi.hm.health&hl=en). You can also find those on multiple APK mirror sites that often allow you to download various versions which is neat.
Since APK files are basically Zip archives, you can easily extract them and browse their content. Here are the relevant files that I've found in the APK's ``assets/`` directory:

- ``Mili_wuhan.ft``
- ``Mili_wuhan.ft.kj``
- ``Mili_wuhan.fw``
- ``Mili_wuhan.res``

Those files are being uploaded (or shall I say "flashed") to the device via Bluetooth (Low Energy) using the official app, [Gadgetbridge](https://gadgetbridge.org/) or if you are more adventurous, using the aforementioned DIY method described in [Yogesh Ojha's second post](https://medium.com/@yogeshojha/i-hacked-miband-3-and-here-is-how-i-did-it-part-ii-reverse-engineering-to-upload-firmware-and-b28a05dfc308).
``Mili_wuhan.fw`` caught my special attention due to its extension -- its literally screaming *"I'm a firmware image"*. Better be unencrypted...


Quick firmware analysis
-----------------------
Compression or even encryption can really obfuscate the file content making the reverse-engineering process a lot harder as the entropy increases. If this would be the case then I would probably try my luck with [``binwalk``](https://github.com/ReFirmLabs/binwalk) to search for common headers/byte sequences.
If this wouldn't help either then as a last resort I would start looking for the very oldest version I could possibly find and try again with it. Who knows, maybe the encryption or compression (please do not mix them up) was introduced with a firmware upgrade...

One really quick way to check if we are doomed before really getting started is to run the [``strings``](https://linux.die.net/man/1/strings) utility on it (or [``hexdump``](https://linux.die.net/man/1/hexdump) if you have it installed).
If you expect some particular strings to be there, you can also filter them with ``grep`` to reduce the noise. FYI: Both ``strings`` and ``grep`` (and many others) are bundled with [Git for Windows](https://gitforwindows.org/).
Long story short: the **data section is not obfuscated** as we can clearly see some strings there!

One last thing that I wanted to check is whether we are dealing with a flat executable image or not.
Cortex-M0 always starts execution at ``0x00000000`` (or whatever is remapped to this address) and expects an [IVT (Interrupt Vector Table)](http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.dui0497a/BABIFJFG.html) at the beginning of the image. The first four bytes is the initial address of Stack Pointer and the next DWORD is the address of reset handler.
Since the datasheet for DA14681 lists all 32 [IRQs](https://en.wikipedia.org/wiki/Interrupt_request), I've prepared a short Python script to dump all the addresses from the firmware blob where the IVT should be located in a more readable form to check if there is garbage or not. Here's the output:

```
$ python dump_vector_table.py Mili_wuhan.fw
Init SP = 0x07fc7800
==================================================
0x08004141 = #1 Reset
0x07fcb401 = #2 NMI
0x07fcb419 = #3 HardFault
0x00000000 = #4 Reserved
0x00000000 = #5 Reserved
0x00000000 = #6 Reserved
0x00000000 = #7 Reserved
0x00000000 = #8 Reserved
0x00000000 = #9 Reserved
0x00000000 = #10 Reserved
0x080041a1 = #11 SVCall
0x00000000 = #12 Reserved
0x00000000 = #13 Reserved
0x08018719 = #14 PendSV
0x08004195 = #15 SysTick
0x07fcba25 = IRQ0: ble_wakeup_lp_irq
0x07fcb9f9 = IRQ1: ble_gen_irq
0x08004195 = IRQ2: RESERVED
0x08004195 = IRQ3: RESERVED
0x08004195 = IRQ4: rfcal_irq
0x0800c77d = IRQ5: RESERVED
0x0800c751 = IRQ6: crypto_irq
0x08004195 = IRQ7: mrm_irq
0x0800f02d = IRQ8: uart_irq
0x0800f03d = IRQ9: uart2_irq
0x0800d651 = IRQ10: i2c_irq
0x0800d66d = IRQ11: i2c2_irq
0x0800ecd1 = IRQ12: spi_irq
0x0800ece1 = IRQ13: spi2_irq
0x0800d081 = IRQ14: adc_irq
0x0800d79d = IRQ15: keybrd_irq
0x0800d689 = IRQ16: irgen_irq
0x08010f5d = IRQ17: wkup_gpio_irq
0x0800ed79 = IRQ18: swtim0_irq
0x08010f71 = IRQ19: swtim1_irq
0x0800da41 = IRQ20: quadec_irq
0x0800f0d1 = IRQ21: usb_irq
0x08004195 = IRQ22: pcm_irq
0x08004195 = IRQ23: src_in_irq
0x08004195 = IRQ24: src_out_irq
0x0800f0f9 = IRQ25: vbus_irq
0x0800cdfd = IRQ26: dma_irq
0x08011b49 = IRQ27: rf_diag_irq
0x0800ee71 = IRQ28: trng_irq
0x08004195 = IRQ29: dcdc_irq
0x08010a11 = IRQ30: xtal16rdy_irq
0x08004195 = IRQ31: RESERVED
==================================================
0x00000000 occurred 9 times: ['#4 Reserved', '#5 Reserved', '#6 Reserved', '#7 Reserved', '#8 Reserved', '#9 Reserved', '#10 Reserved', '#12 Reserved', '#13 Reserved']
0x08004195 occurred 10 times: ['#15 SysTick', 'IRQ2: RESERVED', 'IRQ3: RESERVED', 'IRQ4: rfcal_irq', 'IRQ7: mrm_irq', 'IRQ22: pcm_irq', 'IRQ23: src_in_irq', 'IRQ24: src_out_irq', 'IRQ29: dcdc_irq', 'IRQ31: RESERVED']
```

**Bingo!** Each address is odd indicating that handler is written in Thumb mode, as required by Cortex-M0 specs. It gets only better: there is a high correlation between reserved interrupts and NULL handlers which is a next good sign!
We also found out, that there is one shared handler used for 10 different IRQs! Thanks, Python!

> In layman terms: *Thumb* is a second instruction set that is optimized for size.
> ARM cores support two instruction sets: the full 32-bit one and 16-bit called Thumb which simply provides shorter codes for common 32-bit instructions.


Roadmap
-------
So, in theory, we can "attack" the device by uploading a modified or even totally rewritten firmware. I will start dissecting the firmware image starting with next blog post to see if there are any other attack vectors, so **please stay tuned**.

I believe the first milestone would be to get a complete memory dump (ROMs, OTP and RAM) to get a clearer picture of what's there. This should definitely aid reverse-engineering. Perhaps this can be accomplished without any firmware modifications by exploiting some [common vulnerabilities](https://en.wikipedia.org/wiki/Buffer_overflow) -- we will see what the future will bring.
One brilliant idea that stroked me recently is to either add a new GATT service or to modify existing one allowing me to [peek and poke](https://en.wikipedia.org/wiki/PEEK_and_POKE) the memory remotely.
This might be dangerous indeed, as [Bluetooth is the only way to upload firmware](https://en.wikipedia.org/wiki/Over-the-air_programming) so we must, at all costs, not break it!

Next epic milestone would be the development of an SDK that would allow anybody to build their own firmware or at least individual functions that can be injected into official firmware to augment it.
Finally, the last milestone would be to reproduce the complete source code that would compile to something similar to what's already in the compiled blob and release it under an open-source license!


Wrap up
-------
If everything will go smoothly then we -- developers -- will have a new *toy* to play with and have the possibility add some useful features (e.g. music controls that I really want).
On the other hand, if I screw at least one thing up, then I will end up with a brick that can be attached to my wrist. No pressure.
No matter which outcome the future will give, I will definitely be in a better place equipped with new skills 😎 
