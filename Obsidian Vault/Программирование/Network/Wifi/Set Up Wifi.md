# Проверка оборудования

Процедуры проверки оборудования позволяют убедиться, что оборудование опознано операционной системой и готово к настройке. Простейшие проверки можно выполнить без установки дополнительных пакетов, для более детальных проверок понадобятся дополнительные пакеты.

## Установка пакетов для проверки оборудования

Для выполнения простейших проверок дополнительные пакеты не требуются. Для детальной проверки установить пакеты rfkill и iw:
```bash
sudo apt install rfkill iw
```

## Проверка и подготовка оборудования

Убедиться, что устройство действительно подключено (дополнительные пакеты не требуются): 
```bash
lsusb

---- Вывод
Bus 002 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub  
Bus 001 Device 004: ID 148f:5370 Ralink Technology, Corp. RT5370 Wireless Adapter  
Bus 001 Device 002: ID 0627:0001 Adomax Technology Co., Ltd  
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
```

Вывод команды показывает наличие подключенного USB-устройства "Ralink Technology, Corp. RT5370 Wireless Adapter"

Убедиться, что устройство опознано как сетевой адаптер (дополнительные пакеты не требуются):
```bash
ip a

---- Вывод
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000  
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00  
    inet 127.0.0.1/8 scope host lo  
       valid_lft forever preferred_lft forever  
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000  
    link/ether 52:54:00:0f:e7:bf brd ff:ff:ff:ff:ff:ff  
    inet 192.168.56.31/24 brd 192.168.56.255 scope global noprefixroute dynamic eth0  
       valid_lft 966sec preferred_lft 966sec  
    inet6 fe80::5054:ff:fe0f:e7bf/64 scope link noprefixroute   
       valid_lft forever preferred_lft forever  
3: wlan0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc mq state DOWN group default qlen 1000  
    link/ether 8a:69:8b:6c:25:92 brd ff:ff:ff:ff:ff:ff
```

Вывод команды показывает наличие сетевого адаптера wlan0;

Убедиться, что устройство управляется (опознано) службой Network Manager (дополнительные пакеты не требуются):
```bash
sudo nmcli dev status

--- Вывод
DEVICE  TYPE      STATE           CONNECTION           
eth0    ethernet  подключено      Проводное соединение 1  
wlan0   wifi      отключено       --
```

Убедиться, что использование устройства разрешено (в частности, устройство не отключено аппаратно или настройками BIOS). Требуется пакет rfkill. Команда:
```bash
sudo rfkill list

--- Вывод
0: phy0: Wireless LAN  
        Soft blocked: no  
        Hard blocked: no
```

Вывод команды показывает, что устройство с именем phy0 не заблокировано ("no"). Если устройство заблокировано ('yes"), то: Если устройство заблокировано программно ("Soft blocked: yes"), то можно попробовать разблокировать его командой:
```bash
sudo rfkill unblock <номер_устройства> # например sudo rfkill unblock wifi
```

Если это не помогает, то, возможно, в системе отсутствуют драйверы для устройства;

Если устройство заблокировано аппаратно ("Hard blocked: yes"), то следует искать способ разблокировки в настройках оборудования (например, включить WiFi кнопкой на ноутбуке);

Убедиться, что устройство поддерживает нужные режимы работы (в частности, режим точки доступа - AP (Access Point)). Требуется пакет iw. Команда может выглядеть так:
```bash
sudo iw list | grep "Supported interface modes:" -A 10

--- Вывод
        Supported interface modes:  
                 * IBSS  
                 * managed  
                 * AP  
                 * AP/VLAN  
                 * monitor  
                 * mesh point  
        Band 1:  
                Capabilities: 0x17e  
                        HT20/HT40  
                        SM Power Save disabled
```

Вывод команды показывает наличие режима работы AP.


# Программное отключений WiFi

При наличии установленного пакета rfkill получить список модулей можно командой:

```bash
sudo rfkill list
```

  
Программно отключить модуль WiFi можно командой:

```bash
sudo rfkill block <номер_устройства>
```


# Настройка адаптера WiFi как точки доступа с выходом во внешнюю сеть

Далее предполагается, что WiFi-адаптер установлен в компьютере с сетевой картой с именем eth0, подключенной к внешней сети (Интернет или сеть предприятия). Для корректного подключения клиентов выполняются следующие настройки:

1. Настройки сетевого подключения;
2. Настройка службы DHCP для назначения IPv4-адресов WiFi-клиентам при их подключении;
3. Настройка правил IPTABLES для разрешения перенаправления сетевых пакетов от WiFi-клиентов (от WiFi адаптера) во внешнюю сеть.

## Настройка точки доступа при использовании Network Manager

При использовании Network Manager для включения точки доступа WiFi выполнить следующие операции:

### Настройка сетевого подключения

Создать и настроить сетевое подключение. Это можно сделать через графический интерфейс Network Manager или из командной строки следующими командами:

Создание "пустого" соединения с именем соединения alse и с идентификатором сети (SSID) alse-wifi:
```bash 
sudo nmcli con add type wifi ifname wlan0 con-name alse autoconnect yes ssid alse-wifi
```

Назначить параметры WiFi:
```bash
sudo nmcli con mod alse 802-11-wireless.mode ap 802-11-wireless.band bg
```

Назначить IPv4-адрес:
```bash
sudo nmcli con mod alse ipv4.method shared ipv4.addresses 10.42.0.1/24 gw4 10.42.0.1
```
Для примера использован адрес 10.42.0.1. Такой адрес автоматически назначается службой Network Manager при создании подключения через графический интерфейс;

Назначить параметры IPv6:
```bash
sudo nmcli con mod alse ipv6.method shared ipv6.ip6-privacy 0
```

Назначить параметры безопасности:
```bash
sudo nmcli con mod alse wifi-sec.key-mgmt wpa-psk wifi-sec.psk "<пароль_для_подключения>"
```

Активировать интерфейс:
```bash
sudo nmcli con up alse
```

После настройки и активации сетевого подключения убедиться, что служба Network Manager создала путь (route) для пересылки сетевых пакетов от WiFi-адаптера во внешнюю сеть:
```bash
ip route

--- Вывод
default via 192.168.56.1 dev eth0 proto dhcp metric 100  
default via 10.42.0.1 dev wlan0 proto static metric 600  
10.42.0.0/24 dev wlan0 proto kernel scope link src 10.42.0.1 metric 600  
192.168.56.0/24 dev eth0 proto kernel scope link src 192.168.56.31 metric 100
```

Путь для сетевых пакетов WiFi-адаптера: "10.42.0.0/24 dev wlan0 proto kernel scope link src 10.42.0.1 metric 600"

https://wiki.astralinux.ru/pages/viewpage.action?pageId=181668080