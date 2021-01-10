---
title: Terminal Access on routers via UART
layout: post 
author: M. Cory Billington
excerpt_separator: <!--more-->
---
## How to get a shell on your router (hopefully)
Vulnerability hunting is hard, and it's even harder if you don't have access to the source. Hardware devices make this even tougher as usually the firmware is distributed via binary blobs. You can `binwalk` them and get a `squashfs` usually, but IMO it's still not the same as seeing exactly what's running dynamically on _your_ router. So, lets _hardware_ hack our way in using some built-in functionality :)
## Supplies
You will need:  
* An FTDI controller to convert from RS232 to USB  
  [https://www.microcenter.com/product/486570/-ftdi-adapter-usb-controller](https://www.microcenter.com/product/486570/-ftdi-adapter-usb-controller)
    <figure class="image">
      <figcaption>I used this one from Micro Center</figcaption>
      <img src="/static/ftdi.jpeg" width="40%"/>
    </figure>
* Jumpers of some sort (most anything conductive will work...)
    <figure class="image">
      <figcaption>I picked up a pack of these at Micro Center</figcaption>
      <img src="/static/jumpers.jpeg" width="40%"/>
    </figure>
* Multimeter (anything that can measure resistance and voltage will do)
* [optional] Soldering iron (if you want to solder the jumpers on)
* [optional] Diagonal cutters/wire cutters/etc.
## Before we start
The router you see in this how-to is a [tp-link TL-WR841N](https://www.tp-link.com/us/home-networking/wifi-router/tl-wr841n/) but this technique can be used on many different boards. I've also done this successfully on a [NETGEAR R6080](https://www.netgear.com/home/wifi/routers/r6080)  
I am also using Ubuntu 20.04, but any Linux OS should work, or something like Putty on Windows. 
## Hardware mods
Start by taking your router apart and finding the UART port on the circuit board. It will look like this:  
    <figure class="image">
      <img src="/static/jumpers.jpeg" width="40%"/>
    </figure>

Luckily here, the ports are clearly labeled
For this model
```
VCC - 3.3 volts DC constant
GND - Ground
RX - Receive data
TX - Transfer data
```
Now would be a good time to identify these if they weren't labeled. Ground is pretty easy... Just measure resistance between each pin until you find `0` ohms. I use the diode checker as it gives an audible indication when you hit `0` ohms. I leave it for a couple seconds as `RX` can also give `0` for a split second. You want a constant `0`.  

`TX` can be a bit trickier. You will be looking for _almost_ `3.3v`. A solid, steady `3.3v` is likely `VCC`, whereas `TX` is going to fluctuate a bit, as this voltage is actually data being sent. An analog multimeter might make identifying this a bit easier (thinking back to Simpson multimeters in my Navy days...) as you would be able to see the needle bounce a bit. Worst case here, trial and error between `VCC` and `TX`.  

At this point, you are probably ready to start soldering if you want to do that. The process for me looked like this:  
<figure class="image">
    <figcaption>I used this one from Micro Center</figcaption>
    <img src="/static/cut-jumper.jpeg" width="40%"/>
</figure>
<figure class="image">
    <figcaption>I used this one from Micro Center</figcaption>
    <img src="/static/before-solder.jpeg" width="40%"/>
</figure>
<figure class="image">
    <figcaption>I used this one from Micro Center</figcaption>
    <img src="/static/solder.jpeg" width="40%"/>
</figure>
Repeat for each jumper `TX RX GND`  

Now, you are ready to connect these jumpers to your FTDI controller. Here is a pic of the pinout from my setup:  
<figure class="image">
    <figcaption>I used this one from Micro Center</figcaption>
    <img src="/static/pinout.jpeg" width="40%"/>
</figure>
You can see, these will appear backwards. What is being _transmitted_ (`TX`) from the router will be _received_(`RXD`) on the FTDI controller. If you mix these up, your router will probably just not boot, which is what mine just did when I tested this. In that case, just swap them and try again.

You'll then want to confirm voltage and set your controller appropriately. Mine had a switch for `5v` or `3.3v`, so I set it to `3.3v`. To check, measure voltage at the `VCC` port:  
<figure class="image">
    <figcaption>3.23v is close enough...</figcaption>
    <img src="/static/voltage-check.jpeg" width="40%"/>
</figure>

Now you are almost ready to boot the router. Connect your controller to your computer and look for the device file. It will probably be:  
```/dev/ttyUSB0```  
which you can find by running:  
```ls /dev/ttyU*```  
From here, you'll need something to connect to that serial device. I used `screen`, which you can install using:  
```apt install screen```  
To connect to the serial device, you'll run the following as root:  
```screen /dev/ttyUSB0 <baud rate>```  
The possible baud rates are:  
```110 300 600 1200 2400 4800 9600 14400 19200 38400 57600 115200 128000 256000```  
In this case, the correct baud rate was `115200`, so the command looked like this:  
```screen /dev/ttyUSB0 115200```  
After running the command, you will then power the router on. It will automatically start transmitting data and you will see output. If you get this incorrect, you'll just get jibberish on the screen like so when you plug the router in.:  
<figure class="image">
    <figcaption>non-printable garbage from incorrect baud rate</figcaption>
    <img src="/static/bad-baud.png" width="40%"/>
</figure>
This is fine, just exit using `Ctrl + a` and then `:quit` and try again. When you get the baud rate correct, you will see a boot log output to the screen when you power the router on. It will look like this gif and you will drop into a shell:  
<figure class="image">
    <figcaption>Successful connection to UART</figcaption>
    <img src="/static/boot-uart.gif" width="60%"/>
</figure>
After running the screen command in the gif above, I power the router on, which is the delay before the log is printed to screen. Also, you will likely not get a shell prompt, and messages will keep displaying to screen even after you have a shell. So, I recommend running something like `ls` to see if you get output. You can also see that this router does not have a lot of commands. Most of the functionality on this machine would come from `/bin/busybox`, which contains more functionality/commands. Sometimes, you may want to copy over a full `busybox` binary that has more built-in commands.

## Happy hacking!