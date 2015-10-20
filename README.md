# Introduction
This documents down the settings for racoon IPSec VPN server setup in AWS (using Ubuntu 14.04) with OS X (El Capitan) and Windows (ShrewSoft VPN Client) as road warriors.
The VPN use a Pre Shared Key (PSK), so no certificate generation is needed.
The routing is simple route everything through the VPN server.  This is useful if you need a US IP address.

# Server Setup
*All commands should be using root `sudo -i` or add `sudo` to all the commands.  Replace anything like \<to_be_replace\> with your own*

## AWS EC2 Instance
1. Make sure that you boot up an EC2 instance in the region that you want the IP address to be.  Assign an Elastic IP to the instance you booted up.
1. To allow IPSec connection, your Security Group should have:
  1. Custom UDP Rule - UDP - 4500 - 0.0.0.0/0 (or your restricted network)
  1. Custom UDP Rule - UDP - 500 - 0.0.0.0/0 (or your restricted network)

## Installation of Packages
1. Update the system: `apt-get update && apt-get dist-upgrade`
1. Install `ipsec-tools` and `racoon`: `apt-get install ipsec-tools racoon`
  1. Choose "Direct" configuration for racoon when asked

## Configure racoon
1. Edit `/etc/racoon/racoon.conf` file, you can use the example in the repository.  You may want to change lines 28 - 32.
1. Create a the PSK `echo <GroupName> `dd if=/dev/urandom bs=1 count=18 2>/dev/null | base64` > /etc/racoon/psk.txt`
  1. Replace \<GroupName\> with the GroupName you want
  1. Copy down the \<GroupName\> and generated Key in `/etc/racoon/psk.txt`
1. Create the file `/etc/racoon/motd` and put in the following message:
```
There are 10 types of people in this world, those who understand binary and those who dont.
```
1. Restart racoon `service racoon restart`

## Configure NAT Routing
1. Enable IP Forward by editing the file `/etc/sysctl.conf` and remove the comment for the line `net.ipv4.ip_forward=1` (or simple add the line to the end)
1. Run the command `sysctl -p`

## Configure iptables
1. `iptables -t nat -A POSTROUTING --out-interface eth0 -j MASQUERADE`
1. `iptables -A FORWARD --in-interface eth0 -j ACCEPT`

## Add VPN User
1. You need to add VPN User to the system.  It can be done with `useradd -M -s /usr/sbin/nologin <VPNUserID>`, replace <VPNUserID> with the User ID you wish.
1. Give the VPN User a password by `passwd <VPNUserID>`

# OS X VPN Client Setup
1. Open Network Preferences
1. Click on the '+' sign at the bottom left to create a VPN connection:
  1. Interface: **VPN**
  1. VPN Type: **Cisco IPSec**
  1. Service Name: <ServiceName>
  1. Click on the **"Create"** button
1. Once the VPN Interface is created, click on the **"Advance..."** button:
  1. Put in the Shared Secret and Group Name as the information in `/etc/racoon/psk.txt` on the VPN server
1. You should be able to connect to the VPN server using the User ID and password set in **Add VPN User** step

# Windows (ShrewSoft) VPN Client Setup
1. Download [ShrewSoft VPN Client](https://www.shrew.net/download/vpn)
1. Install the VPN client.  If you encounter any error during installation, please refer to [ShrewSoft's FAQ](https://www.shrew.net/support/Frequently_Asked_Questions)
1. Configure the VPN connection:
  1. General tab:
    * Host Name or IP Address: <Your ElasticIP>
  1. Client tab:
    * NAT Traversal: **force-frc**
    * **Uncheck** Enable Dead Peer Detection
  1. Authentication tab:
    1. Local Identity sub tab:
      * Authentication Method: **Mutual PSK + XAuth**
      * Identification Type: **Key Identifier**
      * Key ID String: **<GroupName>**
    1. Remote Identity sub tab:
      * Identification Type: **Any**
    1. Credential sub tab:
      * Pre Shared Key: **<PSK in /etc/racoon/psk.txt>**
  1. Phase 1 tab:
    * Exchange Type: **Aggressive**
    * DH Exchange: **group 2**
    * Cipher Algorithm: **aes**
    * Cipher Key Length: **256**
    * Hash Algorithm: **sha1**
  1. Phase 2 tab:
    * Transform Algorithm: **esp-aes**
    * Transform Key Length: **256**
    * HMAC Algorithm: **sha1**
    * PFS Exhcnage: **disabled**
    * Compress Algorithm: **deflate**
  1. Policy tab:
    * Policy Generation Level: **unique**
1. Connect to the VPN server using the <VPNUserID> and password set in **Add VPN User** step
