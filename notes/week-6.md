# Week-6
*20/07/2026*

## umask command

This command changes the default permissions that a file has when it's created. 
Whenever i make a script i have to change it's permissions with `chmod` to run it, i could change the default permissions so that i don't have to do that but i don't think that will be a good idea and it's easy and quick to change permissions of the scripts anyways.

## /etc/passwd

This file is a user account database.
Every line represents exactly one user account and has several fields separated by `:` :
- username:password:UID:GID:comment:home_directory:shell
- password: It just contains `x` meaning that the real password hash is stored in `/etc/shadow`
- shell: The user's default login shell. ex: `/bin/bash`

When i actually ran the command `cat /etc/passwd` intending to see how many user accounts there are on my laptop, i was shocked to find 49 different accounts.
Then i came to know that there are 2 different type of accounts i.e human accounts and system accounts. For example my laptop only contains two human accounts namely `user` and `tanish`.
System accounts exist to run specific background processes with limited privileges not meant for human login. This separation is for security purposes.

## File Hierarchy Structure

### Root Directory

`/` is the root directory. Nothing exists above it. Only the root user has permission to modify all of it's contents.

### bin directory

It contains a lot of files and directories. 
When i ran `ls` command in this directory, i saw some files were marked blue. So i looked at one of the files by `ls -ld GET`, by doing this i discovered a new type of file.
Before the permissions it had the character `l` which means it's a symbolic link (symlink), It is a special type of file that acts as a pointer to another file or directory.
GET is a symlink to `lwp-request`.

The `/bin` directory contains all the system wide commands needed by all the users. I tried to print these commands but they are encrypted, so i couldn't actually see how these commands/scripts run.

### boot directory

- Stored all the files required for booting the system. It includes the GRUB bootloader configuration and other essential kernel files.
- In `/boot/efi/EFI` i found my PopOS boot folder, it's still there since i used to use it before ubuntu. I didn't delete it as it's harmless and doesn't take up too much space.

### dev directory

- This folder stored device files, these are special files that act as interfaces b/w hardware and software.
- When i did `ls -ld hpet` the output starts with `c`, which indicated that the file is a character device.
- Character devices transfer data byte by byte (a stream), like a keyboard, mouse, etc.

### etc directory

- It contains configuration files for system applications, users, services, and tools.

### home directory

- Every non-root user has a directory in home folder. For example my home folder has `tanish` folder. When i run `cd ~` it took me there.
- It also has it's own bin, dev, etc folders

### media directory

- Devices like CD, pendrive, etc are mounted under /media.

### mnt directory

- This is for external drives.
- Any external drive mounted will be shown here and i can access their content from here.

### proc directory

- It contains a hierarchy of special files which represent the current state of the kernel.
- In contains directories for every running process. The directories are named after their process id.
- A process can read it's own information from `/proc.PID/*` with no extra permissions.
- 

## Inode

-  Inode (Index Node) is a special type of data structure that is created when a file system is initialized.
- The total number of inodes determine the maximum number of files and directories that the file system can hold.
- Each file is identified by an Inode.
- An inode contains essential information (metadata) about a file.
- When a file system is created a fixed number of inodes is allocated.

Stored attributes:
- File size
- Permissions
- File Type
- Timestamps
- Owner and group

## Hard Links

- Each hard linked file is assigned the same inode number as the original, therefore they both reference the same physical location.
- Say that i did `ln original.txt hardlinked.txt` creating a hard link, both files point to the same content (physical location). So i can access the content by clicking any of these 2 files.
- Hardlinks cannot span across multiple file systems.
- Cannot create hard link for a directories (to avoid recursive loops).
- Command to create hard links `ln [original file name] [link name]`.
- if the original file is remove then the link will still show the contents as its pointing to the physical location and is not dependent upon the original file.

## Soft Links (Symbolic link)

- Each soft linked file contains a different inode value that points to the original file.
- Soft links contains the path for the original file and not the content like a hard link.
- They can be linked across different file systems.
- It can link to a directory.
- Breaks if the target is deleted. It becomes a "hanging link".
- It breaks even when the original file's name is changed.
- Command to create a soft link `ln -s [original file name] [link name]`.

## ss command

Started learning this, then i discovered i need to learn more about sockets first.

### sockets
It's a software endpoint facilitating bidirectional communication between process regardless of their location within the system or even beyond it's borders.
two types of sockets:
1. Network Sockets: These enable communication across networks using protocols like TCP/IP.
2. Domain Sockets: These facilitate communication between processes within the same system.

The ss command (Socket Statistic) is used to display detailed information about network sockets.
The output of the `ss` command is:
```
tanish@tanish-Ideapd-S340-14IIL:~/projects/linux-journey$ ss
RTNETLINK answers: Invalid argument
Netid      State        Recv-Q       Send-Q           Local Address:Port             Peer Address:Port             
u_str      ESTAB        0            0                            * 97273                       * 97274            
u_str      ESTAB        0            0                            * 87841                       * 91408            
```

- **State**: Indicated the current status of socket, such as LISTEN (waiting for connection) and ESTABLISHED (active communication between systems).
- **Recv-Q / Send-Q**: Shows the amount of data queued for sending and receiving.
- **Local Address:Port**: Displays the IP address and port number on your system where the socket is created or listening for connections.
- **Peer Address:Port**: Represents the remote system’s IP address and port number connected to your machine.

`ss -s` gives a summary of all socket types,
```
tanish@tanish-Ideapd-S340-14IIL:~/projects/linux-journey$ ss -s
Total: 1110
TCP:   22 (estab 12, closed 6, orphaned 0, timewait 5)

Transport Total     IP        IPv6
RAW	  1         0         1        
UDP	  16        10        6        
TCP	  16        10        6        
INET	  33        20        13       
FRAG	  0         0         0    
```




