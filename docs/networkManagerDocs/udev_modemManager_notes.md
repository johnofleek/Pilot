---
layout: default
title: Modem Manager - Debug
parent: Network Manager
nav_order: 2
has_children: false
has_toc: false
---


# Modem manager notes

The following are notes on an evaluation of using the Raspberry Pi + WASP2 + HL7802 module with modem manager. 

They are also a helpful example on how to debug issues with modem manager.

At the moment the investigation is stalled due to  
https://gitlab.freedesktop.org/mobile-broadband/ModemManager/-/issues/305

## Modem manager versions
Buster version as shipped with Raspian (as of 2020/12)
```
mmcli --version
mmcli 1.10.0
```

```
nmcli tool, version 1.14.6
```

As part of the investigation I installed a later version  
http://marksrpicluster.blogspot.com/2019/12/add-buster-backports-to-raspberry-pi.html  
this didn't fix the issue 


Update the os
```
sudo apt update
sudo apt full-upgrade --fix-missing

sudo apt autoremove
```


Then add in the backport
```
sudo nano /etc/apt/sources.list

Add the following line to the end

deb http://deb.debian.org/debian buster-backports main
```
then

```
curl -O http://http.us.debian.org/debian/pool/main/d/debian-archive-keyring/debian-archive-keyring_2019.1_all.deb

sudo dpkg -i debian-archive-keyring_2019.1_all.deb
```

Check version [installed] vs backport version
```
sudo apt update
```
```
sudo apt list modemmanager* -a
Listing... Done
modemmanager-dev/stable 1.10.0-1 armhf
modemmanager-doc/stable 1.10.0-1 all
modemmanager-qt-dev/stable 5.54.0-1 armhf
modemmanager/stable,now 1.10.0-1 armhf [installed,automatic]
```

```
sudo apt list network-manager* -a
Listing... Done
network-manager-config-connectivity-debian/stable 1.14.6-2+deb10u1 all
network-manager-dev/stable 1.14.6-2+deb10u1 armhf
network-manager-fortisslvpn-gnome/stable 1.2.8-2 armhf
network-manager-fortisslvpn/stable 1.2.8-2 armhf
network-manager-gnome/stable,now 1.8.20-1.1 armhf [installed]
network-manager-iodine-gnome/stable 1.2.0-3 armhf
network-manager-iodine/stable 1.2.0-3 armhf
network-manager-l2tp-gnome/stable 1.2.10-1 armhf
network-manager-l2tp/stable 1.2.10-1 armhf
network-manager-openconnect-gnome/stable 1.2.4-2 armhf
network-manager-openconnect/stable 1.2.4-2 armhf
network-manager-openvpn-gnome/stable 1.8.10-1 armhf
network-manager-openvpn/stable 1.8.10-1 armhf
network-manager-pptp-gnome/stable 1.2.8-2 armhf
network-manager-pptp/stable 1.2.8-2 armhf
network-manager-ssh-gnome/stable 1.2.10-1+deb10u1 armhf
network-manager-ssh/stable 1.2.10-1+deb10u1 armhf
network-manager-strongswan/stable 1.4.4-2 armhf
network-manager-vpnc-gnome/stable 1.2.6-2 armhf
network-manager-vpnc/stable 1.2.6-2 armhf
network-manager/stable,now 1.14.6-2+deb10u1 armhf [installed]
```


Install modem manager backport
```
sudo apt -t buster-backports install modemmanager
mmcli --version
mmcli 1.14.8
```
Install network-manager backport
```
sudo apt -t buster-backports install network-manager
sudo apt -t buster-backports install network-manager-gnome
```





## HL7802 physical connection

For testing purposes the HL7802 physical UART was connected to a USB to TTY serial port board. This resulted in the UART appearing to the RPi as 

```/dev/ttyUSB0```

In order for modem manager to "probe" ttyUSB0 we need to add a udev rule.

Add this text in file /etc/udev/rules.d/ttyACM1-HL7702.rules

```
ACTION=="add|change|move", KERNEL=="ttyUSB0", ENV{ID_MM_TTY_FLOW_CONTROL}="none", ENV{ID_MM_TTY_BAUDRATE}="115200", ENV{ID_MM_CANDIDATE}="1", ENV{ID_MM_DEVICE_PROCESS}="1",ENV{ID_MM_PORT_TYPE_AT_PPP}="1"
```

For documentation see 

see https://man7.org/linux/man-pages/man8/udevadm.8.html

Then to test 
```
sudo udevadm control --reload-rules

sudo udevadm test -a -p  $(udevadm info -q path -n /dev/ttyUSB0)
```

## Debug modem manager

Enable modem manager debug to syslog
```
sudo mmcli --set-logging=DEBUG
Successfully set logging level
```

Observe the debug output
```
tail -f /var/log/syslog
```

Example connection test
```
mmcli -m 0 --simple-connect="apn=internet,username=eesecure,password=secure"
```

## Stop modem manager
This is helps if you want to configure the modem via a tty interface and modem manager is also using the interface. 

```
sudo systemctl stop ModemManager
```



## Debugging modem manager example
Use nmcli to bring up the IP connection on a modem interface
```
sudo nmcli connection up ee ifname ttyUSB0
```

Capture the debug via "tail -f /var/log/syslog"


```
...
Running registration checks (CS: 'no', PS: 'no', EPS: 'yes')

Jan 11 16:38:32 raspberrypi ModemManager[385]: <debug> (ttyUSB0): --> 'AT+CEREG?<CR>'
Jan 11 16:38:32 raspberrypi ModemManager[385]: <debug> (ttyUSB0): <-- '<CR><LF>+CEREG: 2'
Jan 11 16:38:32 raspberrypi ModemManager[385]: <debug> (ttyUSB0): <-- ',0<CR><LF><CR><LF>OK<CR><LF>'
Jan 11 16:38:32 raspberrypi ModemManager[385]: <debug> Modem not yet registered in a 3GPP network... will recheck soon

Running registration checks (CS: 'no', PS: 'no', EPS: 'yes')
...
```

## HL7802 modem manager bug investigation - WIP 

### The HL7802 / modem manager registration incompatibility bug
Do it looks like when jammed in GSM the modem oks creg but not cereg

```
+CEREG: 2,0

+CREG: 0,1

```


From https://www.freedesktop.org/software/ModemManager/man/latest/mmcli.1.html

https://www.freedesktop.org/software/ModemManager/api/latest/ModemManager-Common-udev-tags.html#ID-MM-PORT-TYPE-AT-PPP:CAPS


```
mmcli -m 0 --output-keyvalue
```

```...```
```
modem.generic.supported-modes.length            : 1
modem.generic.supported-modes.value[1]          : allowed: 2g, 4g; preferred: none
modem.generic.current-modes                     : allowed: 2g, 4g; preferred: none
...
```


Ok but we jammed the modem into 2G

Maybe try
```
sudo mmcli -m 0 --set-allowed-modes "2G|4G"
error: couldn't set current
modes: 'GDBus.Error:org.freedesktop.ModemManager1.Error.Core.Unsupported: Setting allowed modes not supported'
```



