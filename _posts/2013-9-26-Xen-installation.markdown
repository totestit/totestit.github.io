---
title: Xen installation on Fedora
layout: post
tags:
    linux
    xen
    virtualization
---

About the following setups were my work logs when I was an intern in Ningbo Shufang Information Technology Co.,Ltd.

**1. Install Fedora 17**  
Here I used 'LiveUSB Createtor' to make a live Fedora.  
About the procedure of Fedora installaztion, I will omit it;  
And the disk partition plan is listed below.  
<table border="1" cellspacing="0" cellpading="0" width="300">
<tr align="center">
<td colspan="2">BIOS Boot</td><td>1MB</td>
</tr>
<tr align="center">
<td>/boot</td><td>ext4</td><td>5120MB</td>
</tr>
<tr align="center">
<td>/</td><td>ext4</td><td>51200MB</td>
</tr>
<tr align="center">
<td colspan="2">swap</td><td>16384MB</td>
</tr >
<tr align="center">
<td colspan="2">physical volume(LVM)</td><td>reamining space</td>
</tr>
</table>
Next, to create a LVM group, to name it vg\_xen.
<table border="1" cellspacing="0" cellpading="0" width="300">
<tr align="center">
<td>lhome</td> <td>/lhome</td><td>ext4</td><td>512000MB</td>
</tr>
</table>
The `/lhome` directory is use to install & store virtual machine.  


**2. Configure static IP**  
Firstly, it is necessary to run the command `ifconfig` to find which network card is being used.  
And then run the command below.  
`vi /etc/sysconfig/network-scripts/ifcfg-XXX`  
'XXX' is the name of the network card which is being used now.  
Modify & add the content to `ifcfg-XXX` below.  
{% highlight cpp%}
BOOTPROTO="static"  
IPADDR="10.16.0.246"  
NETMASK="255.255.0.0"  
GATEWAY="10.16.0.1"  
DNS1="10.16.0.1" 
{% endhighlight %}
Then, restart the network service to take static IP effective. Attention, under Fedora 17, the command is  
`systemctl restart NetworkManager.service`  


**3. Install Xen dom0**  
Firstly, to clear the iptables.  
{% highlight cpp%}
iptables -F   
iptables -t nat -F   
{% endhighlight %}  
For safety's sake, copy a iptables file befor save rules.  
`cp /etc/sysconfig/iptable   /etc/sysconfig/iptables.bak`  
`iptables-save > /etc/sysconfig/iptables`  
Restart iptables service.  
`systemctl restart iptables.service`  

Next step is to disable the SELinux.  
`vi /etc/selinux/config`  
Change 'SELinux' to disabled  
`SELinux=disabled`  
Restart the machine.  

Now, we can install xen. Run the command below.  
`yum -y install xen`  
If the download speed is less than 10KB/S, please read the Q&A.1.

Next is to configure the grub2. Run command.  
`grub2-mkconfig -o /boot/grub2/grub.cfg`  

And then close X windows. Use tty3 as default.  
`rm -f /etc/systemd/system/default.target`  
`ln -s /lib/systemd/system/multi-user.target /etc/systemd/system/default.target`  
Ok, next starting the system, it will enter the tty3 mode automatically.   
For the Xen isn't the default boot item. We must set automactically select the Xen to start.  
So, first list the exsiting items.  
`grep ^menentry /boot/grub2/grub.cfg | cut -d "'" -f`  
Then set grub default item.  
`grub2-set-default 'Linux Fedora, with Xen hyperisor'`  
Attention, the 'Linux Fedora, with Xen hypervisor' may not be yours. You must let the name equal to the name of xen.  
You can run command below to find is the name 'Linux Fedora, with Xen hypervisor' equal to the name of xen. It must be same. Or Xen will not be selected automatically.  
`grub2-editenv list`  
Lastly, generate the configuration file.  
`grub2-mkconfig -o /boot/grub2/grub.cfg`  
If anything is smooth, when you restart the machine, the item 'Linux Fedora, with Xen hypervisor' will be selected automatically.   

Next step is to install SSH. After installing the SSH server, we can use 'ssh' command to link the server from our computers.  
Run the install command.  
`yum -y install openssh`  
And then enable the ssh service.  
`systemctl enable sshd.service`   
If you want to check it ssh service. Run it below.  
`/sbin/service sshd status`  
Then, restart the machine.  


**4. Install Xen domU**  
First, be sure the network is connected.  
Create directory.  
{% highlight cpp%}
cd /lhome
mkdir xen
cd xen
mkdir f12
{% endhighlight %}  

Configure the network bridge.  
Add the content below to the configuration file 'if-XXX'.  
{% highlight cpp%}
DEVICE="em1"
BRIDGE=br0
NM_CONTROLLED=no
{% endhighlight%}  
Create a virtual bridge configuration file 'ifcfg-br0'.  
Add these content below to the 'ifcfg-br0'.  
{% highlight cpp%}
DEVICE=br0 
TYPE=Bridge 
BOOTPROTO=static 
IPADDR=10.16.0.246 
NETMASK=255.255.0.0 
GATEWAY=10.16.0.1 
ONBOOT=yes 
DELAY=0 
NM_CONTROLLED=no
{% endhighlight%}  
Then, to enable the br0.  
`ifup br0`   

Next, It' time to start to install the domU. Here, I first show the installaztion of the Fedora 12 as the virtual machine.  
First, to download two files which names vmlinuz and initrd.img.   
You can download them from the **[CUHK mirror](http://ftp.cuhk.edu.hk/pub/linux/fedora-archive/fedora/linux/releases/12/Fedora/x86_64/os/images/pxeboot/)**.  
Put the two files under the directory `/lhome/xen/f12`. If the download speed is less then 10KB/S, please refer to the Q&A.1.  
Then create a VBD.  
`dd if=/dev/zero of=/lhome/xen/f12/vmdisk0 bs=1k seek=20480K count=1`  
Create a configuration file 'f12.install'. Add these content below to the file.  
{% highlight cpp %}
name="f12install"
vcpus=2
memory=2048
disk = ['file:/lhome/xen/f12/vmdisk0,xvda,w' ]
vif = [ 'bridge=br0' ]
kernel = "/lhome/xen/f12/vmlinuz"
ramdisk = "/lhome/xen/f12/initrd.img"
on_reboot = 'restart'
on_crash = 'restart'
{% endhighlight %}

Then, run the `create` command to create a new virtual machine.  
`xl createa -c f12.install`  
It's a new procedure of operating system installation. 
You must select the 'URL' to install the virtual machine.  
Then, select the DHCP.  
Paste the URL **[CUHK mirror](http://ftp.cuhk.edu.hk/pub/linux/fedora-archive/fedora/linux/releases/12/Fedora/x86_64/os/images/)**, **AND manually enter 'install.img' behind the URL**. I don't why. Maybe totally copy will have some characters.  
Wait for the finishment of retrieving. And the select the 'Use Text Mode'.  
Then it's same to install a new operating system.  
You can back to the dom0 by entering the command `ctrl+]`.  

Run a virtual machine need a configuration file too. But, the Configuration is easy. It's listed below.  
{% highlight cpp%}
name="f12install"
vcpus=2
memory=2048
disk = ['file:/lhome/xen/f12/vmdisk0,xvda,w' ]
vif = [ 'bridge=br0' ]
bootloader="/usr/bin/pygrub"
on_reboot = 'restart'
on_crash = 'restart'
{% endhighlight%}  
Save it and name it 'f12.installed'.  
Run the command below can start the VM.  
`xl create -c f12.installed`  

Then set the domU auto start as the dom0 start.  
{% highlight cpp %}
cd /etc/xen/auto
ln -s /lhome/xen/f12/f12.installed
{% endhighlight%}

<br/>
<br/>
<br/>
**Q&A.1**  
If the download speed less than 10KB/S, you can use this method.  
Edit all .repo file. 
Add the '&country=us' at the end of the 'mirrolist'
