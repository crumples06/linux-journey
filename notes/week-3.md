# Week-3
*08/06/2026*

## ps command
It reports a snapshot of the current processes.  
`ps` accepts 3 different types of options:
1. UNIX options, which are preceded by a dash.
2. BSD options, which must not be used with a dash.
3. GNU long options, which are preceded by two dashes.

Options of different types may be mixed, but conflicts may appear. From my experimentation, it's better to mix options of same type rather that checking if different types of options work together or not, it is much more simpler.

`ps` has a lot of options, some of the ones i consider important/interesting are:
- `-e` means every process. It displays all the processes (Shows just PID, TTY, TIME, CMD). The output can be controlled by adding more letters:
	- `-ef` : f means full format. Adds UID, PPID, C (%CPU), STIME.
	- `-eF` : F means extra full format. A superset of `-f`. Adds SZ (size in pages), RSS, PSR (which CPU core the process last ran on).
	- `-eo` : `o` is used to specify user defined format. after `-eo` i can add column names. ex: `ps -eo pid,c,cmd`. `cmd` shows the full command that is running. If i just want to know executable name and not the full command i can use `comm`, it is much cleaner.
	
- `--sort` is used to sort the output based a column. like `--sort c` will sort acending oder based on %cpu, and `--sort -c` will sort in decending order based on %cpu.

The best command using this that i will likely use a lot is : `ps -eo pid,c,comm --sort -c | head -30`

## htop
- Using `htop` is jst better than using `ps` commands as it provides a real time update and is more interactive and provides a lot of options.
- `ps` is used in scripts and `htop` if i'm actually investigating the system.

- RES (Resident Set Address) is an attribute in `htop` that is displayed in the table. It tells how much RAM is that process using.
- S means state. Types : R (Running), S (Sleeping), D (Uninterruptable sleep), Z (Zombie - process finished but parent hasn't acknowledged it yet), T (Stopped - Paused), I (Idle Kernel Thread)
- MEM% is RES as a percentage of total phycical RAM.
- CPU% is the percentage of one CPU core being used by this process right now. My laptop is an 8-core machine so it can go up to 800%.

## kill command
- `kill process_id` : It sends a signal to the process asking it to clean up open files, save state and exit gracefully.
- The process can ignore the signal. 
- It's always best to first try this than forcefully killing it.
- `kill -9 process_id` : It's not sent to the process, it is handled by the kernel. The kernel just distriys it completely
- Open files may not br flushed, temp files wont be deleted, etc. It is a last resort used when process can't be closed using any other way.

- `killall process_name` : Targets by process name. ex: `killall brave` or `killall -9 brave`.
- Useful when a program spawn many processes (like brave) an i dont want to kill them all using individual process ids.

## trap
`trap` lets a shell script intercept signals and run code when they arrive.
example: `trap "echo 'Caught SIGINT'; exit" SIGINT`
Now, if someone presses Ctrl+C while the script is running, instead of dying it prints the message first.

## &
Putting `&` at the end of a command make it run in the background immediately.
Example: `sleep 100 &`

## jobs
`jobs` lists all the background jobs in the current shell session.
- The number in the bracket is the job number 
- "+" = current job (default target for fg/bg)
- "-" = previous job
- States : Running, Stopped, Done

## Ctrl Z vs Ctrl C
When i wanted to stop a process i used to randomly use Ctrl Z or Ctrl C, i thought they did the same thing, but they are different.
- Ctrl+C (SIGINT) terminates the process 
- Ctrl+Z (SIGSTOP) pauses/suspends the process (can be resumed)
- Ctrl+Z doesn't kill any process, it is just frozen in memory waiting to be continued with `fg` or `bg`.


