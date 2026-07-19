# Week-5
*13/07/26*

## systemctl command
Started learning the `systemctl` command. 
It is a utility command used to control systemd system and service manager. 

I can see the status of services using `systemctl status <service>`. It gives a lot of information.
For example i did `systemctl status gdm` and i got output:

	● gdm.service - GNOME Display Manager
	     Loaded: loaded (/usr/lib/systemd/system/gdm.service; disabled; preset: enabled)
	     Active: active (running) since Mon 2026-07-13 20:10:30 IST; 2h 59min ago
	 Invocation: 389dd4d58cde4959b7726d7d31caa1b8
	    Process: 1915 ExecStartPre=/usr/share/gdm/generate-config (code=exited, status=0/SUCCESS)
	   Main PID: 1930 (gdm3)
	      Tasks: 5 (limit: 22931)
	     Memory: 6.6M (peak: 10M)
		CPU: 164ms
	     CGroup: /system.slice/gdm.service
		     └─1930 /usr/sbin/gdm3

	Jul 13 20:10:30 tanish-Ideapd-S340-14IIL systemd[1]: Starting gdm.service - GNOME Display Manager...
	Jul 13 20:10:30 tanish-Ideapd-S340-14IIL systemd[1]: Started gdm.service - GNOME Display Manager.
	Jul 13 20:10:31 tanish-Ideapd-S340-14IIL gdm-launch-environment][2021]: pam_unix(gdm-launch-environment:session): session opened for user gdm-greeter(uid=60578) by (uid=0)
	Jul 13 20:11:21 tanish-Ideapd-S340-14IIL gdm-password][3390]: gkr-pam: unable to locate daemon control file
	Jul 13 20:11:21 tanish-Ideapd-S340-14IIL gdm-password][3390]: gkr-pam: stashed password to try later in open session
	Jul 13 20:11:22 tanish-Ideapd-S340-14IIL gdm-password][3390]: pam_unix(gdm-password:session): session opened for user tanish(uid=1000) by tanish(uid=0)
	Jul 13 20:11:22 tanish-Ideapd-S340-14IIL gdm-password][3390]: gkr-pam: unlocked login keyring
	Jul 13 20:11:28 tanish-Ideapd-S340-14IIL gdm3[1930]: Gdm: Child process -2158 was already dead.
	
Exlpaination of some things in the putput above:
- in the 2nd line, the disabled means the service is not configured to start automatically at boot. (I never started this service myself, so it probably was started by some other process)
- Main PID is the primary process of the service, in this case it's called gdm3
- Tasks : 5 means that there are 5 threads/processes currently inside the service's cgroup (including children). [cgroup or control group oragnizes processes into hierarchial groups. It basically groups together processes, that is why i dont need to put `gdm3` in the command and i can just put the name of the group i.e. `gdm`. This cleared my doubt that i have had for a long time about how processes and services are fetched even i dont pu their full process name in the command turns the group names work and are much more simpler).
- The lines containing dates are naturally recent log entries of the service.


`systemctl is-active gdm` and `systemctl is-failed gdm` tell me whether the service (in this case gdm) is active or not. Both commands will give the same output.
`systemctl is-enabled` telld whether the service (gdm) is enabled or disabled. It's just a one word output.	
  
`systemctl list-units` lists all the loaded units. The list was way too big, impractical for any meaningful understanding.
So it's better to be specific i think. 
`systemctl list-units --type=service` will only list loaded units which are of the type service. When i executed this command i still thought the output is way too large.
So i went more specific, `systemctl list-units --type=service --state=running` lists all loaded services which are running. I can put `--state=failed` to get the failed services.

I can also start/stop/restart/reload services using the systemctl command. 
`sudo systemctl start <service>` will start a disabled service.
- restart stops and starts the service
- reload tells the service to re-read it's configuration files without a full interruption. Not all services support this though.

`systemctl` integrates with `journald`. Use `journalctl`:
- `journalctl -u <service>` - I ran `journalctl -u bluetooth` and it showed me logs starting from may 22. It's hard to scroll to a speicific date if i want to see logs of that date. I will probably have to make a script if i want to do that.
- `journalctl -fu bluetooth` make it so that i can see the real time logs of the service (bluetooth). I played around with it as it kept on showing logs when i turned bluetooth on and off, and also when i connected and disconnected a bluetooth device.
- `journalctl -u gdm -b` showed logs since the last boot.
- Ok so i realized i dont have to create a script to get logs of a specific date. `journalctl -u <service> --since "2026-07-13 10:00:00" --until "2026-07-13 11:00:00"` works just fine for this purpose.

## systemd-analyze
This command is great. This would have been a lot helpful previously when i was trying to optimize my ubuntu's boot time.
- `systemd-analyze blame` lists all the systemd units that took part in the last boot. It also includes the time each one took to go from *starting* to *active*.
- Here i found a service named `NetworkManager-wait-online.service` that was taking 5.485 seconds. This service blocks until the network is considered online.
- I didnt have a service that absolutely required the network to be up during boot so i could safely mask this service and decrease my boot time.
- `systemd-analyze critical-chain` gives the critical chain during boot. This also confirmed that the above service was taking up unnecessary boot time.
- Just to be sure that nothing depended on that service i ran `systemctl list-dependencies network-online.target` to check for if anything has this service in it's dependencies. There wasn't any such unit.
- I then masked the service by `sudo systemctl mask NetworkManager-wait-online.service` and rebooted my laptop.
- After rebooting when i check my boot time using `systemd-analyze`, my boot time had dropped to 8.415 secs (from 10.538), making it 2.1 secs faster.


## scripts
I learned that i should put the command `set -euo pipfail` at the start of all my scripts as it makes the script's failure more recognizable and will save me hours of puzzling over silent failures.
I started putting this command at the start of my scripts and running them again to fix them and deal with any silent failues happening that i was not aware of.

- `shiftingParameters` script - Fixed the problem of unbound variable it was causing. I wasn't checking if `$1` existed or not before using it. Normally this is a silent error but now i was able to see it and fix it. 
No other scripts were showing any problems.

# boot-time-check script
Created a script that checks the userspace boot time and notifies me using a notification (using notify-send command) if the boot time is above a certain threshold.
For not i have harcoded the threshold's value but in the future i think i'm going to replace it with some sort of rolling average.
I made it so that the scripts runs after every boot automatically.

First i created a services file (`boot-time-check.services`) in `/etc/systemd/system`. 
```
[Unit]
Description=Check userspace boot time and notify if slow
After=graphical-session.target

[Service]
Type=oneshot
ExecStart=/home/tanish/.local/bin/boot-time-check

[Install]
WantedBy=default.target
```
- `After=graphical-session.target` means that it will run after the GNOME session is fully up.
- `Type=oneshot` means that it only runs once.
- `ExexStart` is the absolute path for the script
- `WantedBy=default.target` This is what makes it run at login

After the service file is created the following steps were taken:
- `systemctl --user daemon-reload` It just reloads 
- `systemctl --user enable boot-time-check.service` This is what created the link that makes the `WantedBy=` field take effect.

I put the script file in `~/.local/bin` file.
	



