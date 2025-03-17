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

### 8. Динамическая трансляция адресов (NAT) [В отчете заносить не нужно будет, поэтому пункт сделан ранее]

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

---

  ![named1.png](https://github.com/dizzamer/DEMO2025/blob/main/dns.png)
  ![named2.png](https://github.com/dizzamer/DEMO2025/blob/main/dns2.png)
