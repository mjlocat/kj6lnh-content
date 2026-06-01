+++
title = "Robust Offsite Backups For Home Network"
date = 2016-01-09T15:56:25
draft = false
categories = ["Computers"]
slug = "robust-offsite-backups-for-home-network"
aliases = ["/2016/01/robust-offsite-backups-for-home-network/"]
+++

I’ve had some bad luck with computer hard drives over the last year. First, the hard drive in my home server failed. Then the hard drive on my laptop failed several months later. The backups for each had stopped working a while ago. I was able to recover some files using <a href="https://www.gnu.org/software/ddrescue/" target="_blank">ddrescue</a>. This is actually a really cool tool for doing a byte-for-byte copy of another disk, skipping over errors and going back to retry them later. Unfortunately for me, this was not only extremely time consuming (both times, I spent over a week on this process), but the blocks that couldn’t be recovered caused a lot of files to be irretrievable. \<sarcasm\>I had a lot of fun with the <a href="http://linux.die.net/man/8/debugfs" target="_blank">debugfs</a> command trying to get some files recovered.\</sarcasm\>

But, this post isn’t about trying to rescue data from a failed drive. This post is about making sure that’s not necessary because you had a good backup plan. I will be discussing the steps I have taken to implement backups from the various Windows and Linux machines on my network to my home Linux server as well as off-site backups in case your particular disaster isn’t limited to just your hard drive. You wouldn’t want to lose all those family photos, would you? Lucky for me, my wife keeps most of those on her computer.

Let’s start with the home server. I’m backing up every machine on my network here, so the server needs a lot of space. I ended up getting 2x 4TB drives and installing them in a RAID-1 setup (the drives are mirror copies of each other). This gives me a total capacity of 4TB and ensures that a single drive failure would not be disastrous. I can just keep running on one drive until I can install a replacement. Don’t wait too long on that replacement though. I’m not going to go over this process here as there are many guides for doing so already. The server is set up as a Samba file server for Windows to use and ssh for Linux to use.

For Windows backups, I went with <a href="http://www.2brightsparks.com/syncback/syncback-hub.html" target="_blank">SyncBack Free</a>. It’s fairly easy to use. I followed the steps in <a href="http://www.cnet.com/how-to/how-to-back-up-your-windows-computer-to-a-network-hard-drive/" target="_blank">this guide</a> to set it up to backup the Windows profile directory and the ProgramData directory. You need to create two separate backup profiles to do both. Point the backup target to the backup directory on the file server.

There were a number of files and directories that couldn’t be backed up. I went through the error log and excluded those files (they’re mostly automatically generated files that don’t really need to be backed up) so that the backup would show as successful. Finally, create a schedule for both profiles to run automatically each night. Make sure your computer will be turned on at the time it’s scheduled to run.

Next, I turned to the Linux backups. For this, I opted to use <a href="http://duplicity.nongnu.org/" target="_blank">duplicity</a>. It supports encryption, full, and incremental backups. I have it encrypting the backups on the way to the file server over sftp. I set up a schedule in cron to do an incremental backup 6 days a week. On the 7th day, it does a full backup and removes the backups from 3 weeks ago. Here’s what the crontab looks like:

```
0 23 * * 1,2,3,4,5,6 duplicity incremental --exclude-filelist exclude.list --encrypt-key <PGP Key ID> --ssh-options="-oIdentityFile=<path to ssh key>" /home/<my user id> sftp://<my user id>@<fileserver>//mnt/backups/<name of system being backed up> 2>&1
0 23 * * 7 duplicity full --exclude-filelist exclude.list --encrypt-key <PGP Key ID> --ssh-options="-oIdentityFile=<path to ssh key>" /home/<my user id> sftp://<my user id>@<fileserver>//mnt/backups/<name of system being backed up> 2>&1 && duplicity remove-all-but-n-full 3 --force --ssh-options="-oIdentityFile=<path to ssh key>" sftp://<my user id>@<fileserver>//mnt/backups/<name of system being backed up> 2>&1
```

The first line runs at 11pm Monday – Saturday and runs the incremental backup. The second line runs at 11pm on Sundays to do the full backup and remove backups older than 3 weeks. One thing to note when running this in cron: the SSH key and the PGP key should either have no passphrase or you will have to set some environment variables that contain the passphrase. Keep the PGP private key backed up separately as you will need this in order to decrypt the backup in the event you need to do a restore.

Finally, I’m storing my backups off-site. After some research, I settled on <a href="https://www.code42.com/products/crashplan/" target="_blank">CrashPlan</a>. For \$5/month (with 1 year purchase, \$6/month on a month-to-month plan), you get unlimited storage. For \$12.50/month, you can avoid all of the mess above and get unlimited storage for up to 10 computers all through CrashPlan’s interface. There’s a 30 day free trial. You need to use the Java interface to configure CrashPlan. If your file server is headless, <a href="https://support.code42.com/CrashPlan/4/Configuring/Using_CrashPlan_On_A_Headless_Computer" target="_blank">use the instructions here</a>. You can throttle the upload so you don’t saturate your upstream and make it difficult to use the Internet until your backup is complete.

Hope this helps anyone looking to do something similar. If not, at least I have some documentation on this process for myself 🙂
