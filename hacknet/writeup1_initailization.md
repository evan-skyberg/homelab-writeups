# The Hacknet part 1: Initialization
## Preface: What is this so-called hacknet?
The hacknet is an idea I had in high school. It's a simple setup that many people before me have done, but I still thought it would be cool regardless. It's an isolated network for practicing both red and blue teaming, all virtualized. The virtual machines would have software installed onto them through the Internet, then would be quickly moved off of the Internet into the isolated network (Which has no Internet connection to prevent the machines from being hacked for real) before being configured. These configurations would serve as a way to make a machine as strong or as weak as one would desire before letting the hackers run wild on it. In this particular writeup, we'll go over the software choices, processes, and eventual creation of this hacknet.
## 1: Software choices
### 1.1: Virtualization
For the virtualization software, I needed something that I could keep running on a separate machine for a long time. The goal is to have downtime for both red and blue teams so that, if the blue team isn't paying attention, the red team can weasel their way into the system or, in contrast, if the red team is lax about leaving logs, then the blue team can detect a compromised machine while the red team isn't actively in the system. Back to the topic of software choices, though, I also needed something capable of running virtual networks and would, ideally, have relatively easy setup and be free and open-source. Because of these constraints, I chose the Proxmox Virtual Environment. I installed it on a spare mini PC (One woefully underpowered for this task, but usable regardless) and started work.
### 1.2: Gateway
For a hacknet to be accessible to a select few people who wish to assist with the project, and/or wish to hack it, a secure gateway is needed. Since the gateway is required to be virtualized as well because I don't have enough hardware to create dedicated machines for everything, I figured that a VPN would do the trick. The debate then began in my head: Tailscale, OpenVPN, or WireGuard? Well, right off the bat, Tailscale was removed because, despite being based on WireGuard, it's also got some proprietary behind-the-scenes shenanigans going on with the server orchestration and that defeats the point of attempting to go full-FOSS. After that revelation, and after reading a few guides on setting up WireGuard and OpenVPN, I was at first going to go with OpenVPN since it seemed to be the tried-and-true option. That, and it was able to be used out-of-the-box by Kali Linux, which I installed on a Raspberry Pi 4 for a friend to learn to hack with. However, the deeper down the rabbit hole I dove, the more I found that the antiquated and bloated OpenVPN codebase was getting slow in its old age and that WireGuard was (allegedly) the easier system to set up. Therefore, WireGuard was chosen.
## 2: Setup
### 2.1: Proxmox Virtual Environment
Setup with the Proxmox VE was as simple as installing any other Linux distro, bar those like Arch, Gentoo, or LFS. I won't go too much into depth. The only troublesome part was finding a static IP for it. In order to do so, I had to go into my router's configuration panel and find the DHCP range. Anything below the DHCP range is fair game for static IPs, at least on my router.
### 2.2: VM Bridge
In order to set up a network for the hacknet, I had to configure something that would allow for connections in and out through the gateway, and only through the gateway. Thus, a bridge was chosen. The VPN would act as a router for the hacknet as well as a firewall to prevent inner traffic from escaping beyond what the VPN would allow. Setting up the bridge itself wasn't difficult once I knew what I was doing, though I'll be the first to admit that I was flailing about until I read some excellent guides by some excellent people. Unfortunately, I didn't have the foresight to write these guides' links nor authors' names down before I finished the setup. All that was required was that a bridge needed to be created without a CIDR or a gateway, which was accomplished by leaving those respective fields blank when I made the bridge. For future reference, the bridge was named `vmbr1` for no other reason than that's what Proxmox named it.
### 2.3: VPN
**"At first, it was going well, then setback after setback, loss after loss, made what was going to be a quick and decisive win into 5 years of hell." - Captain Cutter of the UNSC *Spirit of Fire* (*Halo Wars*)**

At first, it was simple. I spun up the TurnKey Linux WireGuard appliance provided by the Proxmox web interface, added a second NIC connected to the bridge, copied the keys over to the Pi, copied the Pi's keys over to the VPN, wrote some configs, and set up some firewall rules, which were as follows:
```
iptables -A FORWARD -i wg0 -o eth1 -d [HACKNET SUBNET IP/MASK] -j ACCEPT
```
Append the following rule to the FORWARD chain: Input interface is wg0, output interface is eth1, and destination is [HACKNET SUBNET IP/MASK]. If the address is within the subnet, accept the packet.
```
iptables -A FORWARD -i eth1 -o wg0 -m state --state RELATED,ESTABLISHED -j ACCEPT
```
Append the following rule to the FORWARD chain: Input interface is eth1, output is wg0, and match the state extension. If the state is RELATED or ESTABLISHED, accept the packet.
```
iptables -A FORWARD -i wg0 -j DROP
```
Append the following rule to the FORWARD chain: Input interface is wg0. Drop the traffic. (iptables works like a match case statement where it returns on the first matched case)

Let it be known I also messed up typing the commands a couple times, but I fixed them. iptables commands also don't persist through reboots, so I had these commands placed in the `PostUp = ` portions of the WireGuard configs and their respective takedown commands in the `PostDown = ` sections. However, there was a different problem. I pinged the WireGuard exit, and no response. I pinged the entrance from the same network, and that one worked. Now, try as I might, I couldn't get a response from the demo machine inside the hacknet. Attempt after attempt, all to no avail. After a few hours of angry tinkering (Of which I took no notes, unfortunately) and many different attempts with different network configurations such as the Pi on my mobile hotspot, I tried to install `tcpdump` on the WireGuard machine. But, lo and behold, I found that there was no Internet connection, which was... odd. After a little more digging, I discovered the WireGuard server had no default route. I set the default route by editing my `/etc/network/interfaces` and adding `gateway [ROUTER IP]` to the eth0 entry, rebooted, and it worked! I was able to ping the demo machine inside the hacknet. The reason this broke is that, because I was pinging from outside my home LAN, the network path had to go through my router. Whenever my pings would go in, they'd find the demo machine inside the hacknet just fine. However, because the VPN had no default route, it had no idea where the router lived, so it couldn't send anything back.

## NETWORK DIAGRAM
```
+------+     +-----------------+   INTERNET   +-----------+     +-----+   vmbr1   +-----------------+
| RPi4 | <-> | Friend's Router | <----------> | My router | <-> | VPN | <---+---> | Vuln. Machine 1 |
+------+     +-----------------+              +-----------+     +-----+     |     +-----------------+
                                                                            |     +-----------------+
                                                                            +---> | Vuln. Machine 2 |
                                                                                  +-----------------+
```

## What's next?
The next goal is to teach my friend the basics of penetration testing. He's already set up and ready to go with his mini Kali box, and now the time has come for him to learn. In other words, the next item on the docket is creating intentionally vulnerable machines, ideally without getting hacked for real.