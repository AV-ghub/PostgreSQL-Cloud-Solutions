## Install
> https://lumpics.ru/installing-centos-on-virtualbox/

## Screen resolution
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

## Guest tools installation
> https://routerus.com/how-to-install-virtualbox-guest-additions-on-centos-8/
<details>
   <summary>Настройка</summary>
   
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


