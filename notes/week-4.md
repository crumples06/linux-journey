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

	
