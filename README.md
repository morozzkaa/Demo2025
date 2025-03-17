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
      ip route 0.0.0.0/0 172.16.4.1
      ```
    - **BR-RTR:**  
      ```bash
      ip route 0.0.0.0/0 172.16.5.1
      ```

- **Пример конфигурации на EcoRouter для HQ-RTR:**
  ```bash
  en
  conf t
  int ISP
  ip add 172.16.4.2/28
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
  ip add 172.16.5.2/28
  port te0
  service-instance toISP
  encapsulation untagged
  connect ip interface ISP
  wr mem
  ```

- **Настройка динамической NAT на ISP:**
  ```bash
  echo net.ipv4.ip_forward=1 > /etc/sysctl.conf
  dnf install iptables-services -y
  iptables -t nat -A POSTROUTING -o ens3 -j MASQUERADE
  iptables-save > /etc/sysconfig/iptables
  systemctl enable --now iptables
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
    username net_admin
    password P@$$word
    role admin
    ```

---

### 4. Настройка виртуального коммутатора и VLAN

#### Создание подсети управления (VLAN 999)

- **На HQ-RTR:**
  ```bash
  int te1.999
  ip add 192.168.0.81/29
  description toSW
  port te1
  service-instance toSW
  encapsulation dot1q 999 exact
  rewrite pop 1
  connect ip interface te1.999
  ```

- **На HQ-SW:**  
  *(Перед настройкой убедитесь, что интерфейс `ens3` отключён через nmtui)*
  ```bash
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
    ip add 192.168.0.1/26
    port te1
    service-instance te1.100
    encapsulation dot1q 100 exact
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
    ip add 192.168.0.65/28
    port te1
    service-instance te1.200
    encapsulation dot1q 200 exact
    rewrite pop 1
    connect ip interface te1.200
    ```
  - **На HQ-SW:**  
    *(Убедитесь, что `ens5` отключён в nmtui)*
    ```bash
    ovs-vsctl add-port ovs0 ens5
    ovs-vsctl set port ens5 tag=200 trunks=200
    ovs-vsctl add-port ovs0 ovs0-vlan200 tag=200 -- set Interface ovs0-vlan200 type=internal
    ifconfig ovs0-vlan200 up
    ```

> **Отчёт:** Сведения по настройке коммутатора и выбору реализации разделения на VLAN занесите в отчёт.

---

### 5. Настройка безопасного удалённого доступа

- **Изменение SSH порта и SELinux:**
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
    Banner /etc/ssh/sshd_banner
    ```
  - Создание баннера:
    ```bash
    echo "Authorized access only" > /etc/ssh/sshd_banner
    systemctl restart sshd
    ```

---

### 6. Настройка IP-туннеля между офисами

- **На HQ-RTR:**
  ```bash
  interface tunnel.1
  ip add 10.0.0.1/30
  ip ospf network broadcast
  ip ospf mtu-ignore
  ip tunnel 172.16.4.2 172.16.5.2 mode gre
  exit
  conf t
  router ospf 1
    ospf router-id 10.0.0.1
    network 10.0.0.0 0.0.0.3 area 0
    network 192.168.0.0 0.0.0.255 area 0
    passive-interface default
    no passive-interface tunnel.1
  ```

- **На BR-RTR:**
  ```bash
  interface tunnel.1
  ip add 10.0.0.2/30
  ip ospf mtu-ignore
  ip ospf network broadcast
  ip tunnel 172.16.5.2 172.16.4.2 mode gre
  exit
  conf t
  router ospf 1
    ospf router-id 10.0.0.2
    network 10.0.0.0 0.0.0.3 area 0
    network 192.168.1.0 0.0.0.31 area 0
    passive-interface default
    no passive-interface tunnel.1
  ```

> **Примечание:** Выбор технологии – GRE или IP-in-IP – производится по усмотрению.

---

### 7. Динамическая маршрутизация

- **Цель:** Обеспечить доступ ресурсов одного офиса к другому посредством протокола link-state (например, OSPF).

- **Настройка аутентификации OSPF на EcoRouter:**

  **На HQ-RTR:**
  ```bash
  router ospf 1
    area 0 authentication ex
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

- **На EcoRouter HQ-RTR:**
  ```bash
  ip nat pool OVER 192.168.0.2-192.168.0.254
  ip nat source dynamic inside-to-outside pool OVER overload interface ISP
  ```
- **На EcoRouter BR-RTR:**
  ```bash
  ip nat pool nat3 192.168.1.2-192.168.1.31
  ip nat source dynamic inside-to-outside pool nat3 overload interface ISP
  ```

- **Настройка NAT на интерфейсах:**

  **HQ-RTR:**
  ```bash
  en
  conf t
  int ISP
    ip nat outside
  exit
  int te1.999
    ip nat inside
  exit
  int te1.100
    ip nat inside
  exit
  int te1.200
    ip nat inside
  ```

  **BR-RTR:**
  ```bash
  en
  conf t
  int ISP
    ip nat outside
  exit
  int SRV
    ip nat inside
  exit
  ```

- **Настройка шлюзов на серверах:**
  - **HQ-SRV:** Шлюз – `192.168.0.1/26`
  - **BR-SRV:** Шлюз – `192.168.1.1/27`

---

### 9. Настройка DHCP-сервера

- **Для офиса HQ (на HQ-RTR):**
  ```bash
  ip pool dhcpHQ 192.168.0.66-192.168.0.78
  en
  conf t
  dhcp-server 1
    pool dhcpHQ 1
      domain-name au-team.irpo
      mask 255.255.255.240
      gateway 192.168.0.65
      dns 192.168.0.2
  exit
  wr mem
  ```
- **Клиентом является HQ-CLI:**  
  На интерфейсе `te1.200` добавьте:
  ```bash
  dhcp-server 1
  ```

> **Примечания:**
> - Исключите из выдачи адрес маршрутизатора.
> - DNS-сервер для HQ-CLI – HQ-SRV.
> - DNS-суффикс – `au-team.irpo`.

---

  ![named1.png](https://github.com/taharakuro/DEMO25/blob/main/named1.png)
  ![named2.png](https://github.com/taharakuro/DEMO25/blob/main/named2.png)
