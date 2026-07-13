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

 	  



