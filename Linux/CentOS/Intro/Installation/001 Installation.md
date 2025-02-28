### Install
> https://lumpics.ru/installing-centos-on-virtualbox/

### Screen resolution
> https://www.youtube.com/watch?v=TqD-VPhdjLY
>
> https://inforkomp.com.ua/poleznoe/izmenenie-razresheniya-ekrana-v-centos-ubuntu.html
<details>
   <summary>Настройка</summary>
Фактически после запроса поддерживаемых параметров
<details>
<summary>Code</summary>
   
```
xrandr
Screen 0: minimum 1 x 1, current 1680 x 1050, maximum 8192 x 8192
Virtual1 connected primary 1680x1050+0+0 (normal left inverted right x axis y axis) 0mm x 0mm
   1680x1050     60.00*+  59.95  
   2560x1600     59.99  
   1920x1440     60.00
```
</details>
   
если они есть в списке, можно сразу установить требуемый режим

```
xrandr --output Virtual1 --mode "1680x1050"
```

Далее делаем скрипт для автозапуска
<details>
<summary>Code</summary>

```
cd ~
mkdir bin
touch ~/bin/fullscreen.sh
chmod +x ~/bin/fullscreen.sh
sudo nano ~/bin/fullscreen.sh
```
</details>
Пишем туда

```
#!/bin/bash

xrandr --output Virtual1 --mode "1680x1050"
```

И далее прописываем скрипт в автозапуск "Параметры системы"/"Запуск и завершение"/"Автозапуск".
Перезагружаемся.
</details>

### Guest tools installation
> https://routerus.com/how-to-install-virtualbox-guest-additions-on-centos-8/
<details>
   <summary>Настройка</summary>
   
> Обращаем внимание на версию вари! Тулзы не поставятся на новых осях со старой вари!

Создаем новый каталог и монтируем файл ISO:

``` 
sudo mkdir -p /mnt/cdrom
sudo mount /dev/cdrom /mnt/cdrom
```

Перейдите во вновь созданный каталог и выполните сценарий VBoxLinuxAdditions.run чтобы начать установку гостевых дополнений:

``` 
cd /mnt/cdrom
sudo sh ./VBoxLinuxAdditions.run --nox11
```

Параметр --nox11 указывает программе установки не создавать окно xterm.
Результат будет выглядеть следующим образом:

```
Verifying archive integrity... All good. Uncompressing VirtualBox 6.0.16 Guest Additions for Linux........ ... ... VirtualBox Guest Additions: Starting.
```

Перезагрузите гостевую систему CentOS, чтобы изменения вступили в силу:

``` 
sudo shutdown -r now
```

После загрузки виртуальной машины войдите в нее и убедитесь, что установка прошла успешно и модуль ядра загружен с помощью команды lsmod :

```
lsmod | grep vboxguest
```

Результат будет выглядеть примерно так:

```
vboxguest 348160 2 vboxsf
```
</details>

### [Change Hostname](https://github.com/AV-ghub/PostgreSQL-Cloud-Solutions/blob/main/Linux/Ubuntu/001%20Installation.md#change-hostname)

<details>
<summary><H3>Proxy</H3></summary>

```
sudo nano /etc/yum.conf

proxy=http://proxy.domain:port_number
http_proxy=http://proxy.domain:port_number
https_proxy=http://proxy.domain:port_number
no_proxy=.domain,.domain1,.domain2
```
</details>

<details>
<summary><H3>SSH</H3></summary>
<href>https://www.cyberciti.biz/faq/how-to-install-ssh-on-ubuntu-linux-using-apt-get/</href>
   
```
install openssh-client
sudo systemctl enable ssh

sudo ufw allow ssh

sudo systemctl start ssh
sudo systemctl stop ssh
sudo systemctl restart ssh

sudo systemctl status ssh

ssh user@server-ip-here
```
</details>





