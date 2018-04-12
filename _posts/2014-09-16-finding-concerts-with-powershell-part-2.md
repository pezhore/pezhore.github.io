---
layout: single
title:  "Finding concerts with PowerShell - Part 2"
date:   2014-09-16
categories: "API Development"
tags: archived api powershell
---
I had a few minutes to get back to that [PowerShell code]({% post_url 2014-09-12-finding-concerts-with-powershell %}) that found upcoming concerts in St. Louis, and I added a bit. Skip ahead for the code if you most - the most notable feature upgrade includes finding the closest place that the target band is playing and displaying that show's City/Region and drive time.

## Geolocation / Google Distance API

Finding bands who were playing shows in a given city was pretty easy, but what if a band I want to see is playing in a suburb of my target city? Or what about if the band is playing one town over and I would totally be willing to drive the 30 minutes to see the show?

I already have a full list of all the events/venues where each band is playing, I just need to parse them to determine the closest place they're playing if they're not playing in my target City. I originally thought about comparing the target city's latitude and longitude to each event venue's latitude and longitude. Unfortunately, there's some serious math that goes into figuring out the distance between two points (See this [Stackoverflow discussion][1] to get an idea how crazy things can get). I started down the path of doing a simplified version of comparison to try and limit the calls to an API (Google Maps, Openmaps, Mapquest, etc), but when that failed, I just oped to use Google's [Distance Matrix API][2] as it allows 2.5k requests per day.

Integrating with Google's API was rather simple. I requested a new API key, then followed their documentation to craft an `Invoke-RESTMethod` call.

```language-powershell
$key = ""
$origin = "St. Louis, MO"
$destination = "Chicago, IL"
$result = Invoke-RestMethod "https://maps.googleapis.com/maps/api/distancematrix/json?origins=$origin&destinations=$destination&sensor=false&key=$key"
```

The resulting JSON includes various helpful bits of information like distance in KM: `$result.rows[0].elements[0].distance.value` and driving duration: `$result.rows[0].elements[0].duration.text`. Once I had each venue's distance in KM, a simple comparison enabled me to find the shortest distance, and record the appropriate information (Venue, City/Region, Driving Duration, etc).

## Final Tweaks

I added proper help syntax then saved the cmdlet as get-UpcomingConcerts.ps1. Other additions included a progress bar that details which band is currently being examined and a sorted output: first bands that are performing in the target city, then bands that are outside of the city. Various comments, verbose/debug statements, and other formatting fixes went into the finished product found below.

If you wish to use this code yourself, sign up for an API key from Google and enable the Google Distance Matrix API, replacing the value of $key with your API key. You may adjust your target band list in code, or provide it as an array to the -targetBands parameter.

## Get-UpcomingShows.ps1

```language-powershell
<#
    .SYNOPSIS
    Utilizes the BandsInTown API to search for a given band(s) in a given city, or
    surrounding cities

    .DESCRIPTION
    Iterates through all TargetBands and uses the BandsInTown API to determine if
    the specific band is playing in the given city. If they are, helpful
    information is provided.

    .EXAMPLE
    .get-UpcomingConcerts.ps1 -TargetBands "NOFX", "The Blaggards" -TargetCity "Boston"

    This will output the event details if NOFX or The Blaggards are playing in Boston, MA.

    .EXAMPLE
    .get-UpcomingConcerts.ps1 -TargetBands "NOFX", "The Blaggards"

    This will output the event details if NOFX or The Blaggards are playing in the default
    city, St. Louis, MO.

    .PARAMETER TargetBands
    An array of strings, this variable contains the target bands to search for.

    .PARAMETER TargetCity
    This is the target city for which concerts will be filtered.

    .NOTES
    Author: Brian Marsh
    Version: 1.0
#>

[CmdletBinding()]
param( [String[]]$TargetBands,
        $TargetCity = "St Louis")

BEGIN {
    # If no target band(s) were provided, let's do some defaults!
    if (!($TargetBands)){
        $TargetBands = "Mighty Mighty Bosstones", "Taking Back Sunday", "Chris Thile"
        $TargetBands += "NOFX", "Reel Big Fish", "Surburban Legends", "H2O", "Sick of it all"
        $TargetBands += "New Found Glory", "Ingrid Michaelson", "Dropkick Murphys", "Mad Caddies"
        $TargetBands += "Coheed and Cambria", "Placebo", "Foo Fighters", "Rush", "Bastille"
        $TargetBands += "Chuck Ragan", "Rancid", "Bad Religon", "Bouncing Souls"
    }

    $targetCityShows = @()
    $remoteShows = @()

    # Quick function to use Google APIs to determine distance, and compare all given events
    # for the geographically closest event
    function get-ClosestVenue{
        param ( $events )

        # Google API Key
        $key = ""

        # Set Default value for min distance
        $minDistance = 9999999

        # Create a custom PSObject to hold pertinent information
        $goodEvent = New-Object -TypeName PSObject
        $goodEvent | Add-Member -MemberType NoteProperty -Name "City" -value ""
        $goodEvent | Add-Member -MemberType NoteProperty -Name "Region" -value ""

        $goodEvent | Add-Member -MemberType NoteProperty -Name "Duration" -value ""

        $goodEvent | Add-Member -MemberType NoteProperty -Name "Datetime" -value ""


        # Iterate through each event
        foreach ($event in $events) {

            # Ignore venues outside the US for now (omit this line for international searches)
            if ($event.venue.country -match "United States" -or $event.venue.country -match "Canada") {

                # Set the destination to this Event's City and Region/state
                $destination = $($event.venue.city+", "+$event.venue.region)

                # We're starting in Target City
                $origin = $TargetCity

                # Get the destination JSON detailing from Origin to Destination
                $result = Invoke-RestMethod "https://maps.googleapis.com/maps/api/distancematrix/json?origins=$origin&destinations=$destination&sensor=false&key=$key"

                # Select only the kilometer's value
                $km = $result.rows[0].elements[0].distance.value

                # If our Minimum distance is greater than this distance, we have a shorter route!
                if ($minDistance -gt $km) {

                    # Set our new Minimum Distance
                    $minDistance = $result.rows[0].elements[0].distance.value

                    # Grab the appropriate information and save it to the custom PSObject
                    $goodEvent.City = $event.venue.city
                    $goodEvent.Region = $event.venue.region
                    $goodEvent.Duration = $result.rows[0].elements[0].duration.text
                    $goodEvent.Datetime = $event.datetime
                }
            }
        }

        # Output the Custom PSObject data
        $goodEvent | select City, Region, Duration, Datetime

    }


}

PROCESS {

    # Set a band counter
    $count = 0

    # Iterate through all the target bands
    foreach ($TargetBand in $TargetBands) {

        # Show some progress bar for this activity
        write-progress -activity "Search in Progress" -status "Band: $TargetBand" -percentcomplete ($count/$TargetBands.Length * 100)

        # Initialize variables
        $r = $events = $null
        $playingSoon = $false

        # Use Try/Catch to deal with exceptions here
        try {
            # Get the appropriate body for a RestMethod
            $r = Invoke-WebRequest "http://api.bandsintown.com/artists/$TargetBand/events.json?api_version=2.0&app_id=PowerShell" -ErrorAction Stop -Verbose:$false
        } catch {
            Write-Debug "Something went wrong... debug?"
        }

        # Retrieve all events that this band is performing in from BandsInTown.com
        $events = Invoke-RestMethod http://api.bandsintown.com/artists/$TargetBand/events.json -Body $r -Verbose:$false

        # Iterate through each event
        foreach ($event in $events) {

            # Grab this Event's Venue/City
            $myVenue = $event.venue
            $city = $myVenue.city

            # If this is happening in our target city
            if ($TargetCity -match $city){

                # Start output message accordingly
                $msg = "$TargetBand is playing in $TargetCity at $($event.venue.name) on $([Datetime]$event.datetime)."

                # Check ticket status and append appropriate information to the message
                if ($event.ticket_status -match "available") {
                    $msg += " Tickets are available now at: $($event.ticket_url)"
                } else {
                    $msg += " Tickets are unavailable."
                }

                # Output the message
                $targetCityShows += $msg

                # Set flag variable to true
                $playingSoon = $true
            }
        }

        # If the target band is not playing soon
        if ($playingSoon -eq $false){

            # Check if there are any events at all
            if ($events) {

                # Extra Verbose output
                Write-Verbose "$TargetBand is not playing in $TargetCity for the forseeable future."

                # Get the closest venue in the array of events
                $closest = get-ClosestVenue $events

                # Output the closest venue
                $msg = "$TargetBand is playing is $($closest.duration) drive away in $($closest.city+", "+$closest.region) on $([DateTime]$closest.datetime)"
                Write-Debug "Stahp?"
                $remoteShows += $msg

            } else {

                # Extra verbose output
                Write-Verbose "Sorry, $TargetBand is not playing a show in the forseeable future."
            }
        }

        # Increase band counter
        $count ++
    }
}
END {
    # Last minute debugging
    Write-Debug "Anything Else?"
    Write-Output "Shows in $targetCity"
    Write-Output $targetCityShows
    Write-Output ""
    Write-Output "Closest Shows"
    Write-Output $remoteShows
}
```

[1]: http://stackoverflow.com/questions/7120872/algorithm-to-calculate-nearest-location-based-on-longitude-latitude
[2]: https://developers.google.com/maps/documentation/distancematrix/
