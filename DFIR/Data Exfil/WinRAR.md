# Description #

DISCLAIMER: As with all my research posts, this one is written in one-go and what is presented here is the result of my own testing. It does not cover any and every scenarios and/or situations one can think of. Hopefully, it'll still suit your needs and/or answer your questions.

This page holds my research and findings regarding the usage of WinRAR for creating "split" archives (partXXX.rar) in a context of collection for exfiltration.

# Context #

WinRAR is a known and popular trialware (with no trial limit, let's be honest) for data compression/decompression. It uses the popular RAR archive file format by default, but support many other (archive) file types.

It is also often used by threat actors during intrusion (e.g.: ransomware) to collect (compress, stage, etc.) data before exfiltrating it. There are multiple ways to use WinRAR for that, but the one that I'm going to be highlighting here today is the use of WinRAR's "split" mode. Basically, it allows you to create multiple RAR archives of the target folders or files to compress.

Before we go any further, I suggest that you read this article which show cases this mode and a few forensic artifacts related to it. This is the research we're going to expand on today:

https://www.axr1.net/winrar-split-archives-how-much-data-was-exfiltrated/

I found this article by simply googling: "winrar" + "volsize" and "winrar" + "arcname" and coming across the tweet below. You would be surprised by how little results (of interest) there are for this.

https://x.com/manictrains/status/1760625589659295883

# WinRAR Split Mode #

This being said, let's say you want to compress a 10 GB folder. By default, you can simply use WinRAR to archive it as a whole and end up with an archive that is maybe 7 or 8 GB big (abitrary values have been picked here).
However, WinRAR "split" mode let's you define the maximum size of each archives. For instance, if you were to go with a size of 2 GB, you may end up with let's say, 4 archives (4 archives x 2 GB = 8 GB).
Note that, as stated in the article above, there's no guarantee that the last archive will have the same size as the others, which are the "full" size chosen, because since it's the last archive, it may hold less data.
This makes it easier to upload these "part" files in parallel or on lower speed connections as you only have to upload smaller files at once.

# Setup #

Here's the setup I used for my tests:

- Windows 11 23H2 (22631.4169)
- WinRAR 7.1.0 (installed through the official website's downloads page)
- A 11.3 GB folder called Downloads
- A 4.24 GB folder named Tools

You may have guessed from these folder names and sizes that these may be actual folders I use. I work with what I have! :)

# Registry Keys #

From the article above, once again, we know that when we create a split RAR archive from the WinRAR GUI, the following registry keys are updated:

*HKCU\Software\WinRAR\DialogEditHistory\ArcName*
*HKCU\Software\WinRAR\DialogEditHistory\VolSize*

A new REG_SZ value with the name "0" is going to be created. The ArcName 0 value is going to have your resulting RAR archive name, and the VolSize 0 value is going to have the defined "split" size for that archive.
In my tests for instance, I created a RAR archive of the Tools folder, split in 500 MB archives.

![image](https://github.com/user-attachments/assets/d065594e-c029-4095-b7e8-3ba55891ee2a)
![image](https://github.com/user-attachments/assets/19b7dffb-21ce-4a29-aa13-664c004da8bb)


Now, one thing you may notice here is that the 0 value for ArcName only has the resulting archive name and NOT it's full path. In my case, via File Explorer, I manually went in the folder where the Tools folder was located (C:\Backup), right-clicked on it and selected "Add to archive..." from the WinRAR menu.
Which means that from this value alone, you cannot know where the archive was created. Or if it was created through the same steps, which folder "may" have been targeted by that compression operation. You would have to correlate this with other sources (e.g.: Recent Files and Folders).

As for the resulting part archives that were created, you'll notice that, as explained above, the last archive is smaller than the others since it holds less files.

![image](https://github.com/user-attachments/assets/aec60014-db9c-414b-95d7-c31ac4384d69)

Now, this is where the fun (or headache) begins.

# One, Two ... ? #

The article above is based on a single test. That means, a single WinRAR split archive was created and the forensic artifacts related to it were exposed. What happens however, if you create a 2nd split archive? This time, let's go ahead with the Downloads folder and use 2 GB archives.
We'll also use the full output path to see how the ArcName new value looks like.

![image](https://github.com/user-attachments/assets/48baeb57-5171-4c5e-a9d4-094d214fb071)

If we look at the ArcName Registry key again, we can see that a new value was created, "1". HOWEVER ... it does not hold our new archive path, but the previous one, "Tools.rar". As if the most recent value (the "Downloads.rar" archive, pushed the previous one up a number).
For those familiar with basically, any kind of Registry forensics, this is far from being a new concept to you.

![image](https://github.com/user-attachments/assets/6a73f961-09be-4187-ab00-e620ea7c7fcd)

So then you would expect the VolSize key to have been updated a similar way, right? I mean, 1 +1 ...

![image](https://github.com/user-attachments/assets/f9e691be-9396-4743-bdcb-ace0db2a72f3)

Doesn't seems to equal 2 ... ? Now that's funny, why wouldn't the VolSize key be updated with a new value and look like this:

- 0 - REG_SZ - 2 GB
- 1 - REG-SZ - 500 MB

And this is exactly (well, not totally, it was the opposite) what kicked off this research. On a recent IR where the threat actor used WinRAR to collect the data prior to exfil, the ArcName key had one (1) value only (the archive name), while the VolSize key had TWO (2) values (different ones at that).
How are we supposed to interpret that then?

# ProcMon Peek #

Since this was a test for research purposes, I had fired up Process Monitor (ProcMon) before creating the 2nd archive (Downloads.rar) to try and see what was happening.
From an ArcName perspective, we can see that WinRAR.exe successfully updates the 0 and 1 values with the data we saw in the Registry previously.

![image](https://github.com/user-attachments/assets/d763dc04-637a-4153-8a52-4e5a0c1f2179)

However, from a VolSize perspective, we get way less operations. None of them indicating that WinRAR.exe tries to update the values under the VolSize key.

![image](https://github.com/user-attachments/assets/1b9f690a-b519-49aa-8e17-8b05fe31a596)

# Third time's the charm as they say #

At this point, I tried to create a 3rd archive. I used the "Tools" folder as a target, but I defined the output RAR archive name as "Tools2.rar" and used a split size of 1 GB instead. Low and behold ...

![image](https://github.com/user-attachments/assets/46d7c4df-e99d-4e1f-8ef0-bb9fb03bc2b7)
![image](https://github.com/user-attachments/assets/38357fde-6546-41af-a383-a4ebcb03375e)

Now that operation successfully updated both keys with a new value! However, since there's a mismatch with the number of values on each key, how can we make an ArcName to VolSize association? I don't think we can.
The only difference I noticed here this time is that, the output RAR archive name was the filename only, and not a full path (like we C:\Temp\Downloads.rar).
So I went ahead and ran a 4th test with the "Downloads" folder: output RAR archive of "Downloads2.rar" (which means, inside the folder where I launched the WinRAR GUI from) and a size of 3 GB (I'm fancy like that). Low and behold ...

![image](https://github.com/user-attachments/assets/5999b30c-b1e2-41a2-b2cd-646fac0a3f43)
![image](https://github.com/user-attachments/assets/e6922b81-1e13-4b24-a4ac-1396f36ed9f8)

At this point, it's Sunday evening and I'm writing this blog post instead of enjoying the rest of my weekend (though some might say that I'm enjoying doing these tests, but that's beside the point).
Could the fact that specifying the full path of the output RAR archive really make it so that the VolSize key does not get updated?
So I went ahead with two (2) last tests:

- Test #1 - Creating a new split archive with a full path defined as output, C:\Temp\Downloads2.rar (used a file size of 750 MB if you are interested)
- Test #2 - Deleted all the values under the ArcName and VolSize keys (as to reset them) and run Test #1 again (with "clean" Registry keys this time)

The results? Otherwise, why else would you still be reading, obviously.

- Test #1 - Both the ArcName AND VolSize keys got properly updated with new values (What the ...)
- Test #2 - Both the ArcName and VolSize keys got properly updated with new values (this didn't surprise me however)

So at this point, I was forced to run another test. Create a new archive from the Downloads folder and have the output be C:\Temp\Downloads.rar (split size of 250 MB for those following us at home).
As for the results ... ?!? Both the ArcName and VolSize keys got properly updated with new values.

# Conclusion #

At this point, I ran many other tests, the same way I did them throughout this research and was not able to reproduce the original issue I faced on my test system. Nor was I able to reproduce the output from what I found in my IR.
Therefore, at this point, based on these tests, I only have one conclusion (more like, warning):

If you have a discrepancy in the number of values in the ArcName and VolSize Registry keys for WinRAR, you're in for a bad time. This means that you cannot exactly say, from these artifacts alone, what size would be the archive at value 0 under ArcName since a corresponding VolSize key value did not get created.
You can most likely assume at this point that in this scenario, the ArcName value 1 and the VolSize value 0 go together but even there... without fully understanding what's happening, I would be careful.

You would have to find your answer through the discovery and/or correlation of other artifacts.

If anyone has any idea of why this is happened or what the following scenarios means (how they can be achieved), please let me know because I am very curious about this whole situation:

- Discrepancy between ArcName and VolSize keys, where ArcName has more values than VolSize
- Discrepancy between ArcName and VolSize keys, where VolSize has more values than ArcName

This being said, it's Sunday night and I'll try to go enjoy the few hours of peace I have remaining before the next work week starts!
Till next time!
