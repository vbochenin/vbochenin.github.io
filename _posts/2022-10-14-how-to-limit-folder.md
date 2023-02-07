---
title: "How to create limited-size directory"
layout: post
author: vbochenin
tags:
  - linux
  - windows
categories:
  - how-to
---
## What an issue I've got

Today I've got a task to test (and fix) how the application behaves during a backup when it does not have a space on the device.
As result, there shouldn't be any new files once the backup is failed with "no space left" error.

## How did I investigate the issue

So far, I see the following options for how to reproduce the issue:
- to have enormous application data to backup, either on some shared machine or generate the content locally
- to create a limited-size directory and try to write a backup there

First is time-consuming to reproduce it locally, so let's start with creating limited-size directories.

I have two target environments for testing: Windows 10 and Debian-based Linux.
I've chosen Ubuntu 18.04 because I have the VM already installed.

So, let's make a backup for the newly installed application and check its size.
And empty backup is 8.5Mb. It is suspicious the app has an 8Mb backup from scratch, and I will write it down to fix later, but I have another issue so far.

### Windows

I haven't found a way to create a limited-size directory on Windows, but I can create a limited-size virtual disk and use it for my purposes.

1.  Go to Disk management

2.  Select your local disk and choose "Create VHD" in "Action".

    ![Create VHD](assets/img/posts/2022-10-14-how-to-limit-folder/windows-step-2.png)

3.  Choose Virtual Disk file location and set disk size. For my pupose I need 10Mb (8Mb for backup plus some extra for partitions and system information) and I need "Fixed size".  
    ![Choose virtual disk file](assets/img/posts/2022-10-14-how-to-limit-folder/windows-step-3.png)

4.  Initialize the new disk (choose Master Boot Record)

    ![Initialize the new disk](assets/img/posts/2022-10-14-how-to-limit-folder/windows-step-4.png)

5.  Create new simple volume.
    ![Create a new volume](assets/img/posts/2022-10-14-how-to-limit-folder/windows-step-5.png)

6.  Next... Next... Next... and disk is ready
    
    ![Disk is ready](assets/img/posts/2022-10-14-how-to-limit-folder/windows-step-6.png)

Wow!!! It took almost 7Mb from the original 10Mb for system data. WOW!!!

But... it is enough for my purposes

### Linux

1.  Create file with size you need

```
    ➜  ~ touch 10mbarea
    ➜  ~ truncate -s 10M 10mbarea
```

2.  Create file system with the file

```
    ➜  ~ mke2fs -t ext4 -F 10mbarea
    mke2fs 1.46.5 (30-Dec-2021)
    Discarding device blocks: done                            
    Creating filesystem with 10240 1k blocks and 2560 inodes
    Filesystem UUID: 03249ba7-6abc-4afb-a496-b5857f529859
    Superblock backups stored on blocks: 
            8193

    Allocating group tables: done                            
    Writing inode tables: done                            
    Creating journal (1024 blocks): done
    Writing superblocks and filesystem accounting information: done
``` 
3.  Create a disrectory you will use later

```
    ➜  ~ mkdir 10mbup
```

4.  Mount the file system to the folder

```
    ➜  ~ sudo mount 10mbarea 10mbup 
    [sudo] password for vbochenin: 
```

5.  And limited size directory is ready


```
    ➜  ~ df -h 10mbup              
    Filesystem      Size  Used Avail Use% Mounted on
    /dev/loop1      8.3M   14K  7.5M   1% /home/vbochenin/10mbup
```

Once limited size directories created, I may specify them as backup target direcotry and run the same empty backup.