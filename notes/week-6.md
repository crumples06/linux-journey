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

 
