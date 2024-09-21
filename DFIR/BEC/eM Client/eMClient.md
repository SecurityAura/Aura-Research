# Description #

DISCLAIMER: This post was written on a late Friday evening after a long week of work so ... be wary of the typos and the likes.

This page holds my research and findings regarding the usage of eM Client (https://www.emclient.com/) to access a Microsoft 365 account.

# Context #

I recently took part in a BEC (Business Email Compromise) engagement where the threat actor leveraged eM Client to access the compromised user's mailbox. There's already a bit of research out there for eM Client, so before you go any further, I strongly recommend that you read these articles before going ahead with this one.

https://cybercorner.tech/malicious-usage-of-em-client-in-business-email-compromise/
https://www.aon.com/cyber-solutions/aon_cyber_labs/microsoft-365-identifying-mailbox-access/ (referenced in CyberCorner's article)

As with all BECs, one of the most interesting piece of information you want to look at are the MailItemsAccessed events, may they be from CloudAppEvents or OfficeActivities in Microsoft Sentinel or the Unified Audit Log (UAL) directly.
To sum it up, this event, which should now be available at any level of licensing following the Storm-0558 intrusion of Microsoft over the summer of 2023, basically tells you if an email was accessed or not. Even more so, it tells you if it was only accessed (Bind) or "downloaded" (Sync).
Per Microsoft own documentation on MailItemsAccessed events (https://learn.microsoft.com/en-us/purview/audit-log-investigate-accounts), a Sync event will ONLY occur for the Windows and Mac versions of Outlook.

So, where does that leave eM Client, which is obviously not Outlook

# Hypothesis & Testing #

In order to better understand how eM Client work, I proceeded to install it (free version, straight from the website) in a Windows 11 VM. I then signed in using my Microsoft 365 account. At the end of the setup, I made sure to check the option to automatically download any and all emails for "offline use" and this, no matter how old they are. Effectively, enabling some kind of "sync" if you want.
Next, I gave it a few minutes to grab the emails and went poking around the OfficeActivities events in Microsoft Sentinel. Sure enough, loads of MailItemsAccessed events came in, as expected.
However, as I kind of also expected after reading Microsoft's documentation, they all had the same MailAccessType: Bind. Meaning that if we were to stop there, we would assume that these emails were simply "opened" (accessed) and not downloaded.
So in order to test that theory, I closed eM Client, made sure none of its processes were running, disconnected the VM network card and opened it up again.
Sure enough ... all the emails that were previously downloaded were there.
Therefore, even though OfficeActivities/CloudAppEvents, and technically the UAL, report that eM Client only generated Bind MailItemsAccessed events, the reality, is that it downloaded them.

# Full Mailbox Sync OR ... partial sync ? #

Now, on my specific engagement, after parsing the MailItemsAccessed events from the UAL and getting a list of unique InternetMessageIds involved, I ended up with thousands of InternetMessageIds. The mailbox however, was over 10 GB. Forgetting email size for a moment, I wondered: can I somehow verify and/or validate that the eM Client ONLY sync'd the emails involved in the MailItemsAccessed events and NOT the full mailbox?
At this point, we already had one source of comparison, the UAL, but we needed another coming from eM Client itself.
So I started poking around its folders on my VM and like every good application, found it in %AppData%. As to get directly to the point, a SQLite database file with all the emails present (that were sync'd) exists at:

%AppData%\eM Client\GUID-or-CLSID-not-sure\another-GUID-or-CLSID-not-sure\mail_data.dat

![image](https://github.com/user-attachments/assets/f8889df4-65d8-4034-94e2-789307d360c0)

I downloaded and fired DB Browser (SQLite) and found out that the partHeader column actually holds the email header. Which means, it also holds the Message-Id field (Internet Message Id).
What if there was a way to:
- Connect to that SQLite database
- List the partHeader column of all rows
- Extract the Message-Id field value using regex
- Clean-up the results as to get a list of unique Internet Message Ids

Which is exactly what I proceeded to do, in PowerShell, with the following code. I used the following article to get my started on using SQLite through PowerShell, since I had never done it before:

```powershell
# Title: Extract-eMClientDatabaseInternetMessageIds.ps1
# Author: SecurityAura (@SecurityAura)
# Date: 2024/09/20
# Version: 1.0
# Description: Script that extracts the Message-Id for each email in eM Client's mail_data.dat SQLite database file and outputs the unique list of these ids in a CSV file

# Update that path for the System.Data.SQLite.dll you're going to use
[Reflection.Assembly]::LoadFile("C:\Users\Aura\Downloads\sqlite-netFx46-binary-bundle-x64-2015-1.0.118.0\System.Data.SQLite.dll")

# Make sure to update that path to your mail_data.dat file
$eMClientDatabasePath = "C:\Users\Aura\AppData\Roaming\eM Client\d1fa7ff3-d605-48c1-a589-7cfc0d86df8d\f3541067-80b4-4c6f-b6fe-681e684cf215\mail_data.dat"

$ConnectionString = "Data Source=$eMClientDatabasePath"
$SQLiteConnection = New-Object System.Data.SQLite.SQLiteConnection
$SQLiteConnection.ConnectionString = $ConnectionString
$SQLiteConnection.Open()

$Command = $SQLiteConnection.CreateCommand()
$Command.CommandText = "SELECT partHeader FROM LocalMailContents"
$Command.CommandType = [System.Data.CommandType]::Text
$Reader = $Command.ExecuteReader()

$Headers = @()
$InternetMessageIds = @()
$InternetMessageIdRegex = '(?s)(?<=Message-Id: <).*?(?=>)'

while ($Reader.HasRows){
    if ($Reader.Read()) {
        $Header = [char[]]$Reader.GetValue(0) -join ''
        $Headers += $Header

        $InternetMessageId = ($Header | Select-String -Pattern $InternetMessageIdRegex).Matches.Value
        $InternetMessageIds += $InternetMessageId
    }
}

$Reader.Close()

$UniqueInternetMessageIds = $InternetMessageIds | Sort-Object | Group-Object | Select-Object -Property Name
Write-Host -ForegroundColor Cyan "[*] Found $($UniqueInternetMessageIds.Count) unique InternetMessageIds in mail_data.dat"

# You can change the output path
$UniqueInternetMessageIds | Export-Csv -Path "eMClient_UniqueInternetMessageIds.csv" -NoTypeInformation
```

https://sqldocs.org/sqlite/sqlite-with-powershell/

Now, I had my second point of comparison: a unique list of Internet Message Ids (sorted by "name") from the eM Client itself.
Ah, and I forgot to share a very important piece of information previously. Parsing the OfficeActivities MailItemsAccessed events gave me 818 unique Internet Message Id.
Parsing the eM Client mail_data.dat database gave me, you guessed it, 818 unique Internet Message Id (but that's not really accurate, as we'll see in a few minutes).

So at this point, we were ready for the ultimate test: do both lists actually match each other?! Do we have the SAME Internet Message Id FROM the OfficeActivities AND the eM Client database?

# The moment of truth ... ? #

As I'm lazy with that kind of stuff, I went to my favorite online text comparison website (text-compare.com, but feel free to use whatever suits you) and threw both list.
Low and behold I ended up with a ... 99% match!
What? 99% isn't 100%? What is this?!
I know, that's what I told myself. So I scrolled up and down to find the culprits and ended up identifying two (2) of them. Which I'm guessing we could call exceptions? But since there's not a lot of research on that topic (I guess), it may be too soon to say.
The discrepencies actually came from two (2) emails that were sent through Microsoft services.
The first one was a notification email notifying me that someone shared a folder on SharePoint/OneDrive with me.
And the second one was the FIRST email that account had ever received, an email notifying me that an Office 365 Audio Conference licence has been assigned/turned on for my account.
For the first email, the Message-Id field properly existed in the header stored in the eM Client database. And I checked the header from the email itself, it was a match. However, for some reason, the Internet Message Id used in the OfficeActivities event was NOT the Message-Id, but the value of the X-MS-TNEF-Correlator field. I have no idea of what TNEF is and why OfficeActivites decided to log that value instead so if you have an explanation, please let me know!

https://learn.microsoft.com/en-us/office/client-developer/outlook/mapi/tnef-correlation

As for the second email, I, by pure luck to be honest, managed to find it in my Outlook and found out that the Message-Id was:

Message-ID: <[e6179a1ea0d8435a84af89d9b24e5c76-JFBVALKQOJXWILKCJQZFA7CTNN4XAZKDOBRXYVLTMVZEK3TBMJWGKZD4IV4G6U3NORYA====@microsoft.com]>

However, when looking at the partHeader for that email in the eM Client database, the Message-ID is:

Message-ID: <>

Why? Who knows. Maybe the brackets ([ ]) in the actual Message-ID are messing up the eM Client? Maybe I mishandled my char array conversion to string for the partHeader blob. But at this point, I was satisfied in identifying why both emails were "amiss" in the results.

# Conclusion #

While the test mailbox I used only had less than 1,000 emails, there are a few things we can gather from this test:
- eM Client has the ability to download ("sync") emails for offline use (which we already knew)
- When eM Client "access" and/or "sync" an email, MailItemsAccessed events are generated
- The MailItemsAccessed events for these have a MailAccessType of Bind, DESPITE eM Client ACTUALLY downloaded the full email for offline use ("syncing" it)
- eM Client has a SQLite database with header information on all the emails it holds for "offline use"
- All the Internet Message Id from the eM Client database are present in the MailItemsAccessed events from OfficeActivities (and therefore CloudAppEvents, and technically the UAL)

So my conclusion at this point, at least, until more research is done on the subject is:

If you work on a BEC where eM Client was used by the threat actor, and you have MailItemsAccessed events from eM Client for that period of time (that are NOT throttled, obviously), based on that test, it may be safe to assume that ONLY the emails whose Internet Message Id are present in the MailItemsAccessed events have been downloaded ("sync'd) and NOT the whole mailbox.
This is my own conclusion as of 2024/09/20. Should further testing be done on the matter, by me or someone else, I'll try me best to update that page with any new relevant information.

This being said, till next time!
