# Demo2025

## Модуль №1: Настройка сетевой инфраструктуры

### 1. Базовая настройка устройств

- **Имена устройств**  
  - **На роутерах HQ-RTR и BR-RTR:**  
    ```bash
    hostname {hq-rtr, br-rtr}
    ```
  - **На серверах и клиентах (HQ-SRV, HQ-CLI, BR-SRV):**  
    ```bash
    hostnamectl set-hostname {hq-srv, hq-cli, br-srv}.au-team.irpo; exec bash
    ```

- **IPv4 адресация**  
  - Настройка через `nmtui`.  
  - Адресация должна использовать приватные диапазоны (RFC1918):
    - 10.0.0.0/8  
    - 172.16.0.0/12  
    - 192.168.0.0/16

- **Локальные сети и маски:**
  - **HQ-SRV (VLAN100):** до 64 адресов, маска `/26` (255.255.255.192)
  - **HQ-CLI (VLAN200):** до 16 адресов, маска `/28` (255.255.255.240)
  - **BR-SRV:** до 32 адресов, маска `/27` (255.255.255.224)
  - **Управляющая сеть (VLAN999):** до 8 адресов, маска `/29` (255.255.255.248)

- **Пример таблицы адресации:**

  | Имя Устройства | IPv4                    | Интерфейс | NIC     | Шлюз        |
  |----------------|-------------------------|-----------|---------|-------------|
  | ISP            | NAT (inet)              | ens3      | Internet|             |
  |                | 172.16.4.14/28          | ens4      | ISP_HQ  |             |
  |                | 172.16.5.14/28          | ens5      | ISP_BR  |             |
  | HQ-RTR         | 172.16.4.1/28           | te0       | ISP_HQ  | 172.16.4.14 |
  |                | 192.168.0.81/29         | vl999     |         |             |
  |                | 192.168.0.62/26         | te1.100   |         |             |
  |                | 192.168.1.78/28         | te1.999   | HQ_NET  |             |
  |                | 172.16.0.1/30           | GRE       | TUN     |             |
  | HQ-SW          | 192.168.0.82/29         | ens3      | HQ_NET  | 192.168.0.81| 
  |                | -                       | ens4      | SRV_NET |             |
  |                | -                       | ens5      | CLI_NET |             |
  | HQ-SRV         | 192.168.0.2/26          | ens3      | SRV_NET | 192.168.0.62|
  | HQ-CLI         | 192.168.1.65/28 (DHCP)  | ens3      | CLI_NET | 192.168.1.78|
  | BR-RTR         | 172.16.5.1/28           | te0       | ISP_BR  | 172.16.5.14 |
  |                | 192.168.2.1/27          | te1       | BR_NET  |             |
  |                | 172.16.0.2/30           | GRE       | TUN     |             |
  | BR-SRV         | 192.168.2.2/27          | ens3      | BR_NET  | 192.168.2.1 |

---

### 2. Настройка ISP

- **IP-адресация интерфейсов:**
  - Интерфейс, подключённый к магистральному провайдеру – получает адрес по DHCP.
  - Маршруты по умолчанию:
    - **HQ-RTR:**  
      ```bash
      ip route 0.0.0.0/0 172.16.4.14
      ```
    - **BR-RTR:**  
      ```bash
      ip route 0.0.0.0/0 172.16.5.14
      ```

- **Пример конфигурации на EcoRouter для HQ-RTR:**
  ```bash
  en
  conf t
  int ISP
  ip add 172.16.4.1/28
  port te0
  service-instance toISP
  encapsulation untagged
  connect ip interface ISP
  wr mem
  ```

- **Аналогичная настройка для BR-RTR:**
  ```bash
  en
  conf t
  int ISP
  ip add 172.16.5.1/28
  port te0
  service-instance toISP
  encapsulation untagged
  connect ip interface ISP
  end
  wr mem
  ```

   ```bash
  en
  conf t
  int SRV
  ip add 192.168.2.1/27
  port te1
  service-instance SRV
  encapsulation untagged
  connect ip interface SRV
  end
  wr mem
  ```
  
- **Настройка динамической NAT на ISP:**
  ```bash
  echo net.ipv4.ip_forward=1 > /etc/sysctl.conf
  sysctl -p ( от sudo)
  dnf install iptables-services -y   
  systemctl enable ––now iptables  
  iptables –t nat –A POSTROUTING –s 172.16.4.0/28 –o ens3 –j MASQUERADE  
  iptables –t nat –A POSTROUTING –s 172.16.5.0/28 –o ens3 –j MASQUERADE  
  iptables-save > /etc/sysconfig/iptables  
  systemctl restart iptables 

  ```

---

### 3. Создание локальных учетных записей

- **На серверах (HQ-SRV, BR-SRV):**
  - Создание пользователя `sshuser`:
    ```bash
    useradd -m -u 1010 sshuser
    echo "sshuser:P@ssw0rd" | sudo chpasswd
    ```
  - Настройка sudo без пароля:
    ```bash
    nano /etc/sudoers
    ```
    Добавить строку:
    ```
    sshuser ALL=(ALL) NOPASSWD:ALL
    ```

- **На маршрутизаторах (HQ-RTR, BR-RTR):**
  - Создание пользователя `net_admin` на EcoRouter:
    ```plaintext
    conf t
    username net_admin
    password P@$$word
    role admin
    ```

---

### 4. Настройка виртуального коммутатора и VLAN

### (4.1). Динамическая трансляция адресов (NAT)

- **На HQ-RTR:**
  ```bash
  conf t
  ip nat pool nat1 192.168.0.1-192.168.0.254
  ip nat source dynamic inside-to-outside pool nat1 overload interface ISP
  ip nat pool nat2 192.168.1.65-192.168.1.79
  ip nat source dynamic inside-to-outside pool nat2 overload interface ISP

  - **На HQ-RTR:**
  ```bash
  conf t
  ip nat pool nat3 192.168.2.2-192.168.2.31
  ip nat source dynamic inside-to-outside pool nat3 overload interface ISP

#### Создание подсети управления (VLAN 999)

- **На HQ-RTR:**
  ```bash
  int vl999
  ip add 192.168.0.81/29
  description toSW
  port te1
  service-instance toSW
  encapsulation untaagged
  connect port te1 service-instance toSW
  end
  wr mem
  ```
  - **На HQ-RTR:**
  ```bash
  conf t
  int ISP
  ip nat outside
  ex
  int vl999
  ip nat inside
  ```
  - **На BR-RTR:**
  ```bash
  conf t
  int ISP
  ip nat outside
  ex
  int SRV
  ip nat inside
  ```
  
- **На HQ-SW:**  
  *(В nmtui прописываем айпи и шлюз, устанавливаем openvswitch,затем перед настройкой убедитесь, что интерфейс `ens3` отключён через nmtui)*
  ```bash
  dnf install openvswitch -y
  systemctl enable --now openvswitch
  ovs-vsctl add-br ovs0
  ovs-vsctl add-port ovs0 ens3
  ovs-vsctl set port ens3 vlan_mode=native-untagged tag=999 trunks=999,100,200
  ovs-vsctl add-port ovs0 ovs0-vlan999 tag=999 -- set Interface ovs0-vlan999 type=internal
  ifconfig ovs0-vlan999 inet 192.168.0.82/29 up
  ```

#### Настройка VLAN для серверов и клиентов

- **VLAN 100 для HQ-SRV:**
  - **На HQ-RTR:**
    ```bash
    int te1.100
    ip add 192.168.0.62/26
    port te1
    service-instance te1.100
    encapsulation dot1q 100
    rewrite pop 1
    connect ip interface te1.100
    ```
  - **На HQ-SW:**  
    *(Убедитесь, что `ens4` отключён в nmtui)*
    ```bash
    ovs-vsctl add-port ovs0 ens4
    ovs-vsctl set port ens4 tag=100 trunks=100
    ovs-vsctl add-port ovs0 ovs0-vlan100 tag=100 -- set Interface ovs0-vlan100 type=internal
    ifconfig ovs0-vlan100 up
    ```

- **VLAN 200 для HQ-CLI:**
  - **На HQ-RTR:**
    ```bash
    int te1.200
    ip add 192.168.1.78/28
    port te1
    service-instance te1.200
    encapsulation dot1q 200
    rewrite pop 1
    connect ip interface te1.200
    ex
    wr mem
    ```
  - **На HQ-SW:**  
    *(Убедитесь, что `ens5` отключён в nmtui)*
    ```bash
    ovs-vsctl add-port ovs0 ens5
    ovs-vsctl set port ens5 tag=200 trunks=200
    ovs-vsctl add-port ovs0 ovs0-vlan200 tag=200 -- set Interface ovs0-vlan200 type=internal
    ifconfig ovs0-vlan200 up
    ```
- **На HQ-RTR:**
    ```bash
    conf t
    int te1.100
    ip nat inside
    ex
    int te1.200
    ip nat inside
    ex
    wr mem
    ```
    

> **Отчёт:** Сведения по настройке коммутатора и выбору реализации разделения на VLAN занесите в отчёт.

---

### 5. Настройка безопасного удалённого доступа

- **Изменение SSH порта и SELinux:(на HQ-SRV и BR-SRV) **
  - Перевод SELinux в режим `permissive`:
    ```bash
    setenforce 0
    ```
    *(Также изменить в файле `/etc/selinux/config`)*
  - Изменение порта в `/etc/ssh/sshd_config`:
    ```
    Port 2024
    AllowUsers sshuser
    MaxAuthTries 2
    ```
  - Создание баннера:
    ```bash
    echo "Authorized access only" > /etc/ssh/sshd_banner
    echo Banner /etc/ssh/sshd_banner >> /etc/ssh/sshd_config
    systemctl restart sshd
    ```

---

### 6. Настройка IP-туннеля между офисами

- **На HQ-RTR:**
  ```bash
  Ip add 172.16.0.1/30
  Ip mtu 1476  
  ip ospf network broadcast  
  ip ospf mtu-ignore  
  Ip tunnel 172.16.4.1 172.16.5.1 mode gre  
  end  
  wr mem  
  Conf t
  Router ospf 1
  Ospf router-id  172.16.0.1
  network 172.16.0.0 0.0.0.3 area 0
  network 192.168.0.0 0.0.0.63 area 0
  network 192.168.1.78 0.0.0.15 area 0
  passive-interface default
  no passive-interface tunnel.1
  ```
- **На BR-RTR:**
  ```bash
  Interface tunnel.1
  Ip add 172.16.0.2/30
  Ip mtu 1476
  ip ospf mtu-ignore
  ip ospf network broadcast
  Ip tunnel 172.16.5.1 172.16.4.1 mode gre
  end
  Conf t
  Router ospf 1
  Ospf router-id 172.16.0.2
  Network 172.16.0.0 0.0.0.3 area 0
  Network 192.168.2.0 0.0.0.31 area 0
  Passive-interface default
  no passive-interface tunnel.1
  end
  wr mem
  ```

> **Примечание:** Выбор технологии – GRE или IP-in-IP – производится по усмотрению.

---

### 7. Динамическая маршрутизация

- **Цель:** Обеспечить доступ ресурсов одного офиса к другому посредством протокола link-state (например, OSPF).

- **Настройка аутентификации OSPF на EcoRouter:**

  **На HQ-RTR:**
  ```bash
  router ospf 1
    area 0 authentication
  interface tunnel.1
    ip ospf authentication-key ecorouter
  wr mem
  ```

  **На BR-RTR:**
  ```bash
  router ospf 1
    area 0 authentication ex
  interface tunnel.1
    ip ospf authentication-key ecorouter
  wr mem
  ```

> **Отчёт:** Занесите сведения о настройке и защите протокола в отчёт.

---

### 8. Динамическая трансляция адресов (NAT) 
[В отчете заносить не нужно будет, поэтому пункт сделан ранее]

- **Настройка шлюзов на серверах:**
  - **HQ-SRV:** Шлюз – `192.168.0.1/26`
  - **BR-SRV:** Шлюз – `192.168.1.1/27`

---

### 9. Настройка DHCP-сервера

- **Для офиса HQ (на HQ-RTR):**
  ```bash
  ip pool dhcpHQ 192.168.1.65-192.168.1.79
  en
  conf t
  dhcp-server 1
  pool dhcpHQ 1
  domain-name au-team.irpo
  mask 255.255.255.240  
  dns 192.168.0.2    
  gateway 192.168.1.78    
  end  
  wr mem
  ```
- **Клиентом является HQ-CLI (на hq-rtr) :**  
  ```bash
  conf t
  interface te1.200
  dhcp-server 1
  ```

### 10. Настройка DNS-сервера

- **Установка и базовая настройка:**
  ```bash
  dnf install bind -y
  systemctl enable --now named
  chattr -f +i /etc/resolv.conf
  cp /etc/named.conf /etc/named.conf.backup
  nano /etc/named.conf
  ```

  ![named1.png](https://github.com/dizzamer/DEMO2025/blob/main/dns.png)
  ![named2.png](https://github.com/dizzamer/DEMO2025/blob/main/dns2.png)


```bash
  chown -R root:named /var/named/master/
  chmod 750 /var/named/*
  chmod 750 /var/named/master/*
  systemctl restart named
  ```
Проверить зоны можно командой named-checkconf -z

Для полной работоспособности на HQ-CLI нужно установить в качестве dns севрера HQ-SRV:

```bash
  nano /etc/resolv.conf
  ```
11. Настройте часовой пояс на всех устройствах, согласно месту проведения экзамена.

```bash
  timedatectl set-timezone Europe/Moscow
  ```

## Модуль №2: Организация сетевого администрирования ОС

### 1. Доменный контроллер Samba

#### Подготовка сервера
  ```bash
  выставляем 192.168.0.2 в качестве нащего днс сервера на линке в nmtui и домен поиска au-team.irpo  
  setenforce 0  
  nano /etc/selinux  
  Замените в файле конфигурации /etc/selinux/config режим enforcing на permissive   
  dnf install samba* krb5* -y  
  Проверьте доступные серверы имен, просмотрев файл resolv.conf:  
  cat /etc/resolv.conf  
  В выводе должно отобразиться наш dns сервер и домен для поиска. 
  ```

#### Создание домена через `Samba DC`

- Удалите старый файл конфигурации:
  ```bash
  Создание резервных копий файлов  
  Переименуйте файл /etc/smb.conf, он будет создан позднее в процессе выполнения команды samba-tool.  
  cp /etc/samba/smb.conf /etc/samba/smb.conf.back  
  Создайте резервную копию используемого по умолчанию конфигурационного файла kerberos:  
  cp /etc/krb5.conf /etc/krb5.conf.back  
  ```
## Конфигурирование сервера с помощью утилиты samba-tool
  ```bash
  Файла /etc/samba/smb.conf быть не должно, он сам создаст.  
  rm -rf /etc/samba/smb.conf  
  samba-tool domain provision --use-rfc2307 --interactive
  ```
  ![named1.png](https://github.com/dizzamer/DEMO2025/blob/main/samba-toolprovision.png)

#### Удаление использования службы dns
  ```bash
  systemctl stop samba  
  Подчищаем, все где могут храниться наши записи  
  sudo rm -rf /var/lib/samba/private/dns_update_cache  
  sudo rm -rf /var/lib/samba/private/dns_update_list  
  sudo rm -rf /var/lib/samba/private/dns  
  sudo rm -rf /var/lib/samba/private/dns.keytab  
  Добавляем в файл /etc/smb.conf следующее  
  ```
  ![named2.png](https://github.com/dizzamer/DEMO2025/blob/main/smbconf.png)

  ```bash
  Запустите и добавьте в автозагрузку службы samba и named:  
  systemctl enable named samba --now  
  systemctl status named samba  
  Проверка созданного домена с помощью команды samba-tool domain info au-team.irpo: 
  ```

  ![named2.png](https://github.com/dizzamer/DEMO2025/blob/main/samba-tool.png)

### Создайте 5 пользователей для офиса HQ: имена пользователей формата user№.hq. Создайте группу hq, введите в эту группу созданных пользователей
  ```bash
  sudo samba-tool group add hq  
  for i in {1..5}; do  
  sudo samba-tool user create user$i.hq P@ssw0rd$i  
  sudo samba-tool group addmembers hq user$i.hq  
  done    
  ```
### Введите в домен машину HQ-CLI
  ```bash
  Перед вводом необходимо ввести на HQ-SRV:  
  smbclient -L localhost -U administrator  
  Проверка kerberos: 
  kinit administrator@localhost    
  ```
### Настройка проивзодится на HQ-CLI:

  https://redos.red-soft.ru/base/redos-7_3/7_3-administation/7_3-domain-redos/7_3-domain-config/7_3-redos-in-samba/?nocache=1730793368537

### Пользователи группы hq имеют право аутентифицироваться на клиентском ПК
### Пользователи группы hq должны иметь возможность повышать привилегии для выполнения ограниченного набора команд: cat, grep, id. Запускать другие команды с повышенными привилегиями пользователи группы не имеют права
### Выполните импорт пользователей из файла users.csv. Файл будет располагаться на виртуальной машине BR-SRV в папке /opt

### 2. Настройка файлового хранилища


---

### 3. Настройка службы сетевого времени (chrony)

- **На HQ-RTR:**
  ```bash
  en  
  conf t  
  ntp server 172.16.4.1   
  ntp timezone UTC+3  
  end  
  wr mem
  ```
- **Настройка сервера chrony:**  
  Выберите стратум 5 и настройте клиентов: HQ-SRV, HQ-CLI, BR-RTR, BR-SRV.

---

### 4. Настройка Ansible

- **Установка Ansible:**
  ```bash
  dnf install ansible -y
  ```
  
### 1) В файле /etc/ansible/hosts.yml прописать все хосты, на которые будет распространяться конфигурация.  
### Хосты можно разделить по группам, а так же, если у вас есть домен, то автоматически экспортировать список из домена.  
### Можно прописывать как ip адреса так и имена хостов, если они резолвятся DNS ом в сети.  
  ```bash
    nano /etc/ansible/inventory.yml

   clients:  
     hosts:  
       hq-cli.au-team.irpo:  
   servers:  
     hosts:  
       hq-srv.au-team.irpo:  
   routers:  
     hosts:  
       hq-rtr.au-team.irpo:  
       br-rtr.au-team.irpo: 
       
    2) Подключение к хостам осуществляется по протоколу ssh с помощью rsa ключей.    
    Сгенерировать серверный ключ можно командой ниже. При её выполнении везде нажмите Enter.    
    ssh-keygen -C "$(whoami)@$(hostname)-$(date -I)"  
    
    3) Далее нужно распространить ключ на все подключенные хосты.  
    Распространить ключи на хосты можно командой:  
    ssh-copy-id root@server  
    где:  
    root - это пользователь, от имени которого будут выполняться плейбуки;  
    server - IP-адрес хоста.
   ```

---

### 5. Развертывание MediaWiki в Docker

Установка необходимых пакетов:  
  ```bash
  dnf install docker-ce docker-ce-cli docker-compose -y  
  systemctl enable docker --now
  Добавляем текущего пользователя в группу докер  
  usermod -aG docker $USER
  ```

- **Создание файла `wiki.yml` в домашней директории пользователя:**
  ```yaml
  services:
    MediaWiki:
      container_name: wiki
      image: mediawiki
      restart: always
      ports:
        - 80:8080
      links:
        - database
      volumes:
        - images:/var/www/html/images
        # - ./LocalSettings.php:/var/www/html/LocalSettings.php
    database:
      container_name: mariadb
      image: mariadb
      environment:
        MYSQL_DATABASE: mediawiki
        MYSQL_USER: wiki
        MYSQL_PASSWORD: WikiP@ssw0rd
        MYSQL_RANDOM_ROOT_PASSWORD: 'yes'
      volumes:
        - dbvolume:/var/lib/mysql
  volumes:
    dbvolume:
      external: true
    images:
  ```
- **Запуск стека контейнеров:**
  ```bash
  docker compose -f wiki.yml up -d
  ```
- **Дальнейшие действия:**  
  После установки раскомментируйте строку с `LocalSettings.php` и выполните:
  ```bash
  docker-compose -f wiki.yml stop
  docker-compose -f wiki.yml up -d
  ```

---

### 6. Статическая трансляция портов

- **Пробросьте порт 80 в порт 8080 на BR-SRV на маршрутизаторе BR-RTR, для обеспечения работы сервиса wiki:**
  ```bash
  ip nat destination static tcp 192.168.1.2 80 192.168.1.65 8080
  ip nat destination static tcp 172.16.5.1 80 192.168.1.2 8080
  ```
- **Пробросьте порт 2024 в порт 2024 на HQ-SRV на маршрутизаторе HQ-RTR**
  ```bash
  Настройка производится на EcoRouter HQ-RTR:  
  ip nat destination static tcp 192.168.0.2 2024 192.168.1.65 2024  
  ```
- **Пробросьте порт 2024 в порт 2024 на BR-SRV на маршрутизаторе BR-RTR**
  ```bash
  Настройка производится на EcoRouter BR-RTR:  
  ip nat destination static tcp 192.168.1.2 2024 192.168.1.65 2024  
  ```

---

### 7. Настройка Moodle



### 8. Настройте веб-сервер nginx как обратный прокси-сервер на HQ-RTR

   ```bash
   Настройка производится на EcoRouter HQ-RTR:  
   en  
   conf t  
   filter-map policy ipv4 moodle 1  
   match 80 172.16.4.1/28 192.168.0.2/26 dscp 0
   set redirect hq-rtr.moodle.au-team.irpo  
   end  
   wr mem  
   en  
   conf t  
   redirect-url SITEREDIRECT  
   url hq-rtr.moodle.au-team.irpo  
   end  
   wr mem  

### 9. Установка Яндекс.Браузера

- **Требование:**  
  Установить Яндекс.Браузер для организаций удобным способом и зафиксировать результат в отчёте.









