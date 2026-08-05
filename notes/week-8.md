# Week-8
*03/07/26-09/07/26*

## netfilter
Netfilter is a subsystem that provides a framework for implementing advanced network functionalities such as packet filtering, network address translation (NAT), and connection tracking. It achieves this by leveraging hooks in the kernel’s network code, which are the locations where kernel code can register functions to be invoked for specific network events.

### netfilter hooks
These are functions that are registered with the kernel to be called at specific points in the network stack. These hooks are like checkpoints at different levels of the network stack.

There are 5 pre- defines network filter hooks:
1. `NF_IP_PRE_ROUTING`: The callbacks registered to this hooks will be triggered by any incoming traffic immediately after entering the network stack. This hook is processed before any routing decisions have been made regarding where to send the packet. 
2. `NF_IP_LOCAL_IN`: The callbacks registered to this hook are triggered after an incoming packet has been routed and the packet is destined for the local system.
3. `NF_IP_FORWARD`: Like above but when packet is to be forwarded to another host.
4. `NF_IP_LOCAL_OUT`: Triggered by any locally created outbound traffic as soon as it hits the network stack.
5. `NF_IP_POST_ROUTING`: Triggered by any outgoing or forwarded traffic after routing has taken place and just before being put on the wire.

## Phone server
I was thinking about putting a basic python server in a VPS (virtual private server) on a cloud service provider like AWS or Oracle, but then i had an idea of turning my old android phone into a server and using that.

The first step involved installing Termux on my phone. Termux is a terminal emulator and Linux environment app for android.
After installing Termux, i opened it which opened a terminal. Firstly, i installed openssh using `pkg install openssh`. This is for connecting to the phone using ssh.
Next i set up a password using the command `passwd`.
To start the sshd i just had to run `sshd`. Termux's sshd runs on the port `8022`, useful to keep in mind while connecting. It runs automatically in the background freeing up the terminal.
To get the ip address of the phone i ran `ip addr show wlan0` which prompted that i have not installed the program `ip`. I installed it by `pkg install iproute2`. 
The ip addr command shows the ip address in the format `inet ip-address`.
From my laptop, i connected it by `ssh -p 8022 u0_a651@ip-address`. `u0_a651` is my termex username.


