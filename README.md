# kvm-scripts
Useful network scripts for QEMU/KVM
===================================

QEMU/KVM offers powerful networking features, including the ability to create many kinds of networks for your VMs. These can be either an isolated network, a NAT network, or a bridge. There are built-in services that can be configured to serve DNS and DHCP (including PXE support), so you do not need to bother maintaining these services separately, if all you need is basic functionality. Everything is easily editable via libvirt-provided mechanisms, like virt-viewer or virsh.

However, a common problem with QEMU/KVM is how it deals with updates. Many actions, like adding DNS or DHCP entries for a network, will not yield instant results, despite it *looking like it did*. For instance, if you change, for example, an entry for a fixed DHCP host, the change will *not* be reflected until you do the following:
1) do a net-destroy on the network
2) do a net-start on the network
3) reattach the network interface to each VM that was using your network previously (yes, all interfaces are detached... but not reattached until you restart the VM)

Also, there's the port forwarding issue. How do you expose a port to the outside world that goes right into that specific VM? You need a set of iptables rules for this. These rules must be added dynamically, via a hook script. 

The official libvirtd documentation mentions these problems, and offers two example hook scripts (https://wiki.libvirt.org/page/Networking#Applying_modifications_to_the_network). However, their approach is too generic, and uses too many hardcoded values. For example, it tries to reattach interfaces on ALL VMs, even the ones that are not using the network you're modifying. Also, they do not integrate well with each other.

My solution is to rewrite them with the following objectives:

1) easy port forwarding configuration
2) make the process of refreshing the network configuration easy and faster
3) only change the affected VMs/interfaces

Adding port forwards, the easy way
===================================

1. copy and rename the "qemu-hook-script" file to "/etc/libvirt/hooks/qemu". Please note this is a *file* called "qemu", not a directory.

2. modify the file to add your forwarding rules. Adding a port forward is as simple as adding a line like this:

       addForward <VM name> <source port> <destination address> <destination port>
       addForward <VM name> <source NIC> <source address> <source port> <destination NIC> <destination address> <destination port> <protocol>

For example, to redirect incoming connections on eth0 (192.168.56.1) port 20022/tcp to an internal address of 192.168.101.200 on port 22/tcp, whenever the VM named "caasp-admin" is started, you'd write:

    addForward caasp-admin eth0 192.168.56.10 20022 virbr0 192.168.101.200 22 tcp

To redirect all TCP ports for the same host instead:

    addForward caasp-admin eth0 192.168.56.10 1:65535 virbr0 192.168.101.200 1-65535 tcp

3. Set it as executable, or else QEMU/KVM won't call the hook script:

       chmod +x /etc/libvirt/hooks/qemu

The next time you start the VMs listed, the port forwards will be defined properly.

Alternative script that accepts multiple network devices (via nftables): https://github.com/se1by/libvirt-hook-nftables


Refreshing network configurations, the easy way
================================================
1. copy the "kvm-network-restart" script to your path (I recommend /usr/sbin).

2. make it an executable:

       chmod +x /usr/sbin/kvm-network-restart
      
and rename the "qemu-hook-script" file to "/etc/libvirt/hooks/qemu". Please note this is a *file* called "qemu", not a directory.

Whenever you change an attribute on a network, say via "virsh net-edit <network>", you can now make the changes available by running "kvm-network-restart <network>".
  
For example, let's say you wanted to reserve an IP address for a specific VM, and add it to the DNS. Just run "virsh net-edit <network name>" and add the following:

    <dns>
      <host ip='192.168.101.200'>
        <hostname>admin.local</hostname>
      </host>
    </dns>
    <ip address='192.168.101.1' netmask='255.255.255.0'>
      <dhcp>
        <range start='192.168.101.128' end='192.168.101.254'/>
        <host mac='52:54:00:26:37:81' name='admin' ip='192.168.101.200'/>
       </dhcp>
    </ip>
  
Then, run "kvm-network-restart <network name>" and you're set.


Doing everything at once: the Python way
========================================

I'm also providing a hook script written in Python, that can be used to do everything in one place. This script receives (and parses) the machine information from stdin, and automatically starts/kills websocket servers for SPICE displays. It also knows how to deal with both iptables and firewalld to add/remove the proper ports.

Just copy the provided qemu-hook-script-python to "qemu" under the /etc/libvirt/hooks directory. Don't forget to chmod +x it. 

Syslog status messages are generated every time you start/stop a VM, just check out journalctl.


Talking to QEMU Guest Agent
===========================

I included a small tool to talk to the installed QEMU Guest Agent (only on the KVM host, as it talks directly to the socket provided by the special guest-agent device inside the VM!) and do some common functions.
The official API is not very easy to work with from the command-line, so this tool makes things a little easier.

```
# qemuguest [-n <VM name>] [-h|--help] [-e|--exec <command to execute>] [-i|--info] [--changepassword|-p <username:password>] [-f|--filesystems]
```

Executing a command:

```
# ./qemuguest -n MYVM-1 -e "uname -a"
Command execution returned PID=15643
Command execution returned RC=0
exec rc = 0, output=[Linux myvm1.localdomain 4.12.14-197.37-default #1 SMP Fri Mar 20 14:56:51 UTC 2020 (e98f980) x86_64 x86_64 x86_64 GNU/Linux
]
```

It only waits for 5 seconds by default, so if your command takes too long, it may not retrieve the results.


Getting information:

```
# qemuguest -n MYVM-1 -i

---- System information ----

version-id: 15.1
kernel-version: #1 SMP Fri Mar 20 14:56:51 UTC 2020 (e98f980)
name: SLED
machine: x86_64
version: 15-SP1
pretty-name: SUSE Linux Enterprise Desktop 15 SP1
kernel-release: 4.12.14-197.37-default
id: sled

---- Network Interfaces ----

lo (mac: 00:00:00:00:00:00)
        127.0.0.1/8 (ipv4)
eth0 (mac: 52:54:00:6d:46:d4)
        10.24.101.169/24 (ipv4)
        fe80::76f5:aa91:4640:5a8b/64 (ipv6)
docker0 (mac: 02:42:8c:e2:4d:2b)
        10.1.1.1/24 (ipv4)

```


Listing file systems:

```
# ./qemuguest -n MYVM-1 -f
---- Filesystem information ----

Device: sda7 (QEMU_HARDDISK_QM00003)
Mount point: /home
File system: ext4
Used bytes: 15 MiB (Total: 3644 MiB)

Device: sda2 (QEMU_HARDDISK_QM00003)
Mount point: /boot/efi
File system: vfat
Used bytes: 0 MiB (Total: 99 MiB)


Device: sda4 (QEMU_HARDDISK_QM00003)
Mount point: /
File system: ext4
Used bytes: 7184 MiB (Total: 8530 MiB)
```

Changing a user password:

```
# qemuguest -n MYVM-1 -p root:password
set password = [True]
```

Downloading a file from the guest:

```
# qemuguest -n MYVM-1 -d /etc/issue:test.remote
Received file /etc/issue (104 bytes)
```

Files are limited to 65KiB.


Uploading a file to the guest:

```
# qemuguest -n MYVM-1 -u test.remote:/root/blah
Sending file test.remote to /root/blah
Total bytes written: 104
```

Files are limited to 65KiB. All files are created with the default umask and root:root.
If you need a different owner/group and/or permission, run the appropriate command via the "--exec" parameter.



Have fun!
