# homework-LDAP

Цель домашнего задания
---
- Научиться настраивать LDAP-сервер и подключать к нему LDAP-клиентов


---
ОС для настройки: 
- CentOS Linux release 7.9.2009 (Core) (особо сложностей из-за отличий с методичек не увидел, кроме того, что не увидел вывода данных сертификата после установки сервера)

Vagrant версии 2.4.1

VirtualBox версии 7.0.18

**Примечание** В Vagrantfile были внесены изменения в части урезания ресурсов, выделяемых для VM, но оставлено достаточно. 

---
- Этап 1: Создаем Vagrantfine, запускаем ВМ.

После того, как VM все таки развернулись, создадим\обновим файлы для приведения стенда к работе:

**На всех хостах с ОС Centos:**
1. Меняем репозиторий, потому что из коробки не работает
2. Менял на https://mirror.yandex.ru/centos/centos/7/os/x86_64/
3. Обновляем пакеты, устанавливаем ПО: yum install -y nano 


---
- Этап 2: Установка FreeIPA сервера

**Примечание**: В методичке предлагается сначала настроить FreeIPA-сервер. Однако анализируя команды, их понадобился выполнить скорее всего и на клиентах. Так что следующих список команд относится ко всем хостам. 


```bash
#Установим часовой пояс:
timedatectl set-timezone Europe/Moscow

#Установим утилиту chrony:
yum install -y chrony

#Запустим chrony и добавим его в автозагрузку:
systemctl enable chronyd

#Выключим Firewall:
systemctl stop firewalld

#Отключаем автозапуск
Firewalld: systemctl disable firewalld

#ОстановимSelinux:
setenforce 0
```

![images2](./images/ldap_1.png)
![images2](./images/ldap_2.png)

Далле нужно внести изменения в /etc/selinux/config. Меняем Selinux на disabled


![images2](./images/ldap_3.png)


**Вносим изменения в файл /etc/hosts, так как настройка DNS сервера не предполагается**


```bash
nano /etc/hosts

#добавляем в конец
192.168.57.10 ipa.otus.lan ipa
```

![images2](./images/ldap_4.png)


---
**Далее показана настройка именно Сервера. Клиенты тут не участвуют**

**Установим FreeIPA-сервер**


```bash
yum install -y ipa-server
```

Хм, а он много чего себе качает, однако 

![images2](./images/ldap_5.png)



**Запустим скрипт установки**


```bash
ipa-server-install


```
```bash
Далее, нам потребуется указать параметры нашего LDAP-сервера, после ввода каждого параметра нажимаем Enter,
если нас устраивает параметр, указанный в квадратных скобках, то можно сразу нажимать Enter:

Do you want to configure integrated DNS (BIND)? [no]: no
Server host name [ipa.otus.lan]: <Нажимаем Enter>
Please confirm the domain name [otus.lan]: <Нажимем Enter>
Please provide a realm name [OTUS.LAN]: <Нажимаем Enter>
Directory Manager password: <Указываем пароль минимум 8 символов>
Password (confirm): <Дублируем указанный пароль>
IPA admin password: <Указываем пароль минимум 8 символов>
Password (confirm): <Дублируем указанный пароль>
NetBIOS domain name [OTUS]: <Нажимаем Enter>
Do you want to configure chrony with NTP server or pool address? [no]: no
```


![images2](./images/ldap_6.png)


---

Как и говорилось в методичке, установка не быстрая. Я предполагаю, что долго генерируется сертификат, если вспоминать предыдущие ДЗ, где тоже этот процесс был не быстрым 


![images2](./images/ldap_7.png)


**Признаться**, меня удивила ошибка, которая вылезла после установки. 


![images2](./images/ldap_8.png)


Почему сервер не смог разрезолвить сам себя?).... И это при том, что в hosts внесено изменение. DNS не установлен.. Странно, в общем. 


---
**Надо проверить, что сервер Kerberos может выдать нам билет**


```bash
kinit admin                   #Формируем билет
Password for admin@OTUS.LAN:  #Указываем Directory Manager password
klist                         #Запросим список билетов Kerberos
```

![images2](./images/ldap_9.png)


**!!** Для удаление полученного билета воспользуемся командой: 


```bash
kdestroy
```

---
Чтобы попасть в Web-интерфейс сервера со своей, основной хостовой машины, надо внести изменения в файл hosts, дописав в конец строку 
192.168.57.10 ipa.otus.lan

*В Unix-based системах файл хост находится по адресу /etc/hosts, в Windows — c:\Windows\System32\Drivers\etc\hosts. Для добавления строки потребуются права администратора.*


---
Вносим изменения и стучимся в web через браузер. 

![images2](./images/ldap_10.png)


**Но он не пускает** и накатывает настольгия :) PS. Столько уже прошло лет с моей первой работы, а я как-то умудрился вспомнить, что уже сталкивался с таким, надо потыкать "отмена" несколько раз и…. Вуа-ля) Попадаем на истинную страницу авторизации


![images2](./images/ldap_11.png)

![images2](./images/ldap_12.png)


---
**Пришло время настроить клиентов**


Устанавливаем клиента freeipa


```bash
yum -y install freeipa-client
```

![images2](./images/ldap_13.png)


---
**Далее, нужно добавить хост клиента к домену. Для этого можно пройти такого же мастера (почти такого же), как при установке сервера, где поэтапно вводится всё. Но, функционал команды позволяет сделать это в одну строку.** (один минус, пароль светить приходится)



```bash
echo -e "yes\nyes" | ipa-client-install --mkhomedir --domain=OTUS.LAN --server=ipa.otus.lan --no-ntp -p admin -w some_pass
```

**Параметры команды**

```bash
--domain — имя домена
--server — имя FreeIPA-сервера
--no-ntp — не настраивать дополнительно ntp (мы уже настроили chrony)
-p — имя админа домена
-w — пароль администратора домена (IPA password)
--mkhomedir — создать директории пользователей при их первом логине
```

![images2](./images/ldap_14.png)


---
После подключения хостов к FreeIPA-сервер нужно проверить, что мы можем получить билет от Kerberos сервера: kinit admin
Если подключение выполнено правильно, то мы сможем получить билет, после ввода пароля. 


![images2](./images/ldap_15.png)


**проверим работу LDAP** для этого на сервере FreeIPA создадим пользователя и попробуем залогиниться к клиенту

```bash
#Авторизируемся на сервере:
kinit admin
#Создадим пользователя
otus-user
```


![images2](./images/ldap_16.png)

---
**Идем на клиента**, например, client2


**выполним команду**


```bash
kinit otus-user
```

![images2](./images/ldap_17.png)


На этом процесс добавления хостов к FreeIPA-серверу завершен.
**Однако странно**, что после смены пароля, мы ничего не увидели, хотя бы слова "OK"

---
Для наглядности хочется прогуляться в UI сервера через браузер, обновляем вкладку, и видим пользователя. По крайней мере, он добавился. 

![images2](./images/ldap_18.png)


---
А вот если попробовать подключиться с клиента, но ошибиться в пароле, то получаем отбивку. Ну и + проверили, что Client1 тоже цепляется, хотя на запросе билета была тоже проверка доступности сервера


![images2](./images/ldap_19.png)







Участвуют testClient1 и testServer1

---
**На хосте testClient1**

```bash
#Создаем файл
nano /etc/sysconfig/network-scripts/ifcfg-vlan1

VLAN=yes
#Тип интерфейса - VLAN
TYPE=Vlan
#Указываем физическое устройство, через которые будет работать VLAN
PHYSDEV=eth1
#Указываем номер VLAN (VLAN_ID)
VLAN_ID=1
VLAN_NAME_TYPE=DEV_PLUS_VID_NO_PAD
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=none
#Указываем IP-адрес интерфейса
IPADDR=10.10.10.254
#Указываем префикс (маску) подсети
PREFIX=24
#Указываем имя vlan
NAME=vlan1
#Указываем имя подинтерфейса
DEVICE=eth1.1
ONBOOT=yes
```

![images2](./images/vlan_3.png)

---
**На хосте testServer1**

```bash
#Создаем файл (отличие от предыдущего конфига в IP адресе
nano /etc/sysconfig/network-scripts/ifcfg-vlan1

VLAN=yes
#Тип интерфейса - VLAN
TYPE=Vlan
#Указываем физическое устройство, через которые будет работать VLAN
PHYSDEV=eth1
#Указываем номер VLAN (VLAN_ID)
VLAN_ID=1
VLAN_NAME_TYPE=DEV_PLUS_VID_NO_PAD
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=none
#Указываем IP-адрес интерфейса
IPADDR=10.10.10.1
#Указываем префикс (маску) подсети
PREFIX=24
#Указываем имя vlan
NAME=vlan1
#Указываем имя подинтерфейса
DEVICE=eth1.1
ONBOOT=yes
```

![images2](./images/vlan_4.png)

Перезапускаем сеть на **testServer1** и **testClient1**

```bash
systemctl restart NetworkManager
```

![images2](./images/vlan_5.png)


---
Проверим настройку интерфейса, если настройка произведена правильно, то с хоста testClient1 будет проходить ping до хоста testServer1


```bash
ip a
ping 10.10.10.1
```

![images2](./images/vlan_6.png)
![images2](./images/vlan_7.png)


---
**Настройка VLAN на Ubuntu**


Участвуют testClient2 и testServer2

---
**На хосте testClient2**


```bash
#Редактируем файл
nano /etc/netplan/50-cloud-init.yaml


# network: {config: disabled}
network:
    ethernets:
        enp0s3:
            dhcp4: true
            dhcp6: true
            match:
                macaddress: 02:8f:be:f2:49:93
            set-name: enp0s3
        enp0s8: {}
    vlans:
        #Имя VLANа
        vlan2:
            #Указываем номер VLAN`а
            id: 2
            #Имя физического интерфейса
            link: enp0s8
            #Отключение DHCP-клиента
            dhcp4: no
            #Указываем ip-адрес
            addresses: [10.10.10.254/24]

    version: 2
```

---
**На хосте testServer2** Разница только лишь в IP адресе


```bash
#Редактируем файл
nano /etc/netplan/50-cloud-init.yaml


# network: {config: disabled}
network:
    ethernets:
        enp0s3:
            dhcp4: true
            dhcp6: true
            match:
                macaddress: 02:8f:be:f2:49:93
            set-name: enp0s3
        enp0s8: {}
    vlans:
        #Имя VLANа
        vlan2:
            #Указываем номер VLAN`а
            id: 2
            #Имя физического интерфейса
            link: enp0s8
            #Отключение DHCP-клиента
            dhcp4: no
            #Указываем ip-адрес
            addresses: [10.10.10.1/24]

    version: 2
```

---
**Замечание!** 
В текущем конфиге очень важны отступы! 
В общем, правило такое: С каждым новым разделом - отступ 4(четыре) пробела. Для демонстрации и на будущее, привожу правильные и не правильный скрин конфига, а так же скрин с маркерами отступов


**НЕ_правильный конфиг**

![images2](./images/vlan_8.png)


**Норимальный конфиг**

![images2](./images/vlan_9.png)


**Отступы в конфиге**

![images2](./images/vlan_10.png)


**Симптомы проблем:** Допольно неплохо говорится, где ошибки, когда пытаешься применить конфигурацию. Которая, кстати, не применяется.


![images2](./images/vlan_11.png)



---
После настройки второго VLAN`а ping должен работать между хостами testClient1, testServer1 и между хостами testClient2, testServer2. Проверим

между хостами testClient2, testServer2

![images2](./images/vlan_12.png)


между хостами testClient1, testServer1

![images2](./images/vlan_13.png)




---
- Этап 3: Настройка LACP между хостами inetRouter и centralRouter


**Bond интерфейс будет работать через порты eth1 и eth2**


**Работа выполняется на обоих хостах, inetRouter и centralRouter**

---
```bash
#Создаем конфиг для eth1
nano /etc/sysconfig/network-scripts/ifcfg-eth1

#Имя физического интерфейса
DEVICE=eth1
#Включать интерфейс при запуске системы
ONBOOT=yes
#Отключение DHCP-клиента
BOOTPROTO=none
#Указываем, что порт часть bond-интерфейса
MASTER=bond0
#Указываем роль bond
SLAVE=yes
NM_CONTROLLED=yes
USERCTL=no
```

---
```bash
#Создаем конфиг для eth2
nano /etc/sysconfig/network-scripts/ifcfg-eth2

#Имя физического интерфейса
DEVICE=eth2
#Включать интерфейс при запуске системы
ONBOOT=yes
#Отключение DHCP-клиента
BOOTPROTO=none
#Указываем, что порт часть bond-интерфейса
MASTER=bond0
#Указываем роль bond
SLAVE=yes
NM_CONTROLLED=yes
USERCTL=no
```

![images2](./images/vlan_14.png)
![images2](./images/vlan_15.png)


---
**Настраиваем bond-интерфейс, тоже делается на inetRouter и centralRouter** Разница в конфигах только в IP адресе


---
```bash
#Создаем конфиг
nano /etc/sysconfig/network-scripts/ifcfg-bond0

DEVICE=bond0
NAME=bond0
#Тип интерфейса — bond
TYPE=Bond
BONDING_MASTER=yes
#Указаваем IP-адрес 
IPADDR=192.168.255.1
#Указываем маску подсети
NETMASK=255.255.255.252
ONBOOT=yes
BOOTPROTO=static
#Указываем режим работы bond-интерфейса Active-Backup
# fail_over_mac=1 — данная опция «разрешает отвалиться» одному интерфейсу
BONDING_OPTS="mode=1 miimon=100 fail_over_mac=1"
NM_CONTROLLED=yes
```


![images2](./images/vlan_16.png)


---
После создания данных конфигурационных файлов необходимо перезапустить сеть:


```bash
systemctl restart NetworkManager
```


На некоторых версиях RHEL/CentOS перезапуск сетевого интерфейса не запустит bond-интерфейс, в этом случае рекомендуется перезапустить хост.


---
**После настройки агрегации портов, необходимо проверить работу bond-интерфейса**. Для наглядности на скринах показано системное время


**На хосте inetRouter (192.168.255.1) запустим ping до centralRouter (192.168.255.2)**

![images2](./images/vlan_17.png)


**Не отменяя ping, на centralRouter выключим интерфейс eth1**


![images2](./images/vlan_18.png)


**После данного действия ping не прекращается, так как трафик пойдёт по-другому порту**


![images2](./images/vlan_19.png)

