+++
title = "IR Remote Control for RasBMC Media Center"
date = 2014-01-12T17:20:35
draft = false
categories = ["Electronics"]
slug = "ir-remote-control-for-rasbmc-media-center"
aliases = ["/2014/01/ir-remote-control-for-rasbmc-media-center/"]
+++

In my [last posting](https://www.kj6lnh.org/2013/12/raspberry-pi-set-top-box/ "Raspberry Pi Set Top Box"), I turned a Raspberry Pi into a media center PC. One of the things I left undone before declaring the project a success was to set up some kind of hardware based remote control. I was only able to control the PC using the web interface or the XBMC app on Android and the iPad. I originally wanted to build some kind of remote control that didn’t require me to have my phone next to me and unlock the screen every time I wanted to pause the show. I really don’t need another remote control in the living room. I already have one for the TV, one for the Blue Ray, and one for U-Verse. Fortunately, the U-Verse remote is actually a 4-in-1 universal remote. A quick Google found the manual, so I programmed one of the slots with a Sony DVD player code. It had enough of the buttons defined to be usable.

The more interesting part is getting the IR code into the Raspberry Pi. There are <a href="http://forum.stmlabs.com/showthread.php?tid=5549" target="_blank">instructions</a> out there on the XBMC wiki for putting together an IR receiver. It’s basically a receiver on a chip with three leads: ground, power, and digital output. The digital output is suitable for the GPIO pins provided by the Raspberry Pi. The header also provides 3.3 VDC and ground connections. This was easy enough to hook up using an old CD-ROM drive audio cable. On one end, I moved one of the three wires so that all three were next to each other (default is 2 populated, one blank, and one populated on the 4-pin connector). On the other end, I just removed the connector by pulling the pins soldered to the wires out of the housing. I then just plugged the individual pins into the appropriate pins on the Raspberry Pi header. Following the instructions on the XBMC wiki, it was pretty quick to teach the Raspberry Pi the codes for the different buttons on the remote. Now it’s easy to pause a show just by pressing a hardware button on an existing remote. Total cost: \$4.50 for the IR Reciever chip from Radio Shack.
