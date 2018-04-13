---
layout: single
author_profile: true
classes: wide
title:  "Adventures with Raspberry Pi"
date:   2015-07-27
categories: Automation
tags: ['archived', 'raspberry pi']
---
[Amazon][1] recently celebrated their Prime Day, offering **_amazing_** deals - most of which amounted to only a few dollars off their already reduced prices. However, there was one gem that I managed to pick up: the [Raspberry Pi 2 Ultimate Starter Kit][2] for only $55! I've been wanting to get a rpi2 for a while (ever since I saw the updated specs and after hearing that it supports embedded Windows 10). I have the first project idea in mind, but quickly found that I'll need to purchase a few more rpi2s for other projects. While I'm waiting for the first project's parts to arrive, I thought I'd throw together the comprehensive (as of now) outline for future projects.

# WiFi/Bluetooth audio streaming server

By cramming the rpi2 into an old stereo, I can upgrade an old piece of hardware and give it new life. There are a few products out there that can do streaming ([Volumio][3], [RuneAudio][4], etc) however, none of them natively support my music source of choice, Google Music - All Access. In theory [Pi MusicBox][5] does support Google Music (as well as SoundCloud, Spotify, and a whole host of other options). It doesn't natively appear to support bluetooth (which isn't _as_ critical if it can do Google Music), but I may be able to combine it with [this instructable][6] to do both Wifi and Bluetooth.

I've purchased (for a mere $7) a [Sanyo boombox][7] (Woo! Potato Quality Photo!), and after extensive hacked together testing, I realized the rpi2 has crap audio output. It needs an amp. And a DAC. Enter the [HiFiBerry AMP+][8] \- or rather, enter the AMP+ once it arrives from Switzerland. It requires a 12V power supply, and luckily the output from the stock Sanyo psu is right around 11.87V! I need to check the amp pull, but even if things don't work out, I can always hit up the trusty electronic store to grab a new PSU. Expect some further posts once the AMP+ arrives!

# Smart Power Switch

My cable modem keeps crapping out, removing my remote administration (and IRC capabilities). THIS WILL NOT STAND. Unfortunately, the only real fix appears to be unplugging the modem, waiting a bit, then plugging it back in (the power supply is probably on its way out after a few years of 24/7 uptime. I _could_ replace the power, but why not add to the Internet of Things? [This youtube video][9] shows how to use a rpi2 to control a power strip for remote power control. Perfect! My goal is to leverage [webiopi][10] for the fancy web interface to control the power outlet. (Alternatively this [Hackaday project][11] could fit the bill. But wait Brian, how will you power cycle the modem if the internet is down? Well voice in my head, a simple script on my linux jump box should handle that! Continuous pinging to google, when it fails, send the REST API call to power off the outlet, wait 30 seconds, and then power it back on. If stuff still doesn't work, call it a day (or maybe retry a few times - this requires experimentation). I need further research to see how to properly do this project safely...

# Smart Thermostat

The [Nest Thermostat][12] is cool, but really expensive. I'm thinking that with some effort, thinking, and parts, I can accomplish something similar for around $100-150. [There][13] are [several][14] sources [on][15] the [internet][16] that cover various ways to accomplish this. Some directly tie into the wiring for the thermostat, others seem to just trigger the button presses to adjust temperature. I really like the direction that Jeff took things - multiple room have independent temperature sensors. The rpi2 can take an average and use that to drive the thermostat or alternatively, use only one (e.g. I'm going to bed and want my room to be 74°F - and the AC/Heat runs until that sensor registers the proper temperature). I'm hopefully going to be moving into a rental shortly - depending on how much freedom I have to take things apart, this may end up being purely academic.

# Murphy Monitoring

I have an awesome dog named [Murphy][17]. I've always wondered how he would do if we left him out instead of crating him when we run errands. Maybe I could turn my rpi2 into a [CCTV Camera][18] and keep an eye on him? There's probably more options out there, I'll have to keep looking.

# Other fun stuff

I just found this [Micro UPS][19] - a battery backup solution for the rpi2. If I'm doing the Smart Thermostat or Murphy Monitoring (trademark pending) I'll need to have some battery backup in case of power outage - there have been some nasty stories about SD Card corruption when power loss and data writing are involved. I may leverage Arduino to make a home security system (one of the places I'm considering renting has a basement with unprotected windows). Maybe something like [this Arduino security sensor][20] could be cool.

[1]: http://smile.amazon.com
[2]: http://smile.amazon.com/Raspberry-components--Raspberry-Guide--Edimax-Adapter--8GB-Adapter--Case--Power/dp/B00PWYK2V6/
[3]: https://volumio.org/
[4]: http://www.runeaudio.com/
[5]: http://www.pimusicbox.com/
[6]: http://www.instructables.com/id/Turn-your-Raspberry-Pi-into-a-Portable-Bluetooth-A/
[7]: http://i.imgur.com/tSgN2mp.jpg
[8]: https://www.hifiberry.com/ampplus/
[9]: https://www.youtube.com/watch?v=8cPK8lh6oLI
[10]: http://webiopi.trouch.com/
[11]: http://hackaday.com/2015/02/11/wifi-controlled-power-outlets-with-raspberry-pi/
[12]: http://smile.amazon.com/Nest-Learning-Thermostat-2nd-Generation/dp/B009GDHYPQ
[13]: http://imgur.com/gallery/YxElS
[14]: https://www.raspberrypi.org/forums/viewtopic.php?f=37&t=95682
[15]: https://lifehacker.com/build-a-web-connected-thermostat-with-a-raspberry-pi-an-1670379446
[16]: http://wyattwinters.com/rubustat-the-raspberry-pi-thermostat.html
[17]: https://i.imgur.com/VmLYaQS.png
[18]: http://www.averagemanvsraspberrypi.com/2014/09/turn-raspberry-pi-into-cctv-security.html
[19]: http://www.piups.net/
[20]: http://www.freetronics.com.au/products/security-sensor-shield
