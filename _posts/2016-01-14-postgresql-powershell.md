---
layout: single
author_profile: true
classes: wide
title:  "PostgreSQL + PowerShell"
date:   2014-07-01
categories: Automation
tags: archived powershell postgresql
---

I've been absent for quite a while. The home lab automation has taken a back seat to unpacking and setting up IKEA flat-pack furniture.

That is still ongoing, but let's focus on some more fun things. I need to get details from our UCS Environment into an inventory database - this database will be used to compare to the Cisco Compatibility Matrix and determine the potential impact of future upgrades. This will expand beyond UCS (and into VMware) - so leveraging my past Excel reports will not be powerful enough.

Initially, I'm looking at leveraging PostgreSQL because:

* Free
* Plethora of information on the inter-web-nets
* It's new to me (and who doesn't like learning about new stuff?)
* I've been told I have to use PostgreSQL by our DB team (rendering the prior three points somewhat moot).

# The Basics

You need to install the PostgreSQL ODBC driver from [here][1] \- but apparently don't need to actually configure it for anything. PowerShell will leverage .NET to handle the connectivity details.

My data is pulled from the [Cisco UCS HealthCheck v2.3][2]. Note that the script will connect to multiple UCS Managers (woo!), and will output an HTML (boo!). I realized that I didn't need all of the data the script gathers - there's quite a bit of performance metrics that take a ton of time to gather - so I stripped out the bits I didn't need/want and dumped the result prior to the script converting everything to HTML.

My (somewhat embarrassing) hack job is available on [Github][3] \- note that once I get this database stuff figured out, I'll be reworking that gist to make it better.

The object that is generated from the hacked together gist is a nested hash (several layers deep) Note that each of the inventory items (blades, IOMS, FIs, etc) are arrays. Additionally, when more than one UCS Manager is evaluated, `$ucsdatadump` has one key per UCSM.

The data structure looks a little like this:

```
$ucsdatadump["UCS Manager1"]["Inventory"]["Blades"][0..n]
                                                    [Dn]
                                                    [Serial]
                                                    [CIMC]
                                                    [Memory]
                                                    [Model]
                                                    [Adapter]
                                                            [Serial]
                                                            [Name]
                                                            [Model]
                                                            ...
                                                    ...
                                         ["IOMs"][0..n]
                                                 ["Dn"]
                                                 ["Fabric_Id"]
                                                 ["Backup_FW"]
                                                 ["Running_FW"]
                                                 ...
                                         ["FIs"][0..n]
                                             ["Dn"]
                                             ["Kernel"]
                                             ...
                                         ["Chassis"][0..n]
```

Originally, I was put off by the hash(hash(hash(hash))), but it actually works quite well for programmatically dumping into a database.

At this point, I'm only running the code via PowerShell ISE - not as a standalone ps1 file (although there's probably no good reason _not_ to make this into a standalone script).

# Helper Functions

In order to create the database table's columns, I'll need to convert a passed object's PowerShell type to the corresponding postgreSQL type. The function below can be expanded as necessary, but for my reporting generally a text/int is sufficient.

{% highlight powershell %}
function Get-PostgresColumnType
{
    [CmdletBinding()]
    param ($obj)

    # Get this object's PowerShell Type
    if ($obj)
    {
        $TypeName = $obj.getType().Fullname
    }

    # Convert the PowerShell Type to PostGreSQL Column Type
    switch -regex ($TypeName)
    {
        ".*String"    { $columnType = "text" }
        ".*UInt32"    { $columnType = "bigint" }
        ".*ArrayList" { Write-Verbose "ArrayList, cannot convert to PostgreSQL Type"}
    }

    # Return the column type
    $columnType
}
{% endhighlight %}

Next I need to interact with the database - functionality that can generally be broken into two categories: querying to get data, and querying to impact the database without expecting anything in return. I'm using two additional helper functions for this fundamental interaction - both leverage the newly installed ODBC driver. Note the default values provided for the db connection details (those would need to be updated)

{% highlight powershell %}
# This is for PostgreSQL queries that return stuff
function Get-ODBCData{
    param(
            [string]$query,
            [string]$dbServer = "10.1.1.64",   # DB Server (either IP or hostname)
            [string]$dbName   = "inventorydb", # Name of the database
            [string]$dbUser   = "postgres",    # User we'll use to connect to the database/server
            [string]$dbPass   = "postgres"     # Password for the $dbUser
            )

    $conn = New-Object System.Data.Odbc.OdbcConnection
    $conn.ConnectionString = "Driver={PostgreSQL Unicode(x64)};Server=$dbServer;Port=5432;Database=$dbName;Uid=$dbUser;Pwd=$dbPass;"
    $conn.open()
    $cmd = New-object System.Data.Odbc.OdbcCommand($query,$conn)
    $ds = New-Object system.Data.DataSet
    (New-Object system.Data.odbc.odbcDataAdapter($cmd)).fill($ds) | out-null
    $conn.close()
    $ds.Tables[0]
}
}
{% endhighlight %}

{% highlight powershell %}
# This is for PostgreSQL queries that don't return stuff
function Set-ODBCData{
    param(
            [string]$query,
            [string]$query,
            [string]$dbServer = "10.1.1.64",   # DB Server (either IP or hostname)
            [string]$dbName   = "inventorydb", # Name of the database
            [string]$dbUser   = "postgres",    # User we'll use to connect to the database/server
            [string]$dbPass   = "postgres"     # Password for the $dbUser
            )

    $conn = New-Object System.Data.Odbc.OdbcConnection
    $conn.ConnectionString= "Driver={PostgreSQL Unicode(x64)};Server=$dbServer;Port=5432;Database=$dbName;Uid=$dbUser;Pwd=$dbPass;"
    $cmd = new-object System.Data.Odbc.OdbcCommand($query,$conn)
    $conn.open()
    try
    {
        $cmd.ExecuteNonQuery()
    }
    catch
    {
        Throw "BAD QUERY: $query"
    }
    $conn.close()
}
{% endhighlight %}

# Nitty Gritty

There are two key things that need to be accomplished - the table needs to be created (if it isn't already), and the table needs to be populated/updated.

Assuming our hash table layout specified above, we know that our various UCS Managers are specified as keys in `$ucsfulldump.keys` \- we cycle through each UCS Manager with a foreach loop. We also know that there are various tables that we'd like to dump into the database (Blades, IOMs, FIs, etc) - specified as keys in `$ucsfulldump[$ucs].Inventory.keys` \- so we loop through those too.

{% highlight powershell %}
foreach ($UCS in $ucsfulldump.keys)
{
    # Get all the keys in this particular level of things
    foreach ($tableName in $ucsfulldump[$ucs].Inventory.Keys)
    {
        ...
{% endhighlight %}

## Table Creation

At this point in the script we are now at a level something similar to `$ucsfulldump[UCSM1].Inventory.[Blades]`, below which is an array of UCS Blade hashes. This information will be captured in a database table "UCS_Blades" that will be created if it doesn't exist already.

Below, we first query the database to see if the table exists, if it doesn't - we need to create it. Pull the first object off of the array `$myObj` (which is in itself a hash), and begin to craft the create table PostgreSQL query - retrieving the column type from our helper function based on the `$myObj[$key]` PowerShell object type. Lastly, we try to create the table.

{% highlight powershell %}
if (!(Get-ODBCData -query "SELECT relname FROM pg_class WHERE relname='UCS_$tableName'"))
{
    # Get the first object in this value's array
    $myObj = $ucsfulldump[$ucs].Inventory["$tableName"] | select -first 1

    # Build the CREATE TABLE query
    $newTable = "CREATE TABLE `"UCS_$tableName`"(`n"
    foreach ($key in $myObj.Keys)
    {
        $type = Get-PostgresColumnType $myObj["$key"]

        if ($type)
        {
            $newTable += "`t`"$key`" $type,`n"
        }

    }
    $newTable += "`t`"UCSM`" text,`n"
    # Strip off the last new line and "," characters add a new line, close the parenthesis and add a semicolon
    $newTable = $newTable.Substring(0, $newTable.Length -2)+"`n);"

    # Create the new Table
    try
    {
        $output = Set-ODBCData -query $newTable
    }
    catch
    {
        Write-Host -ForegroundColor Red "ERROR! $($_.Exception)"
    }
}
{% endhighlight %}

On a side note, I'm stoked about this script - but I'm super proud of this particular bit of logic. If an additional inventory hash is added, the code does not need to be altered in order to create a new table. Likewise, I can use this code for other hashtable objects (if properly created) to create new tables - all without hardcoding practically anything - save for the "UCSM" column and "UCS_" table prefix.

Also for the purposes of this script, UCS object databases will have a compound primary key - UCSM and Dn. I'm manually creating this constraint for now, but will probably add something to the script to deal with primary keys in the future.

## Populating the Table

I discovered an interesting problem at this point - as the hash object may contain key-value pairs that are not text or numbers, we can't just dump everything in the object into the database. To determine which key-value pairs will be pushed to the database, we query the database's schema and parse the resulting column names.

{% highlight powershell %}
# Query for this table's columns
$query   = "SELECT * FROM information_schema.columns WHERE table_schema = 'public' AND table_name = 'UCS_$tableName';"
$columns = Get-ODBCData -query $query

# Dump the column names into an array
$props   = $columns.column_name
{% endhighlight %}

As those names were derived from the first object's hash keys we should be able to easily pull the information and pull only the `$props` from each inventory object (Blades, etc).

Now that we know which properties correspond to the database table's columns, we can loop through each item and do one of three actions: **Insert**, **Update**, or **Nothing**. If the item doesn't exist in the database, we craft an insert statement. Alternatively, if the item exists already in the database (again, based on that compound primary key), we check to see if the database matches the item. If it does, nothing happens. If there are differences, only the differences are updated.

{% highlight powershell %}
Foreach ($item in $ucsfulldump[$ucs].Inventory["$tableName"])
{
    $dbQuery = $Null

    # Check of this thing exists (Primary Key = Dn & UCSM)
    $exists = Get-ODBCData -query "SELECT * from `"UCS_$tableName`" WHERE `"Dn`" LIKE '$($item["Dn"])' AND `"UCSM`" LIKE '$ucs'"
    if ($exists)
    {
        # Assume it's identical
        $identical = $true
        $diffProps = @()
        #write-host -ForegroundColor Green "This already exists"
        # Go through each property in this item, and compare properties
        foreach ($prop in $($exists |gm -MemberType Properties |select -ExpandProperty Name))
        {
            # If this item, and the existing entry are not identical AND the property is not UCSM (UCSM only exists in the database)
            if (($item[$prop] | Select -First 1) -notlike $exists.$prop -and $prop -notmatch "UCSM")
            {
                $identical = $false
                $diffProps += $prop
            }
        }

        # If we're not identical, we need to update
        if (!$identical)
        {
            #Update this shit.
            Write-host -ForegroundColor Magenta "This item already exists, however properties are different. We need to update things for $ucs >> $tablename >> $($item["Dn"]): $diffProps"
            $dbQuery = "UPDATE `"UCS_$tableName`" SET "
            foreach ($diffProp in $diffProps)
            {
                $dbQuery += "`"$diffProp`" = '$($item["$diffProp"])',"
            }
            $dbQuery = $dbQuery.Substring(0,$dbQuery.Length - 1) + " WHERE `"Dn`" LIKE '$($item["Dn"])' AND `"UCSM`" LIKE '$ucs'"

        }
    }
    # This item doesn't exist in the database, time to add it
    else
    {

        # Start off the insert query specifying the table name and properties/columns
        $dbQuery = "INSERT INTO public.`"UCS_$tableName`" (`"$($props -join '", "')`") VALUES ('"

        # Go through each property and add this Item's corresponding property
        Foreach ($property in $props)
        {
            if ($item["$property"] -match "'")
            {
                Write-Host -ForegroundColor Green "Ooops, found a `"'`" in: $($item["$property"])"
                $dbQuery += $($item["$property"]).Replace("'","''")+"', '"
            }
            if ($item["$property"].count -gt 1)
            {
                write-host "SHIT MORE THAN ONE! $($item["$property"]), using the first one"
                $dbQuery += [string]$($item["$property"] | Select -First 1)+"', '"

            }
            else
            {
                $dbQuery += [string]$($item["$property"])+"', '"
            }
        }



        # Bad addition leads to an extraneous ", '", take that off and add the closing parenthesis
        $dbQuery = $dbQuery.Substring(0,$dbQuery.Length - 6)
        $dbQuery = $dbQuery + " '$UCS');"

        #Double Check the insert query
        #$insertQuery
    }

    if ($dbQuery)
    {
        $output = Set-ODBCData -query $dbQuery
    }
{% endhighlight %}

At this point, we have a pretty solid script that works well and is **_relatively_** generic - at least generic enough that we can take it and massage it to take in VMware inventory (see a future update). Unfortunately, I need more information to complete my inventory database - specifically the Blade VIC information. In the next post, I'll show how I handle this wrinkle. In the meanwhile, the script up to this point is available as a gist on my [github][4].

[1]: http://www.postgresql.org/ftp/odbc/versions/msi/
[2]: https://communities.cisco.com/docs/DOC-52398
[3]: https://gist.github.com/pezhore/d2816dab34a74c8fa6b6
[4]: https://gist.github.com/pezhore/a74c7cefc56cacd9ab67