---
layout: single
author_profile: true
classes: wide
title:  "New job, new PowerShell"
date:   2015-03-05
categories: ['Home Lab']
tags: archived PowerShell
---
I've started a new job (thus the relative quiet lately), and one of the first tasks was to perform information gathering via PowerShell. Luckily, this is old hat. Doubly luckily, I have some coworkers that know a ton about PowerShell! Today I'm going to talk about custom objects.

{% highlight powershell %}
$prop = @{
            'FirstName' = 'Brian';
            'LastName' = 'Marsh';
            'BirthDay' = (Get-Date -Date '1985-04-30 00:00:00Z').ToUniversalTime()
            'Age' = [Math]::Round((New-TimeSpan -Start $((Get-Date -Date '1985-04-30 00:00:00Z').ToUniversalTime()) `
                                                -End $(Get-Date)).Days/365.242,2)
            }

    $person = New-Object -TypeName PSObject -Property $prop
{% endhighlight %}
In the snippet above, we create a new hash table with the keys FirstName, LastName, BirthDay, and Age. In the same step we assign values each key. We then call the New-Object cmdlet and specify the properties of this new object to be the freshly created hash table.

An alternate method for creating a custom object is leveraging the Add-Member cmdlet. The code below creates an identical $person object:

{% highlight powershell %}
$person = New-Object -TypeName PSObject
$person | Add-Member -MemberType NoteProperty -Name 'FirstName' -Value 'Brian' 
$person | Add-Member -MemberType NoteProperty -Name 'LastName' -Value 'Marsh' 
$person | Add-Member -MemberType NoteProperty -Name 'BirthDay' -Value (Get-Date -Date '1985-04-30 00:00:00Z').ToUniversalTime() 
$person | Add-Member -MemberType NoteProperty -Name 'Age' -Value ([Math]::Round((New-TimeSpan -Start $person.BirthDay -End $(get-date)).Days/365,2))
{% endhighlight %}

Both of these are great, but there are some challenges. Using the $prop option results in unreliable ordering (e.g. the output object's properties may be in alphabetical order, or FirstName, BirthDay, LastName, Age - or completely different). The Add-Member method will keep the order, but is very wordy (476 characters vs 326 without tabbing/white space).

That's when I learned of ordered hashes:

{% highlight powershell %}
$prop = [ordered]@{
            'FirstName' = 'Brian';
            'LastName' = 'Marsh';
            'BirthDay' = (Get-Date -Date '1985-04-30 00:00:00Z').ToUniversalTime()
            'Age' = [Math]::Round((New-TimeSpan -Start $((Get-Date -Date '1985-04-30 00:00:00Z').ToUniversalTime()) `
                                                -End $(Get-Date)).Days/365.242,2)
        }

    $person = New-Object -TypeName PSObject -Property $prop
{% endhighlight %}

This will ensure that the resulting object will always have the properties in the order specified (FirstName, LastName, BirthDay, Age).

But that's not all! There has been a way to create a Custom Object using only the hash (and the properties remain ordered) since the PowerShell v3 release candidate.

{% highlight powershell %}
$person = [PSCustomObject]@{
                FirstName = 'Brian';
                LastName = 'Marsh';
                BirthDay = (Get-Date -Date '1985-04-30 00:00:00Z').ToUniversalTime()
                Age = [Math]::Round((New-TimeSpan -Start $((Get-Date -Date '1985-04-30 00:00:00Z').ToUniversalTime()) `
                                                    -End $(Get-Date)).Days/365.242,2)
                }
{% endhighlight %}

We've dropped the wordiness down to 278 characters while maintaining readability! 
