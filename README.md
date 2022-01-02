# Raspberry 4 model B - Web Server

This is the documentation of setting up my Raspberry Pi model B as a web server for static websites - and this in a
clean and secure way.

## Software

- [aarch64 alpine linux](https://alpinelinux.org/downloads/)
- lighttpd
- vsftpd

Find some explanation why I decided to take this in favor of others in the following sub chapters.

### Which Alpine version should I use?

According to the [Alpine wiki page for Raspberry Pi](https://wiki.alpinelinux.org/wiki/Raspberry_Pi) I decided to go
with `aarch64`:

> You should be safe using the armhf build on all versions of Raspberry Pi (including Pi Zero and Compute Modules);
> but it may perform less optimally on recent versions of Raspberry Pi. The armv7 build is compatible with Raspberry
> Pi 2 Model B. The aarch64 build should be compatible with Raspberry Pi 2 Model v1.2, Raspberry Pi 3 and Compute
> Module 3, and Raspberry Pi 4 model B.

### Lighttpd vs nginx

I decided to go with lighttpd since I only wanted to have a web server serving static files and it should have a minimal
footprint.

## Installation

### Prepare your SD card

**This will erase data on the SD card!** That said, be sure that your SD card is empty or the data is no longer relevant
for you!

You can use `fdisk` to perform the following - I was too lazy and simply used Ubuntu's disk tool.

- Add a FAT32 system partition
- Add a FAT32 data partition

### Alpine Linux

#### Prepare the bootable media

Plugin SD card to your machine. Check, where your SD card ended up on your system using `fdisk` or the disk tool. For me
it is `/dev/sda/` mounted on `/media/<user>/system`.

Mount the micro SD card on your machine and initialize the MBR to make it bootable:

```bash
$ sudo apt-get install mbr
$ sudo install-mbr /dev/sda
```

After that, mount the device and the downloaded alpine archive. Open a terminal and point it to the mounted archive

```bash
$ cd $alpine-archive-dir
$ cp -a .alpine-release * /media/<user>/system/
```

Safely eject the SD card.

#### Actually install Alpine

Next put the SD card into the Raspberry Pi, wire it and boot it up. Then make the system persistent using:

```bash
$ setup-alpine
```

This requires some configuration - do as pleased, I took defaults where possible. Finally reboot the system:

```bash
$ reboot
```

#### Configuration

Add a `usercfg.txt` file to `/boot/` - this will enable sound, minimize gpu memory and remove a black border around the
screen:

```
enable_uart=1
gpu_mem=32
disable_overscan=1
```

And perform following step to adjust the welcome text:

```bash
$ echo "Welcome to Raspi Web Server" > /etc/motd
```

To install e.g. vsftpd you have to enable community packages in `/etc/apk/repositories` - simply uncomment that line and
run:

```bash
$ apk update
$ apk upgrade
```

#### Configure chrony

```bash
$ vi /etc/chrony/chrony.conf
```

Add the following line at the bottom of the file.

```bash
makestep 60 10
```

Check the date, restart the service, and check the (now hopefully corrected) date again.

```bash
date
rc-service chronyd restart
date
```

#### There is no `shutdown`?

Simply use `poweroff` or `halt` to shot down and `reboot` to reboot - don't ask :-D

#### Sources

- https://wiki.alpinelinux.org/wiki/Raspberry_Pi_4_-_Persistent_system_acting_as_a_NAS_and_Time_Machine
- https://wiki.alpinelinux.org/wiki/Raspberry_Pi
- https://wiki.alpinelinux.org/wiki/Create_a_Bootable_Device#Format_USB_stick

### lighttpd

#### Installation

```bash
$ apk add lighttpd
$ rc-update add lighttpd default 
```

#### Configuration

Open `/etc/lighttpd/lighttpd.conf` and set

- `var.basedir= "/var/www"`
- `server.document-root = var.basedir + "/html"`
- `server.indexfiles = ("index.html", "index.htm")`
- `server.follow-symlink = "disable"`

Run `rc-service lighttpd restart` to see if the config can be processed and the server is still starting.

#### Sources

https://www.cyberciti.biz/tips/installing-and-configuring-lighttpd-webserver-howto.html

### vsftpd

#### Installation

To install vsftpd you have to enable community packages in `/etc/apk/repositories` - simply uncomment that line.

```bash
$ apk add vsftpd
$ rc-update add vsftpd default
```

#### Configuration

Add a file `/etc/vsftpd/vsftpd.userlist` containing users that are allowed to access the ftp server using a client.
E.g. `ftpuser`.

Create this user using:

```bash
$ addgroup ftpusers
$ adduser ftpuser -G ftpusers -h /var/www/html -s /sbin/nologin
$ chmod a-w /var/www/html
```

Edit `/etc/vsftpd/vsftpd.conf` and set

- `anonymous_enable=NO`
- `local_enable=YES`
- `write_enable=YES`
- `local_umast=022`
- `ftpd_banner=Welcome to Raspi FTP`
- `nopriv_user=ftpuser`
- `chroot_local_user=YES`
- `pasv_enable=YES`
- `pasv_min_port=10090`
- `pasv_max_port=10100`

#### Sources

https://www.layerstack.com/resources/tutorials/How-to-set-up-configure-secure-vsFTPd-on-Linux-Cloud-Servers

## Hardening

Set a secure password for root:

```bash
$ passwd
```

Setup a basic firewall:

```bash
$ apk add ufw
$ rc-update add ufw default
$ ufw allow 20
$ ufw limit 20/tcp
$ ufw allow 21
$ ufw limit 21/tcp
$ ufw allow in 10090:10100/tcp
$ ufw allow 80
$ ufw allow 443
```

TODO: https://gist.github.com/jumanjiman/f9d3db977846c163df12

## Automate all that?
