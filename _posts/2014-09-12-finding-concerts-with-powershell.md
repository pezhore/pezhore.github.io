---
layout: single
author_profile: true
classes: wide
title:  "Finding concerts with PowerShell and BandsInTown"
date:   2014-09-12
categories: ['API Development']
tags: archived api powershell
---
The move to St. Louis was a success - nothing was broken in transit, and I've settled in pretty well. Now the game begins again! Let's find suitable St. Louis alternatives to my favorite Rochester things. One area that is sorely lacking is upcoming concert schedules. My go-to source in Rochester was the [WBER Concert Schedule][1]. Unfortunately, it doesn't appear that there is an easily parsable alternative in St. Louis.

I stumbled upon [BandsInTown][2], but the forced integration with Facebook, and horrible perma-banner telling me to install the Facebook app quickly turned me off to the site. That's a true shame, because in general the information looked up-to-date and easily searchable/parsable. I toyed with the idea of seeing if they had an rss feed I could use - when lo and behold, it turns out they have a REST API! Time to dig out my PowerShell skills and do something with this.

First off, they require no authentication, but ask for an app_id. Fair enough! My app will be called "wat". With that in mind, here is a simple REST GET call to retrieve all events that NOFX is playing in the near future:

`http://api.bandsintown.com/artists/NOFX/events.json?api_version=2.0&app_id=wat`

That's pretty good, but let's throw that in PowerShell. I can use `Invoke-WebRequest` to get the proper api_version and app_id variables set, then use that for the `Invoke-RestMethod`.

{% highlight powershell %}
$r = Invoke-WebRequest "http://api.bandsintown.com/artists/NOFX/events.json?api_version=2.0&app_id=wat"
$events = Invoke-RestMethod http://api.bandsintown.com/artists/NOFX/events.json -Body $r
{% endhighlight %}

There appears to be a ton of information in that `$events` variable (namely an array of events. Parsing JSON is very easy with PowerShell, so let's take a look at the first event:

{% highlight powershell %}
c:scripting> $events[0]

ticket_url       : http://www.bandsintown.com/event/8144857/buy_tickets
venue            : @{name=Humboldt Park; longitude=-87.7017900; region=IL; latitude=41.9028510; city=Chicago; id=885936; url=http://www.bandsintown.com/venue/885936;
                    country=United States}
ticket_status    : available
artists          : {@{name=NOFX; mbid=dcaa4f81-bfb7-44eb-8594-4e74f004b6e4; url=http://www.bandsintown.com/NOFX}}
id               : 8144857
url              : http://www.bandsintown.com/event/8144857?artist=NOFX&came_from=67
on_sale_datetime : 2013-11-27T19:13:05
datetime         : 2014-09-12T19:00:00
{% endhighlight %}

This event is taking place on 9-12-2014, in Humboldt Park, IL. Tickets are still available. That's nice, but I'm not in Humboldt Park, I'm in St. Louis. I could individually call the rest of the 16 shows and look for St. Louis, but that's not a good use of my time. Let's provide this with a target city:

{% highlight powershell %}
$TargetCity = "St Louis"

$r = Invoke-WebRequest "http://api.bandsintown.com/artists/NOFX/events.json?api_version=2.0&app_id=wat"
$events = Invoke-RestMethod http://api.bandsintown.com/artists/NOFX/events.json -Body $r

foreach ($event in $events) {
    $myVenue = $event.venue
    $city = $myVenue.city
    if ($TargetCity -match $city){
        Write-Output "NOFX is playing in $TargetCity $($event.datetime)"
    }
}
{% endhighlight %}

Let's give that a shot. . . Oops, no output. I guess they're not playing in St. Louis. Rather than guess if the process failed or not, I'll put in a helpful message driven by the new `$playingSoon` variable.

{% highlight powershell %}
$TargetCity = "St Louis"
$TargetBand = "NOFX"
$playingSoon = $false

$r = Invoke-WebRequest "http://api.bandsintown.com/artists/$TargetBand/events.json?api_version=2.0&app_id=wat"
$events = Invoke-RestMethod http://api.bandsintown.com/artists/$TargetBand/events.json -Body $r

foreach ($event in $events) {
    $myVenue = $event.venue
    $city = $myVenue.city
    if ($TargetCity -match $city){
        Write-Output "$TargetBand is playing in $TargetCity $($event.datetime)"
    $playingSoon = $true
    }
}

if (!($playingSoon)) {
    Write-Output "Sorry, $TargetBand is not playing in $TargetCity for the forseeable future."
}
{% endhighlight %}

You may have noticed that I also parameterized the $TargetBand in that last sample - it enables me easily do things like search for multiple bands:

{% highlight powershell %}
$TargetBands = "Mighty Mighty Bosstones", "Taking Back Sunday", "Chris Thile", "NOFX"
$TargetCity = "St Louis"
foreach ($TargetBand in $TargetBands) {
    $r = $events = $null
    $playingSoon = $false

    $r = Invoke-WebRequest "http://api.bandsintown.com/artists/$TargetBand/events.json?api_version=2.0&app_id=wat"

    $events = Invoke-RestMethod http://api.bandsintown.com/artists/$TargetBand/events.json -Body $r

    foreach ($event in $events) {
        $myVenue = $event.venue
        $city = $myVenue.city
        if ($TargetCity -match $city){
            Write-Output "$TargetBand is playing in $TargetCity $($event.datetime)"
            $playingSoon = $true
        }
    }

    if ($playingSoon -eq $false){
        Write-Output "Sorry, $TargetBand is not playing in $TargetCity for the forseeable future."
    }
}
{% endhighlight %}

That's it for today. In the next iteration, I'll be looking to integrate with something like MapQuest's API to set alternative cities - maybe I'd be willing to drive to Chicago for NOFX, but I definitely wouldn't for Taking Back Sunday. I'll also be cleaning up the output to make it more portable/emailable.

[1]: http://summerschool.monroe.edu/wberweb/Wber/concerts.asp
[2]: http://www.bandsintown.com
