# Setting Up an Astrophotography Linux System

# Overview

This document will descibe how anyone could setup a new astrophotography system using open source,
Linux and commercial drivers. The desktop distribution that will be used will be Ubuntu Desktop v22.04.
There can be some ramifications for early adoption of newer flavors of unix/Ubuntu, and as such, 
jumping to newer versions is not recommended without testing.

This document will not be concerned with creating a "hot spot" device that can server as a complete
network for remote shoots. This document is geared towards those with an existing telecom hot spot or
are shooting in their backyards and have access to WiFi.

This document will only cover x86-64bit base IBM Clone pcs. While some of this could 
cover setup on small form factors like the Raspberry Pi or the Pine 64 boards, they are not the focus.

# The Systems

The systems being tested here are going to be two mini-pc's that have a tight VESA mount foot print
that could be easily attached to a telescop mount control the mount and devices remotely.

The setup of the devices and mount are as follows:

- The mount has the following:
  - Power strip to power all of the things on the mount and telescope assembly
  - Linux PC running INDI and possibly KStars and Ekos
  - Power supplies for the mount and pc
  - Communication cable from PC to mount
  - Communication cable from PC to telescope assembly
  - 12v power supply and cable going to telescope assembly
- The telescope assembly has the following
  - USB 3 hub that is externally powered
  - USB devices (cameras, focussers, filter wheels, guiders)
  - Short cables to connect all devices to hub
  - Power cable splitters to supply all devices

In this setup, there are two cables running from the telescope assembly to the
mount. This makes cable management much easier around the moving head of the mount.

Note: I am in the middle of adding the USB 3 hubs, cables and such to each of my 
telescopes to make setup really easy. Grab a scope assembly, put on mount, start 
setting up and shooting.

# Initial Setup

The first thing to do with any Linux setup is to acquire a system and figure out 
what hardware it has. The one thing that can really hold up installing and running
linux on a cheap PC is the wireless boards built into them. PCs with Broadcom, 
Realtek and MediaTek wifi's can be really hard to setup. Figure out what wifi is used
and then see if Ubuntu supports it. 

Note: Google is your friend here. I often google "Install drivers Ubunut (wifi board 
type)". If you see a lot of people writing articles about how hard it is to set that 
board up, don't use that board/PC.

Note: If you really like a particular PC form factor, but it has bad WiFi, then a 
good supported USB 3.0 based WiFi adapter can work well. MediaTek has some good ones
and they can be had for cheap from China, but they also have their downfalls. (I may 
write notes on this later)

Prior to starting the installation, Ubuntu will need to be downloaded from ISO to be 
installed. Using something like Rufus, transfer the ISO to the USB drive and prepare 
for installation.

After the USB stick is ready to boot Ubuntu installation, the next step will be to
start the PC and access the Boot Priority or Boot Menu from the BIOS. This is 
different on every Bios/PC manufacturer, the manual for the PC should have this, if 
not, Google.

Once the USB stick is running, install Ubuntu, consult Ubuntu's installation manual 
for this step, they change frequently and the docs are good on their site.

For now, this install just needs a minimal install with 3rd party drivers. If the 
install asks for a password for the device drivers, it can just be left blank. 

Note: I understand that leaving the password blank for device driver signing is not
a good practice, but these are single use machines at this point with wonky drivers
from niche suppliers. They aren't daily drivers you will be connecting to public 
wifi's or downloading much software to.

Create the first user and password to log into the system during installation.

# Post Install

After installation, update/upgrade the system and install SSH. 

Once logged, in, wait a while, the "automatic updater" will be running and it needs
to finish. Two apps cannot be updating the system at the same time. The updater can
be used to update the installation, but it can also be cancelled and the following 
commmands could be ran:

```bash
sudo apt-get update
sudo apt-get upgrade
```

This will update the system and afterwords, a reboot should be done:

```bash
sudo shutdown -r now
```

# Log In Remotely

Once the system is back up, SSH can be installed to allow remote command line to the
system. Open a terminal window, and do the following:

```bash
# Run the command
sudo apt-get install ssh

# Get the address of your system
ip addr
```

Now that SSH is installed, it can be remotely logged into. To do this open a command line window
from another computer and ssh to it:

```bash
# ssh <youruserid>@<youripaddress>
ssh astrouser@192.168.1.10

# You will be prompted to accept the keys, say "yes"
```

# Prepare for VNC

To run VNC and serve the desktop, including the log in window at startup, the existing GDM (Gnome 
Desktop Manager) will need to be removed and something like LightDM used instead. 

The reason for this is becuase the login screen screen is controlled by a desktop manager. Gnome 
Desktop Manager doesn't play too nice with VNC, so something else is needed.

To install LightDM and use it as default, the following needs to be run:

```bash
sudo apt install lightdm
```

This will run and then ask which Desktop Manager should be used by default. Select "lightdm" and 
then select "Ok". This is a text based UI, so arrow, tab, enter and space keys and such will need be used.

Reboot at this point.

# Install VNC

Once the system is back and running, log in to the terminal (remotely is fine) and do the following:

- Install x11vnc and setup
- Install a dummuy display device (for when there is no monitor attached)
- Create a VNC password. 

```
# Install VNC Server
sudo apt-get install x11vnc

# install Dummy Display
sudo apt-get install xserver-xorg-video-dummy
```

After the software is installed, configuration for the dummy display is needed. Add the following text 
to a file the X11 configuration folder:

```text
Section "Monitor"
  Identifier "Monitor0"
  HorizSync 28.0-80.0
  VertRefresh 48.0-75.0
  # https://arachnoid.com/modelines/
  # 1920x1080 @ 60.00 Hz (GTF) hsync: 67.08 kHz; pclk: 172.80 MHz
  Modeline "1920x1080_60.00" 172.80 1920 2040 2248 2576 1080 1081 1084 1118 -HSync +Vsync
EndSection

Section "Device"
  Identifier "Card0"
  Driver "dummy"
  VideoRam 256000
EndSection

Section "Screen"
  DefaultDepth 24
  Identifier "Screen0"
  Device "Card0"
  Monitor "Monitor0"
  SubSection "Display"
    Depth 24
    Modes "1920x1080_60.00"
  EndSubSection
EndSection
```

One possible way of adding this to the file is to do the following command:
```bash
# The following opens up an editor and allows the above text to be copied in
sudo nano /etc/X11/xorg.conf.d/xorg.conf
```

Next, there needs to be a password for X11VNC to prompt for during client connections, this
can be set with the following command:

```bash
# Replace <yourpassword> with your real password
sudo x11vnc -storepasswd <yourpassword> /etc/x11vnc.pass
```

Next, create an x11vnc.service file in the systemd services folder. The contents should contain:

```text
[Unit]
Description="x11vnc"
Before=x11vnc.service
After=multi-user.target

[Service]
ExecStart=/usr/bin/x11vnc -xkb -noxrecord -shared -forever -nowf -repeat -display :0 -geometry "1920x1080" -auth guess -rfbauth /etc/x11vnc.pass -o /var/log/x11vnc.log
ExecStop=/usr/bin/killall x11vnc
Restart=on-failure
RestartSec=2

[Install]
WantedBy=multi-user.target
```

Next, this service will need to be registered to Ubuntu to start on restart:

```bash
# Tells Ubuntu to reload possible list of services, including the one we made above
sudo systemctl daemon-reload

# Start the new service
sudo systemctl start x11vnc

# Remember to start the new service on each boot
sudo systemctl enable x11vnc
```

Next, install NoVNC. This project allows access to any VNC Server using the browser and JavaScript.
It can be used to control this new PC from just about any browser (but not IE). This will be installed
via Snap because it is just too easy not to do it that way.

```bash
# Install NoVNC
sudo snap install novnc

# Tell snap to start NoVNC on startup using port 6080 and point to the VNC Server on the local system port 5900
sudo snap set novnc services.n6080.listen=6080 services.n6080.vnc=localhost:5900
```

At this point, the system can be rebooted and if all worked well pointing the browser to:

```
Change <yourserverip> to the right value...

http://<yourserverip>:6080/vnc.html?host=<yourserverip>&port=6080
```

