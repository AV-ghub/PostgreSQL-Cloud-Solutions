## Common usefull commands
> [Ubuntu Linux Change Hostname (computer name)](https://www.cyberciti.biz/faq/ubuntu-change-hostname-command/)

<details>

  <summary><h4>Change Hostname</h4></summary>
  
```
:~$ hostname
:~$ hostnamectl

# with reboot
sudo nano /etc/hostname
sudo nano /etc/hosts
sudo reboot

# without reboot
sudo hostname new-server-name-here
sudo nano /etc/hostname
sudo nano /etc/hosts

# via hostnamectl
hostnamectl set-hostname viveks-laptop
sudo nano /etc/hosts
```
</details>



> [How to Renew DHCP IP Address in Ubuntu](https://www.baeldung.com/linux/renew-dhcp-ip-address-ubuntu)

<details>

<summary><h4>Renew DHCP IP Address</h4></summary>
  
```
:~$ ps fax | grep dhclient
   2574 pts/0    S+     0:00  |       \_ grep --color=auto dhclient

:~$ ip addr

:~$ sudo dhclient -r

:~$ sudo dhclient -v
Internet Systems Consortium DHCP Client 4.4.1
Copyright 2004-2018 Internet Systems Consortium.
All rights reserved.
For info, please visit https://www.isc.org/software/dhcp/

:~$ ip addr
```
</details>

> [Прописка прокси](https://github.com/AV-ghub/PostgreSQL-Cloud-Solutions/blob/main/PostgreSQL/Admin/001%20Installation.md#ubuntu)

> [Guest tools installation](https://github.com/AV-ghub/PostgreSQL-Cloud-Solutions/blob/main/Linux/CentOS/Intro/Installation/001%20Installation.md#guest-tools-installation)

> [Интернет. Прозрачный прокси-сервер](https://confluence.speechpro.com/pages/viewpage.action?pageId=165486262)

<details>
  
<summary><h4>transparent proxy</h4></summary>
  
```

```
</details>
