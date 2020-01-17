# Setup a Debian Server

**Sven Haardiek, 2016-01-20**

Some time ago I decided to reinstall my root server from
[Netcup](https://www.netcup.de). I chose [Debian
Jessie](https://www.debian.org) as the operating system, because i have the
most experience with Debian.

In the following I will give a guide to my setup.

## Base Image

It is possible to use the [Netcup Server Control
Panel](https://www.vservercontrolpanel.de/Home) to install the base system on
your server. There you can easily click through the different options. Of
course you have to use the minimal Debian Jessie installation.  After that you
can choose between different predefined partitions. If you have no special
wishes, choose one great partition. So you can use the whole space without
having to partition something manually. At least use the mail notification. So
after your server is installed you got a mail with the long randomly set root
password.

You can log in with this password using the command

```bash
ssh root@<hostname>
```

on your preferred terminal emulator and take a first look at your fresh
installed operating system.

As a first step you should change the root password by using the

```bash
passwd
```

command.

## User

In my opinion it is not a good idea to use the *root* user to configure a
server. So you should create a new user *username* with

```bash
adduser <username>
```

You can use this user to configure also services and other stuff so you should
add this user to the group sudo. The minimal image of netcup do not have sudo
installed, so you have to install it manually

```bash
apt-get install sudo
```

and then add *username* to the group sudo

```bash
adduser <username> sudo
```

Now you can log out your server by using

```bash
exit
```

or using `Ctrl-D` and log in again with your user

```bash
ssh <username>@hostname
```

A password enabled log in is a security issue, so you should configure the SSH
server to disable it as described next.

## SSH Configuration

So now we want to configure SSH. First we create a SSH key on the computer we
want to connect to the server.

```bash
ssh-keygen -t rsa -b 4096
```

Now copy the output of

```bash
cat ~/.ssh/id_rsa.pub
```

to the file `/home/<username>/.ssh/authorized_keys`. If the directory
`/home/<username>/.ssh` does not exists, you can create it by executing

```bash
mkdir /home/<username>/.ssh
```

Now you can check, if you can log in the server without using a password
(except the password you set for the ssh key).

Since the ssh keys are generated during the installation done by netcup, you should renew them with

```bash
sudo rm /etc/ssh/ssh_host_*
sudo dpkg-reconfigure openssh-server
```

Edit `/etc/ssh/sshd_config` with your favorite editor (vim, nano, ...) and set `PermitRootLogin` and PasswordAuthentication `no`. To enable these changes restart SSH by

```bash
systemctl restart ssh
```

## Firewall

And last you should install a firewall. For simple configuration I would choose
[ufw](https://launchpad.net/ufw), because it is very easy to configure. So to install ufw use

```bash
sudo apt-get install ufw
```

Now you should configure the firewall. As a simple setup i would recommend a firewall configuration that denies per default all communication, enable the logging and of course enable communication via ssh. This configuration is achieved by the following commands

```bash
sudo ufw default deny
sudo ufw logging on
sudo ufw allow ssh/tcp
sudo ufw enable
```

## Final Words

Now you should have a basic installation of a debian server to have fun with.
So long...
