# Week-6
*20/07/2026*

## umask command

This command changes the default permissions that a file has when it's created. 
Whenever i make a script i have to change it's permissions with `chmod` to run it, i could change the default pemissions so that i dont have to do that but i don't think that will be a good idea and it's easy and quicj to change permissions of the scripts anyways.

## /etc/passwd

This file is a user account database.
Every line represents exactly one user account and has several fields separated by `:` :
- username:password:UID:GID:comment:home_directory:shell
- password: It just contains `x` meanind that the real password hash is stored in `/etc/shadow`
- shell: The user's default login shell. ex: `/bin/bash`

When i actually ran the command `cat /etc/passwd` intending to see how many user accounts there are on my laptop, i was shocked to find 49 different accounts.
Then i came to know that there are 2 different type of accounts i.e human accounts and system accounts. For example my laptop only contains two human accounts namely `user` and `tanish`.
System accounts exist to run specific background processes with limited privilages not meant for human login. This separation is for security purposes.

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

- Every non-root user has a directory in home folder. For example my home folder has `tanish` folder. When i run `cd ~` it taked me there.
- It also has it's own bin, dev, etc folders

### media directory

- Devices like CD, pendrive, etc are mounted under /media.

### mnt directory

- This is for external drives.
- Any external drive mounted will be shown here and i can access their content from here.

### proc directory

- It contains a hierarchy of special files which represent the current state of the kernel.

