## Common usefull commands
> [Ubuntu Linux Change Hostname (computer name)](https://www.cyberciti.biz/faq/ubuntu-change-hostname-command/)

<details>
<summary>Commands</summary>
  
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
