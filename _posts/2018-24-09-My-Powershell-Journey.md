---
layout: post
title:  "My Powershell Journey"
date:   2018-10-01
author: Jonathan Moss
permalink: blog/my-powershell-journey.html
---

# My Powershell Journey

I've met a handful of folks who, when I talk to them about automating the boring stuff, they tend to shrug it off and say "I don't have time" or "I'm not a developer". This blog is for you (or if you want to know my journey of how I changed my career by learning Powershell).

I've been in IT for over 5 years, and have specifically been using Powershell at work for the last 3 years, and meaningfully in the last 2 years.  

## A Powershell Noob

Two years ago, I was using powershell to run a handful of "one-liners". Such as:

```powershell
Get-ADUser -Identity jmoss
```

Which returned some interesting data. With this in mind, I would simply find the data I'm looking for, and paste it into notepad or an email to my boss. What I didn't realize was this concept that **properties should be treated as objects.**

Once I realized that most of what I write in Powershell can be treated as an object, a light-bulb clicked. This happend about a year and a half ago, in early 2017. It was then where I realized that I had approached Powershell all wrong. It wasn't just another prompt to view an IP Address or list a phone number of an Active Directory user, it was the glue to automating everything I need and exporting it to anything I want.

It was at this point when I started to really kick it into high gear and force myself to use Powershell daily, and not just to get information randomly.

## Walking with Powershell

At an old company I used to work for, our IT department would receive html files from a vendor that included a bunch of tables with data in them, which needed to be exported to a txt file and *finally* imported into a SQL Server. The vendor had been sending us data in this format because their ERP only allowed for that (what year is it?!!??!?).

Our IT department was spending 1.5 hours *a day* parsing this data manually. There was clearly a problem that could be resolved with Powershell.

I used Powershell to parse the data in these HTML files using [ImportExcel](https://github.com/dfinke/ImportExcel)'s `Get-HTMLTable` cmdlet, grab the data, store it in variables, and then export to a TXT file using `Out-File` to a fileshare.

When all was said and done, I ended up creating a 50-60 line script that saved our department about 10 hours a week in manual work that was otherwise deemed "too difficult" to automate.

As far as I know, this code is still being used in production.

It was *ugly* code, but it worked, and gave me a ton of insight in how to use [PSCustomobject](https://kevinmarquette.github.io/2016-10-28-powershell-everything-you-wanted-to-know-about-pscustomobject/)'s and using Git to ensure my code was versioned-controlled.

## Crawling with Powershell

Around the time, I continued to learn more and more about Powershell by attempting to automate other things at work, developed a git repository with VSTS and integrated it into VS Code, shared that code with a co-worker, and encouraged my co-worker to develop his own script and add it to the collection.

I continued developing scripts to deploy a virtual machine in our environment and copy files to 60 physical servers. 

## Running with Powershell

*to be continued*

## Recap

To recapp, here's a few highlights of my journey and observations that I've made in the last 2 years:

- I'm nowhere close to "running with Powershell" but I intend to keep using Powershell to automate what I can.
- I've used Powershell almost daily at work for the last 2 years to automate business "tasks" and produce reports (mostly for myself or my team)
- Companies are hiring people who know Powershell. I've nearly doubled my Salary since first being exposed to Powershell about 3 years ago.
- Powershell can now be used on macOS, Linux, and Windows + it's open-sourced, meaning it will continue to get better.
- Every software service that Microsoft publishes (AD, DNS, DHCP, Azure, O365, Sharepoint, etc) can be managed/automated using Powershell (and Powershell Core!).

Overall, don't be afraid to just dive in and start solving a *real* problem at work with Powershell, and not just simply learn about the concepts (which is also extremely important).

Here are some ways to jump in and start growing in your career using Powershell:

- Join the [Powershell Slack](http://slack.poshcode.org/) and read what others are doing with it.
- Go on [Github](https://github.com/trending/powershell) and see what others are doing with Powershell.
- Go to the [Powershell](https://www.reddit.com/r/powershell) subreddit and read the posts where people talk about what they've done with Powershell each month (link [here](https://old.reddit.com/r/PowerShell/search?q=what+have+you+done&restrict_sr=on&sort=relevance&t=all)).