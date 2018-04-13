---
layout: single
author_profile: true
classes: wide
title:  "Domoticz and home automation"
date:   2016-04-06
categories: Automation
tags: ['archived', 'raspberry pi', 'automation']
---
When I moved into my new townhome, one of the first purchases I made was a 2nd generation Nest thermostat. It's been working fine - the last software update improved auto-away to the point where it's now useful - but the sole thermometer is in the entryway on the first floor. This causes issues as the majority of the living space is either the bedrooms upstairs, or the living/tv room in the basement. As of now, Nest doesn't offer remote temperature nodes, so I have been looking at what other home automation tools I could use to make my smart thermostat even smarter.

My goal is to have remote temperature sensors that can occasionally (perhaps based on proximity/noise/other sensors) override the Nest and increase/decrease the temperature until the occupied portion of my house reaches the desired temperature.

# Domoticz

I stumbled upon [Domoticz][1] while researching open-source/inexpensive home automation software that can manage a Nest thermostat. It has an impressive list of supported hardware:

* RFXCOM Transceiver
* Z-Wave
* P1 Smart Meter
* 1-Wire
* Many more

Nest support is in its infancy - but I can at least see the current temperature/humidity readings, set Away Mode, and change the target temperature. The software itself has a decent web-based interface (that translates well to mobile).

![Main Page](/images/domoticz1.png)

Historical views are available for each sensor - something I'm interested in using to track the guest bedroom's ambient temperature this summer.

![Historical View](/images/domoticz2.png)

With the latest update, adjusting the thermostat finally works!
![Changing the Temperature](/images/domoticz3.png)

Other bonuses: Domoticz runs well on a Raspberry Pi (and even can directly utilize the GPIO ports), has a well documented [API][2]/[extensive wiki][3], and active [community forum][4]. I opted to use a spare Raspberry Pi 2 and the provided image.

# Remote Temperature Sensor

To address the issue of remote temperature sensors, I picked up a few [DHT22][5] sensors - one for the local Domoticz Raspberry Pi which resides in the basement, one for the Raspberry Pi Zero that will live up in the guest bedroom on the second floor.

These little digital temperature/humidity sensors are pretty accurate, and easy to use with just three pins + 10k resistor.

Wiring is relatively straightforward: 3.3v, ground, and gpio with a 10k resistor between ground & data. There is a decent write-up on the physical wiring on [privateeyepi][9].

## Software Configuration

Two things need to be configured in order for the temperature sensor to work: adding the hardware switch/sensor in the Domoticz software and create the underlying script to pull data from the sensor.

## Domoticz Settings

* Create a dummy switch (Setup > Hardware) ![Dummy Switch](/images/domoticz4.png)
* Create the associated sensor hardware when prompted - we want a temperature + humidity as sensor type (note the resulting IDX, this will be used in the script below).

## Sensor Script

For the script, do the following:

Installing Adafruit and WiringPi is simple enough. The bash script has a few sections that need customization.

### Connection Details

```
    # Domoticz server
    SERVER="localhost:8080"
```

The Domoticz server needs to be adjusted (for my guest bedroom I put in the FQDN or IP address):

### Hardware Details

```
    # IDX DHT
    # The number of the IDX in the list of peripherals
    DHTIDX="4"

    #DHTPIN
    # THE GPIO or connects DHT11
    DHTPIN="17"
```

Domoticz assigns an index to each device, that index needs to be updated for `DHTIDX`. The GPIO in which the dataport is connected needs to be updated for `DHTPIN`.

Running the script manually should both update the Domoticz sensor and output to console the resolved humidity and temperature readings. Add this to cron and you should be good to go!

# Other things

The "comfort" level is arbitrary at this point based exclusively on relative humidity - not dew point. as a result, you can have a situation like this:
![Is it dry or comfortable?](/images/domoticz.png)

How is 34% humidity comfortable, but 37% is dry? Also note that dew point is within ~1.5 degrees. This script could probably do with some calculations of dew point and humidity ranges to better calculate the comfort level.

[1]: https://domoticz.com/
[2]: https://www.domoticz.com/wiki/Domoticz_API/JSON_URL's
[3]: https://www.domoticz.com/wiki/Domoticz_Wiki_Manual
[4]: http://www.domoticz.com/forum/
[5]: http://smile.amazon.com/Sunkee-Digital-Temperature-Humidity-Replace/dp/B00CW82DHG/
[6]: http://projects.privateeyepi.com/home/home-alarm-system-project/temperature-and-humidity
