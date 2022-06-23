# roger-skyline-1

This project aims to teach the basics of system and network administration. You are required to, for example, set up a web server and add DDoS and port scanning protection to your VM.

# VM Part

I chose to install a VM with Debian as its OS with VirtualBox. The disk space of the VM is 8 GB.

During installation, I chose to do the disk partitioning manually and set a partitioning size of 4.5 GiB, which translates to roughly 4.2 GB. There is a requirement of having at least one 4.2 GB partition in this project.

# Network Part

## Create a new user

Logged in as root with

```bash
su
```

Created a new user with

```bash
sudo adduser <username>
```

Added the user to sudo group with

```bash
sudo usermod -aG sudo <username>
```

Also, I had to change the PATH for root by adding line

```
export PATH=$PATH:/usr/sbin
```

to the the root’s `~/.bashrc` . This was due to the VM not finding the commands that I was typing.

## Configure network

The goal is to disable the DHCP service and use a static IP.

I changed the VMs network connection to bridged adapter from VirtualBox settings.

Changing the network connection to work with a static IP instead of the DHCP service is done by modifying the file `/etc/network/interfaces`.

My network interface in the VM is called `enp0s3`, so that is the one I need to configure.

My configurations look like this:

```
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
auto enp0s3
iface enp0s3 inet static
	netmask 255.255.255.252
	gateway 10.13.254.254
	address 10.13.254.253
```

The netmask has to be /30 and the gateway is my host networks default gateway. The address is chosen by me by using the netmask.

In the case of error **Failed to bring up interface**, flushing the interface may work: `ip addr flush dev <interface>` .

## Configure SSH

Installed SSH with

```bash
sudo apt-get install ssh
```

From the file `/etc/ssh/sshd_config` , set password authentication to “no”.

I copied my host user’s public SSH key to the VM user’s `~/.ssh/authorized_keys` . The key can be copied to a remote host with `ssh-copy-id -i ~/.ssh/mykey user@host` , but password authentication (or other possible authentication method) has to be on during this action.

The default port of the SSHD service is 22, which I changed to 5085 (the number was chosen arbitrarily) by modifying `/etc/ssh/sshd_config`.

Finally, SSH root login has to be disabled, which is done by setting PermitRootLogin to “no” in the above file.

## Firewall

Installed UFW with

```bash
sudo apt-get install ufw
```

Added a rule to allow SSH via its port 5085

```bash
sudo ufw allow 5085/tcp
```

Enabled the firewall with

```bash
sudo ufw enable
```

## DoS protection

### General

Installed Fail2ban with

```bash
sudo apt-get install fail2ban
```

Copied `/etc/fail2ban/fail2ban.conf` as `fail2ban.local`. Repeated for `/etc/fail2ban/jail.conf`.

### SSH
Changed the LogLevel to VERBOSE in `/etc/ssh/sshd_config` in order to create logs detailed enough to catch DoS attacks.
Enabled the SSHD filter in `/etc/fail2ban/jail.local` by adding the line `enabled = true` to the filter. By default, all filters are not enabled. Also, added line `mode = ddos` to the SSHD filter in order to make it aggressive enough during DoS attacks.

Command `fail2ban-client start` starts the Fail2ban service.

### Apache

Enabled every jail in `/etc/fail2ban/jail.local` relating to Apache.

Added two new jails to `/etc/fail2ban/jail.local` :

```
[http-get-dos]
enabled  = true
port     = http,https
filter   = http-get-dos
logpath  = %(apache_access_log)s
maxretry = 20
findtime = 400
bantime  = 200

[http-post-dos]
enabled  = true
port     = http,https
filter   = http-get-dos
logpath  = %(apache_access_log)s
maxretry = 60
findtime = 29
bantime  = 6000
```

These jails are intended to catch GET and POST entries in the access log of Apache. I also needed to add the new filter to `/etc/fail2ban/filter.d` called `http-get-dos.conf` The filter is configured like this:

```
[Definition]

# This regex will match GET and POST entries in the logs
failregex = ^<HOST> -.*"(GET|POST).*
Ignoreregex =
```

<aside>
💡 Note that the `maxretry` value is low for `http-get-dos` for testing purposes only.

</aside>

## Service management

The command

```bash
systemctl list-unit-files --type=service --state=enabled
```

lists all enabled services. All unneeded services have to be disabled, which can be done with:

```bash
systemctl disable <service>
```

I also used the command

```bash
service --status-all
```

to check for enabled services. I checked the functionalities of each service which were listed with the above commands and disabled everything that I deemed unnecessary. The end result looks like this:

```
root@roger:/# service --status-all
 [ - ]  apache-htcacheclean
 [ + ]  apache2
 [ + ]  apparmor
 [ - ]  console-setup.sh
 [ + ]  cron
 [ + ]  dbus
 [ + ]  exim4
 [ + ]  fail2ban
 [ - ]  hwclock.sh
 [ - ]  keyboard-setup.sh
 [ + ]  kmod
 [ + ]  networking
 [ + ]  procps
 [ + ]  psad
 [ - ]  rsync
 [ + ]  rsyslog
 [ + ]  ssh
 [ - ]  sudo
 [ + ]  udev
 [ + ]  ufw
```

```
root@roger:/# systemctl list-unit-files --type=service --state=enabled
UNIT FILE                 STATE   VENDOR PRESET
apache2.service           enabled enabled      
apparmor.service          enabled enabled      
cron.service              enabled enabled      
fail2ban.service          enabled enabled      
getty@.service            enabled enabled      
networking.service        enabled enabled      
rsyslog.service           enabled enabled      
ssh.service               enabled enabled      
systemd-timesyncd.service enabled enabled      
ufw.service               enabled enabled      
update_packages.service   enabled enabled      

11 unit files listed.
```

## Protection from port scanning

Configured UFW to create proper logging rules for PSAD by adding the following lines to both `/etc/ufw/before*.rules` files:

```bash
-A INPUT -j LOG --log-tcp-options --log-prefix "[IPTABLES] "
-A FORWARD -j LOG --log-tcp-options --log-prefix "[IPTABLES] "
```

Installed PSAD

```bash
sudo apt-get install psad
```

Added the following modifications to `/etc/psad/psad.conf` :

- added hostname next to `HOSTNAME`
- changed the `IPT_SYSLOG_FILE` to point to the syslog file `/var/log/syslog`
- set `ENABLE_AUTO_IDS` to `Y` to manage firewall rulesets automatically (ban the IP)

<aside>
💡 As a side note, PSAD sends A LOT of email if you do multiple test scans with `nmap <server_ip>`

</aside>

## Scheduling tasks

### Creating a script

Created a script `update_packages.sh` which downloads updates and updates all packages which need updating. I stored the script in `/usr/local/bin` , because I was advised from multiple sources that it is a good location to store system-wide scripts. The output of the script should also be stored in a log file, which I created to `/var/log/update_script.log`. The script looks like this:

```bash
#!/bin/sh

LOG=/var/log/update_script.log
exec >> $LOG
sudo apt-get update && sudo apt-get upgrade -y
```

### Scheduling with crontab

Edited the file `/etc/crontab` by adding the following line at the end

```bash
0 4 * * 0 root sh /usr/local/bin/update_packages.sh
```

where the first five values indicate the time when the command is executed, the next indicates the user and the last is the actual command. The values indicating the time are as follows (starting from the left):  

- the minute
- the hour
- the day of the month
- the month
- the day of the week

In the above example, the command `sh /usr/local/bin/update_packages.sh` is executed by `root` at 4:00 AM every Sunday. An asterisk means any value.

### Run script at boot

As the above script requires internet connection to work, I created a systemd service to run the script as soon as the connection has been made. This is achieved by creating a service file under `/etc/systemd/system`. I named the file `update_packages.service` which looks like this:

```
[Unit]
Description=Update and upgrade all packages
Requires=network-online.target
After=network-online.target

[Service]
Type=oneshot
RemainAfterExit=no
ExecStart=/usr/local/bin/update_packages.sh

[Install]
WantedBy=multi-user.target
```

This service requires unit `network-online.target` and is executed after it. `Type=oneshot` indicates that the process is executed once and `RemainAfterExit=no` means that the service is inactive after the process execution. `ExecStart` is the script that is executed and `WantedBy=multi-user.target` indicates that the service should be started as part of system start-up.

### Script which sends email to root

I was required to write a script that checks modifications in the `/etc/crontab` file and sends email to root if the file has been modified. In order to accomplish this, I created a copy of the file `/etc/crontab` called `/etc/crontab_backup` with the `cp` command. Then, I wrote the following script to `/usr/local/bin/crontab_mod.sh`:

```bash
#!/bin/bash

ORIG=/etc/crontab
BACKUP=/etc/crontab_backup
if [ $ORIG -nt $BACKUP ]
then
	diff $ORIG $BACKUP | mail -s "Crontab file modified" root
	cp /etc/crontab /etc/crontab_backup
fi
```

The if statement `$ORIG -nt $BACKUP` checks if file $ORIG is newer (by modification date) than $BACKUP. If the statement is true, an email is sent to root with a subject of “Crontab file modified” and its content is the output of the command `diff $ORIG $BACKUP`. Then the script updates the backup file with the current content of the crontab file. I also scheduled this script to be run every day at midnight via cron.

# Web Part

I decided to use Apache to set up my web server. Installed it with

```bash
sudo apt-get install apache2
```

and opened the HTTP and HTTPS ports via UFW:

```bash
sudo ufw allow http
sudo ufw allow https
```

## SSL Certificate

The first thing is to enable mod_ssl, which is an Apache module for SSL encryption. Enabled the module with:

```bash
sudo a2enmod ssl
```

and restarted apache to enable the module:

```bash
sudo systemctl restart apache2
```

### Generate an SSL Certificate

The SSL certificate and a key file storing encrypted data securely can be created with:

```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/apache-selfsigned.key -out /etc/ssl/certs/apache-selfsigned.crt
```

where

- `req -x509` specifies an X.509 standard to be used for the certificate
- `-nodes` skips the option to use the certificate with a passphrase
- `-days 365` sets the time the certificate will be valid
- `-newkey rsa:2048` generates an RSA key that is 2048 bits long to sign the certificate
- `-keyout`  sets the path for the created key
- `-out` sets the path for the certificate

The command opens a prompt to type information about the website. The section *Common Name*

needs to be filled with either the domain or IP of the server.

### Configure Apache with SSL

The first thing to do is to create an Apache configuration file which will contain the SSL certificate and the key:

```bash
sudo vim /etc/apache2/sites-available/<mydomain_or_myip>.conf
```

I used the following configuration:

```html
<VirtualHost *:443>
	ServerName 10.13.254.253
	DocumentRoot /var/www/10.13.254.253

	SSLEngine on
	SSLCertificateFile /etc/ssl/certs/apache-selfsigned.crt
	SSLCertificateKeyFile /etc/ssl/private/apache-selfsigned.key
</VirtualHost>
```

The document root folder needs to created as well as an HTML file to create the website

```bash
sudo mkdir /var/www/<mydomain_or_myip>
sudo vim /var/www/<mydomain_or_myip>/index.html
```

Enabled the configuration file with

```bash
sudo a2ensite <mydomain_or_myip>.conf
```

Added redirection from HTTP to HTTPS by adding another VirtualHost block the configuration file:

```html
<VirtualHost *:80>
	ServerName 10.13.254.253
	Redirect / https://10.13.254.253/
</VirtualHost>
```

Finally, reloaded Apache to implement the changes.

## Deployment

I did a very simple automated deployment for my website. I copied the folder `/var/www/10.13.254.253` (the folder where I have my website files) into `/home/arttu/web_test` and wrote a script that will copy the files from the copied folder back to the source if there are any differences. The script will then reload the Apache service. The script looks like this:

```bash
#!/bin/bash

SRC=/home/arttu/web_test
DEST=/var/www/10.13.254.253
LOG=/var/log/web_deployment.log
exec >> $LOG 2>&1
diff $SRC $DEST
if [ $? -eq 1 ]
then
	cp $SRC/* $DEST
	sudo apachectl graceful
fi
```

I also scheduled this script to be run every day at 2 AM with cron.
