---
title: PowerShell & Concurrency
date: 2025-08-19 8:00:00 +0800
categories: [Automation, PowerShell]
tags: [powershell,ops,automation,concurrency]     # TAG names should always be lowercase
---
# PowerShell & Concurrency
PowerShell on its own is a **powerful** language. Many mistake it as only viable for small scripts, ad-hoc one liners, and general IT support. However, PowerShell goes far beyond that, being able to rival many non-compiled languages, with a little help.

## PowerShell Problems
PowerShell, while extensible, suffers from a few issues. The two major ones, in my eyes, being speed and multi-threading. While it is possible to set up scripts to process quickly, and even establish multi-threading, it can be difficult to manage, and turns simple scripts into complex scripts quickly. Additionally, it requires a high level of comfort within the language, and can make junior team members struggle to understand.

Lets take an easy example, looping through 500 endpoints, and verifying they have a remote support tool running. Let's pretend that tool is called *remotesupport.exe* for ease of understanding. We'll also assume that computer hostnames are stored in a .CSV file, endpoints.csv.

A quick script to easily stand up, would be the below script. Ingesting the CSV, looping through devices, and checking for a running process.
```powershell
#Import CSV of devices
$Endpoints = Import-Csv -Path 'C:\Examples\endpoints.csv'
#Create an empty array to store results from jobs
$QueryResults = @()
#Loop through items and query
foreach ($Item in $Endpoints) {
    #Try to invoke command against machine, getting process
    try {
        $InvokeResult = Invoke-Command -ComputerName $Item -ScriptBlock {
            if (Get-Process -Name remotesupport -ErrorAction SilentlyContinue) {
                return 'Running'
            }
            else {
                return 'Not Found'
            }
        } -ErrorAction Stop
        $QueryResults += [PSCustomObject]@{
            HostName = $Item
            Result   = "$InvokeResult"
        }	
    }
    catch {
        $QueryResults += [PSCustomObject]@{
            HostName = $Item
            Result   = 'Failed to connect to endpoint'
        }
    }
}
$QueryResults | Export-csv -Path "c:\Examples\QueryResults.csv" -NoTypeInformation
```
This script would run, and for 500 endpoints, likely finish within a day. However, this quickly becomes a scaling issue. We could streamline some aspects of the script, but what happens when we need to run this on 2,000 endpoints? What about 10,000?

## Concurrency
Rather quickly in my career, I ran into issues with needing to iterate over large data sets. In a large corporation, its not uncommon to need to work across 5,000 or more items on a regular basis, and need to do it quickly. Thankfully, PowerShell has native support for this.

### Jobs
Jobs will likely be the first stopping point for concurrency. This allows you to execute scripts in the background of your system, and retrieve data from these jobs once they complete. This is thankfully, a rather easy solution to stand up. Jobs are created by running *Start-Job* and passing a script block. Data can be retrieved by using *Receive-Job*, and they can easily be monitored via *Get-Job*. While it is tempting to spin up a job for every single computer we need to query, that is a quick road to overwhelming our hardware quickly. Often, I find its much more practical to create jobs based on a range of data, and allow that job to process that range. Allowing 10 jobs to process 50 entries each, resulting in 10x faster processing.
```powershell
#Import CSV of devices
$Endpoints = Import-Csv -Path 'C:\Examples\endpoints.csv'
#Create an empty array to store results from jobs
$QueryResults = @()
#Loop through items and query
for ($i = 0; $i -lt 5; $i++) {
#Define min and max items for each job to process
    if ($i -eq 0) {
        $min = $i * 100
    }
    else {
        $min = $i * 100 + 1
    }
    $max = $i * 100 + 100
    Write-Output "Starting query for items $min to $max"
    Start-Job -ScriptBlock {
        foreach ($Item in $Endpoints[$min..$max]) {
            try {
                $InvokeResult = Invoke-Command -ComputerName $Item -ScriptBlock {
                    if (Get-Process -Name TeabTip -ErrorAction SilentlyContinue) {
                        return 'Running'
                    }
                    else {
                        return 'Not Found'
                    }
                } -ErrorAction Stop
                $QueryResults += [PSCustomObject]@{
                    HostName = $Item
                    Result   = "$InvokeResult"
                }	
            }
            catch {
                $QueryResults += [PSCustomObject]@{
                    HostName = $Item
                    Result   = 'Failed to connect to endpoint'
                }
            }
        }
        $QueryResults | Export-Csv -Path "C:\Examples\query_results_$min-$max.csv" -NoTypeInformation
    } -ArgumentList $Endpoints,$min,$max
}
#Wait for all jobs to complete
Get-Job | Wait-Job
#Retrieve results from all jobs
Get-ChildItem -Path 'C:\Examples\' -Filter 'query_results_*.csv' | ForEach-Object {
    $CSVData = Import-Csv -Path $_.FullName
    $QueryResults += $CSVData 
}
$QueryResults | Export-Csv -Path 'C:\Examples\final_results.csv' -NoTypeInformation
#Clean up jobs
Get-Job | Remove-Job
```
This solution requires some more engineering, but overall, grants the ability to quickly scale this solution as needed. This solution could also be improved for some extra speed and hardware considerations. Such as modifying the script to only pass the needed endpoints, rather than all endpoints, as well as functions to streamline code. However, in its current state, it allows for easy scaling of the associated script.

## I ate 64 GB of RAM, now what?
Depending on your data sets, amount of scripts running, and frequency of running, you may not ever reach this point. However, its not uncommon to have massive amounts of scripts running on massive data sets, and scaling becomes another issue.

From this point, your options are relatively open. You can streamline scripts, ensuring they are running with minimal hardware usage. Additionally, you can scale additional servers, keeping track of their work loads and jobs. However, that becomes an issue of availability. Too often have I seen a server go down in an environment, and suddenly a swath of scripts are no longer running.

My preferred solution is to set up a centralized SQL table, allowing multiple servers to work off a data set in parallel, and store data within a central location. This provides the additional benefit of allowing reporting tools, and other scripts, to all operate on datasets at the same time. This topic gets complex quickly in its initial stand up, and will be written in another blog post.

## Thank you
Thanks for reading, and I hope this was useful, or at least interesting to read. This blog post mostly serves as an introduction to discuss using SQL and PowerShell together, which will be a future write up soon.