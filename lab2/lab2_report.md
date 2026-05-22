University: [ITMO University](https://itmo.ru/ru/)  
Faculty: [FICT](https://fict.itmo.ru)  
Course: [Introduction in routing](https://github.com/itmo-ict-faculty/network-programming)  
Year: 2025/2026  
Group: K3323  
Author: Ivanova Ekaterina Andreevna  
Lab: Lab2  
Date of creation:  
Date of finish:  

## Лабораторная работа №2 "Развертывание дополнительного CHR, первый сценарий Ansible"

### Описание
В данной лабораторной работе вы на практике ознакомитесь с системой управления конфигурацией Ansible, использующаяся 
для автоматизации настройки и развертывания программного обеспечения

### Цель работы
С помощью Ansible настроить несколько сетевых устройств и собрать информацию о них. Правильно собрать файл Inventory

### Ход работы

В качестве основы используется конфигурация из первой лабораторной работы: VPS на базе Kamatera с Ubuntu и установленным
Ansible, локально на Mac M1 в гипервизоре UTM развернут первый CHR (RouterOS), между ними поднят WireGuard-туннель в 
подсети `10.10.10.0/24`. Задача второй лабораторной работы — добавить второй CHR, подключить его к тому же VPN-серверу 
и через Ansible сразу настроить оба роутера, включая OSPF-смежность между ними

### Установка второго CHR

Второй CHR был получен клонированием первой виртуальной машины в UTM. Для этого нужно выключить исходную VM, нажать на 
ней правой кнопкой и выбрать **Clone**. После клонирования у новой VM был перегенерирован MAC-адрес в настройках 
сетевого интерфейса — иначе обе VM в Shared Network получают одинаковый MAC и одна из них остается без IP

После запуска CHR2 на нем были удалены остатки WireGuard-конфигурации, унаследованные от клонирования, и переименован 
identity:

```
/system identity set name=CHR2
/interface wireguard peers remove [find]
/interface wireguard remove [find]
/ip address remove [find interface=wg-to-vps]
```

### Настройка второго WireGuard-клиента

На сервере была сгенерирована новая пара ключей для CHR2:

```bash
cd /etc/wireguard
sudo wg genkey | sudo tee chr2_private.key | sudo wg pubkey | sudo tee chr2_public.key
```

В конфигурационный файл `/etc/wireguard/wg0.conf` был добавлен второй peer, не затрагивая существующий блок для CHR1:

```ini
[Peer]
# CHR2
PublicKey = <chr2_public.key>
AllowedIPs = 10.10.10.3/32
```

После перезапуска службы оба peer'а отображаются в `wg show` с актуальным handshake

На CHR2 был настроен WireGuard-клиент по аналогии с первым CHR, но с туннельным IP `10.10.10.3`:

```
/interface wireguard add name=wg-to-vps listen-port=51820 private-key="<chr2_private.key>"
/ip address add address=10.10.10.3/24 interface=wg-to-vps
/interface wireguard peers add interface=wg-to-vps public-key="<server_public.key>" \
    endpoint-address=<VPS_IP> endpoint-port=51820 \
    allowed-address=10.10.10.0/24 persistent-keepalive=25s
```

Параметр `allowed-address=10.10.10.0/24` (а не `/32`) важен — он разрешает CHR2 отправлять в туннель не только пакеты 
до сервера, но и до CHR1, что позволяет OSPF-смежности устанавливаться поверх WireGuard

### Подготовка CHR2 к управлению через Ansible

На CHR2 была создана та же группа прав и пользователь, что и на CHR1, ограниченный туннельной сетью:

```
/user group add name=ansible-grp policy=ssh,read,write,test,api,rest-api,!local,!telnet,!winbox,!password,!web,!sniff,!sensitive,!romon
/user add name=ansible password=ansible_pass group=ansible-grp address=10.10.10.0/24
/ip service disable telnet,ftp,www,www-ssl,winbox
/ip service set ssh address=10.10.10.0/24
/ip service set api address=10.10.10.0/24
/ip service enable api
```

### Inventory-файл для двух CHR

Файл `inventory.ini` описывает оба роутера в одной группе `mikrotik`. У каждого хоста заданы переменные `ansible_host` 
(туннельный IP) и `router_id` (уникальный OSPF Router ID). Параметры подключения вынесены в групповую секцию `vars`, 
чтобы не дублировать их в каждом хосте:

```ini
[mikrotik]
chr1 ansible_host=10.10.10.2 router_id=1.1.1.1
chr2 ansible_host=10.10.10.3 router_id=2.2.2.2

[mikrotik:vars]
ansible_user=ansible
ansible_password=ansible_pass
ansible_connection=ansible.netcommon.network_cli
ansible_network_os=community.routeros.routeros
ansible_python_interpreter=/usr/bin/python3
```

Конфиг `ansible.cfg`:

```ini
[defaults]
inventory = inventory.ini
host_key_checking = False
stdout_callback = yaml
```

Проверка связи с обоими CHR одновременно:

```bash
ansible mikrotik -m community.routeros.command -a "commands='/system identity print'"
```

В ответ возвращаются имена обоих роутеров — Ansible видит оба CHR через VPN-туннель:

```
chr1 | CHANGED => {
    "changed": true,
    "stdout": ["name: CHR1"]
}
chr2 | CHANGED => {
    "changed": true,
    "stdout": ["name: CHR2"]
}
```

Также можно посмотреть структуру inventory:

```bash
ansible-inventory --list
```

### Плейбук `lab2.yml`

Плейбук одной командой настраивает оба CHR: задает имя, NTP-клиента, OSPF с уникальным Router ID и собирает конфиги в 
локальные файлы. Использует переменные `inventory_hostname` и `router_id` из inventory, что позволяет применять одну и 
ту же логику ко всем устройствам без копипасты:

```yaml
---
- name: Lab2 setup
  hosts: mikrotik
  gather_facts: no

  tasks:

    - name: Create local configs directory
      delegate_to: localhost
      file:
        path: "./configs"
        state: directory

    - name: Allow OSPF in firewall and set WireGuard MTU
      community.routeros.command:
        commands:
          - /ip firewall filter add chain=input protocol=ospf action=accept place-before=0 comment="Allow OSPF"
          - /interface wireguard set [find] mtu=1300
      ignore_errors: yes

    - name: Clean previous OSPF configuration
      community.routeros.command:
        commands:
          - /routing ospf static-neighbor remove [find]
          - /routing ospf interface-template remove [find]
          - /routing ospf area remove [find]
          - /routing ospf instance remove [find]
      ignore_errors: yes

    - name: Set identity and configure NTP client
      community.routeros.command:
        commands:
          - /system identity set name={{ inventory_hostname | upper }}
          - /system ntp client set enabled=yes servers=pool.ntp.org,time.google.com
      ignore_errors: yes

    - name: Configure OSPF
      community.routeros.command:
        commands:
          - /routing ospf instance add name=default router-id={{ router_id }} disabled=no
          - /routing ospf area add name=backbone instance=default area-id=0.0.0.0
          - /routing ospf interface-template add area=backbone networks=10.10.10.0/24 type=ptp
          - "/routing ospf static-neighbor add address={{ '10.10.10.3' if inventory_hostname == 'chr1' else '10.10.10.2' }} instance=default"
      ignore_errors: yes

    - name: Wait for OSPF adjacency
      pause:
        seconds: 30

    - name: Collect config and OSPF neighbors
      community.routeros.command:
        commands:
          - /export compact
          - /routing ospf neighbor print detail
      register: chr_output

    - name: Save output files locally
      delegate_to: localhost
      copy:
        content: "{{ item.content }}"
        dest: "./configs/{{ inventory_hostname }}_{{ item.suffix }}"
      loop:
        - { content: "{{ chr_output.stdout[0] }}", suffix: "export.rsc" }
        - { content: "NEIGHBORS:\n{{ chr_output.stdout[1] }}", suffix: "ospf.txt" }
```

Пояснения по ключевым моментам плейбука:

- **Allow OSPF in firewall + WireGuard MTU = 1300** — OSPF поверх WireGuard в RouterOS 7.x работает нестабильно с MTU 
  по умолчанию (1420). Снижение до 1300 решает проблему с прохождением OSPF DBD-пакетов через туннель
- **Clean previous OSPF configuration** — гарантирует чистое состояние при повторном запуске плейбука, иначе команды 
  `add` ругаются на «already exists»
- **type=ptp в interface-template** — point-to-point режим обходит проблему с multicast-рассылкой hello-пакетов 
  через WireGuard, который не реплицирует multicast между peer'ами
- **static-neighbor с условным выражением** — для каждого CHR указывается явный адрес соседа (CHR1 → 10.10.10.3, 
  CHR2 → 10.10.10.2), что заставляет OSPF отправлять hello unicast'ом вместо multicast. Это финальный фикс, 
  необходимый для работы OSPF поверх WireGuard

Запуск:

```bash
ansible-playbook lab2.yml
```

Плейбук выполняется на обоих хостах параллельно, без ошибок:

```
PLAY [Lab2 setup] *****************************************************

TASK [Create local configs directory] *********************************
ok: [chr1 -> localhost]
changed: [chr2 -> localhost]

TASK [Allow OSPF in firewall and set WireGuard MTU] *******************
changed: [chr1]
changed: [chr2]

TASK [Clean previous OSPF configuration] ******************************
changed: [chr1]
changed: [chr2]

TASK [Set identity and configure NTP client] **************************
changed: [chr1]
changed: [chr2]

TASK [Configure OSPF] *************************************************
changed: [chr1]
changed: [chr2]

PLAY RECAP ************************************************************
chr1 : ok=9  changed=6  unreachable=0  failed=0  skipped=0
chr2 : ok=9  changed=6  unreachable=0  failed=0  skipped=0
```

### Проверка результата плейбука на роутерах

После выполнения плейбука изменения действительно произошли на обоих CHR. Проверка NTP-клиента:

```
[admin@CHR1] > /system ntp client print
        enabled: yes
           mode: unicast
        servers: pool.ntp.org, time.google.com
            vrf: main
     freq-drift: -2.197 PPM
         status: synchronized
   synced-server: pool.ntp.org
  synced-stratum: 2
   system-offset: 5.587 ms
```

Аналогичный вывод на CHR2. Проверка identity:

```
[admin@CHR1] > /system identity print
name: CHR1

[admin@CHR2] > /system identity print
name: CHR2
```

Проверка пользователя для Ansible:

```
[admin@CHR1] > /user print
Flags: X - disabled
 #   NAME      GROUP         ADDRESS         LAST-LOGGED-IN
 0   admin     full
 1   ansible   ansible-grp   10.10.10.0/24   2026-05-22 11:38:33
```

### Проверка связности

Проверка туннелей с сервера:

```bash
sudo wg show
ping -c 4 10.10.10.2
ping -c 4 10.10.10.3
```

Оба peer'а имеют свежий handshake и ненулевой transfer. Пинги до обоих CHR через WireGuard проходят без потерь

Проверка с CHR1:

```
/ping count=4 10.10.10.1
/ping count=4 10.10.10.3
/ping count=4 2.2.2.2
```

С CHR2:

```
/ping count=4 10.10.10.1
/ping count=4 10.10.10.2
/ping count=4 1.1.1.1
```

Пинги до сервера, до второго CHR через туннель, и до loopback-адреса другого CHR через OSPF-маршрут проходят без потерь

### Сбор данных и конфигов

Плейбук сохраняет результаты в локальную папку `./configs/` на сервере:

```bash
$ ls -la configs/
total 24
drwxr-xr-x 2 root root 4096 May 22 11:38 .
drwxr-xr-x 3 root root 4096 May 22 11:37 ..
-rw-r--r-- 1 root root 1415 May 22 11:38 chr1_export.rsc
-rw-r--r-- 1 root root  280 May 22 11:38 chr1_ospf.txt
-rw-r--r-- 1 root root 1473 May 22 11:38 chr2_export.rsc
-rw-r--r-- 1 root root  280 May 22 11:38 chr2_ospf.txt
```

Содержимое `chr1_ospf.txt` показывает OSPF-интерфейсы и установленное соседство:

```
NEIGHBORS:
Flags: D - dynamic
 0 D address=10.10.10.2%wg-to-vps area=backbone state=ptp network-type=ptp
     cost=1 use-bfd=no retransmit-interval=5s transmit-delay=1s
     hello-interval=10s dead-interval=40s

 1 D address=1.1.1.1%loopback area=backbone state=ptp network-type=ptp
     cost=1 use-bfd=no retransmit-interval=5s transmit-delay=1s
     hello-interval=10s dead-interval=40s

Flags: V - virtual; D - dynamic
 0 D instance=default area=backbone address=10.10.10.3
   router-id=2.2.2.2 state="Full" state-changes=1 adjacency=2m14s
   timeout=40s
```

Видно соседство с CHR2 (router-id=2.2.2.2) в состоянии **Full** — OSPF-смежность установлена успешно

Файл `chr1_export.rsc` содержит полный конфиг устройства, включая WireGuard, OSPF, identity, NTP, пользователей и 
сервисы. Это и есть требуемые «2 файла с конфигурациями устройств» из условия лабораторной работы

### Итоговая схема

```
                          VPS (Ubuntu)
                       Ansible + WireGuard
                            10.10.10.1
                          /            \
                         /              \
                  WireGuard tunnel    WireGuard tunnel
                       /                  \
              10.10.10.2              10.10.10.3
                CHR1                      CHR2
                R-ID: 1.1.1.1            R-ID: 2.2.2.2
                  \___                  ___/
                      \___OSPF area 0__/
```

### Вывод

В ходе выполнения лабораторной работы развернут второй CHR клонированием первого в UTM и подключен к тому же 
WireGuard-серверу как второй peer. Собран файл Inventory для Ansible с двумя хостами в одной группе и общими 
переменными подключения. Написан плейбук, который одновременно настраивает оба роутера: задает identity, NTP-клиента 
и OSPF с уникальным Router ID для каждого устройства. Между двумя CHR через VPN-туннель установлено OSPF-соседство в 
area 0. Конфигурации обоих устройств и данные OSPF-топологии собраны в локальные файлы