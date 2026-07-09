# Week-4
*21/06/2026*

## pgrep command
It gives the pid of the specified process.
Ex: `pgrep brave` outputs all the pids of the brave processes running currently.
`pgrep -c brave` outputs the number of pids instead of printing all the pids individually.
`pgrep -l brave` outputs the pid number along with the process number.

## zombieProcesses
I created a simple script that prints any zombie processes pid,s,and comm if they exist.
The main code in that script is `ZOMBIES=$(ps -eo pid,s,comm | awk '$2 ~ /^Z/ { print }')`
If no zombie processes ten it outputs "No zombie processes found".

## systemd
It is a system and serivce manager for linux OS. It's the first process that starts when the kernel boots. It's responsible for initializing user space, managing system services, and handling many core OS tasks.
Evrything in systemd is a unit. Unit is the fundamental building block - every service,socket,device,mount point,timer, or even a group of units is represented as a unit.
Each unit has a distinct suffix in it's file name. Ex: `.service`, `.socket`, `.target`, `.timer`, `.mount`, etc.
Units can declare how they relate ot each other:
- `Wants=`/`Requires=` - dependency strength
- `After=`/`Before=` - order without implied dependency
- `Conflicts=` - prevent simultaneous action

 
