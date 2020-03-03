# KPN-Routed-IPTV-with-OPNsense
Explenation on how to get Routed IPTV from KPN working with OPNsense

So, You have KPN FTTH (or another KPN based) with IPTV but don't want to use the included modum/router?
Come and join the club.

Previously I had this fixed with the Unify Seciruty Gateway Pro from Ubiquiti.
But, I wanted something more that that router. Namely a better IPS and OpenVPN client options.

So, after some searching and testing I landed at OPNsense.

However, I couldn't find a all-in-one guide on how to configure IPTV.

First some basics:
I have a few networks, but the ones needed for this are:
WAN
IPTV_WAN
LAN
IPTV_LAN (optional)

I have my KPN TV Box in a seperate VLAN. If you don't want this, just add the rules/options to your normal LAN instead of the IPTV_LAN

My WAN side is on vmx1
My LAN side is on vmx0

#Interfaces#
On your WAN side you need 2 VLAN's.
VLAN4 with DHCP for the TV
VLAN6 with PPPoE for your Internet

Go to Interfaces -> Other Types -> VLAN and click on the Add button.
Add VLAN4 to your WAN and click save (vmx1 in my case)
Add VLAN6 to your WAN and click save (vmx1 in my case)
Bonus: Add VLAN40 to you LAN and click save (vmx0 in my case)

Go to Interfaces -> Assignments
Assign vmx1_vlan6 to the WAN
Assign a new interface, call it WAN_IPTV and assign it to vmx1_vlan4
Assign vmx0 to LAN
Bonus: Assign a new interface, call it IPTV and assign it to vmx0_vlan40

Go to Interfaces -> WAN
Enable the device, name it WAN (if it isn't already)
Block Private networks
Block Bogon networks
IPv4 type = PPPoE
Username = mac-address@internet
Pass = kpn
click Save

Go to Interfaces -> WAN_IPTV
Enable the device, name it WAN_IPTV (if it isn't already)
IPv4 type = DHCP
DHCP client mode -> Advanced
at Lease Requirements:
Send Options = dhcp-class-identifier "IPTV_RG"
Request Options = subnet-mask, routers, classless-routes
click Save

Go to Interfaces -> LAN
Configure it the way you want. And add the info form interface IPTV if you don't want a seperate network for you IPTV.

Go to Interfaces -> IPTV
Enable the interface, name it IPTV (if it isn't already)
IPv4 type is Static
Give it a IPv4 address (mine is 192.168.40.1/24)
Click Save

#Firewall#
Go to Firewall -> Rules -> IPTV
Add a rule
Action = Pass
Interface = IPTV
Direction = in
TCP version = IPv4
Protocol = any
Source = IPTV net
Destination = any
Advanced features
allow options = true

click Save

Go to Firewall -> Rules -> WAN_IPTV
Add the following rules, they are the same as the IPTV rules accept for the following parts:
Rule A: Protocol = IPv4 IGMP
Rule B: Protocol = IPv4 UDP, Source = 213.75.0.0/16, destination = 224.0.0.0/4
Rule C: Protocol = IPv4 UDP, Source = 217.166.0.0/16, destination = 224.0.0.0/4
Rule D: Protocol = IPv4 UDP, Source = 213.75.0.0/16, destination = WAN_IPTV address
Rule E: Protocol = IPv4 UDP, Source = 217.166.0.0/16, destination = WAN_IPTV address
Rule F: Protocol = IPv4 any, Source = 10.0.0.0/8, Destination = 224.0.0.0/4

#NAT#
Got to Firewall -> NAT -> Outbound
set the mode at Hybrid
Add the following 3 outbound rules:
A: Interface WAN_IPTV, Source IPTV net, source port+destination+destination port = any, NAT address = WAN_IPTV address
B: Interface WAN_IPTV, Source IPTV net, source port = any, destination = 217.166.0.0/16, destination port = any, NAT address = WAN_IPTV address
C: Interface WAN_IPTV, Source IPTV net, source port = any, destination = 213.75.0.0/16, destination port = any, NAT address = WAN_IPTV address

#IGMP Proxy#
Make sure you have IGMP_Proxy installed.
if not, go to System -> Firmware -> Plugins and click on the + behind os-igmp-proxy

Got to Services -> IGMP Proxy
Add the following 2 rules:
A: Interface = WAN_IPTV, Type = Upstream Interface, Networks = 213.75.0.0/16, 10.0.0.0/1, 217.166.0.0/1
B: Interface = IPTV (or your LAN), Type = Downstream Interface, Network = (you network, in my case 192.168.40.0/24)

This should be all.
