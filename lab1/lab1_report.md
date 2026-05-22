University: [ITMO University](https://itmo.ru/ru/)  
Faculty: [FICT](https://fict.itmo.ru)  
Course: [Introduction in routing](https://github.com/itmo-ict-faculty/network-programming)  
Year: 2025/2026  
Group: K3323  
Author: Ivanova Ekaterina Andreevna  
Lab: Lab1  
Date of creation:  
Date of finish: 

## Лабораторная работ №1 "Установка CHR и Ansible, настройка VPN"

### Описание
Данная работа предусматривает обучение развертыванию виртуальных машин (VM) и системы контроля конфигураций Ansible, а 
также организации собственных VPN серверов

### Цель работы
Целью данной работы является развертывание виртуальной машины на базе платформы Microsoft Azure с установленной 
системой контроля конфигураций Ansible и установка CHR в VirtualBox

### Ход работы

Разворачиваем VM на базе Kamatera с Ubuntu 22.04

Обновляем пакеты и устанавливаем python3 и Ansible 

```bash
sudo apt update & sudo apt upgrade
sudo apt install python3-pip
ls -la /usr/bin/python3.10
sudo pip3 install ansible
ansible --version
```

Скачиваем образ CHR с официального сайта MikroTik и создаем виртуальную машину в UTM в режиме Virtualize с
архитектурой ARM64, 512 МБ RAM и сетью Shared Network. Скачанный образ подключаем как диск в разделе Drives.
Дополнительно добавляем Serial-устройство в режиме Built-in Terminal, так как CHR выводит консоль не на графический
адаптер, а в последовательный порт

После запуска заходим в систему под `admin` с пустым паролем, принимаем лицензию и задаем новый пароль. Проверяем 
работоспособность:

```
/system identity set name=CHR
/ip address print
/ping 8.8.8.8
/system resource print
```

CHR получил IP по DHCP от UTM (`192.168.64.2/24`), интернет доступен, версия RouterOS — 7.21.4.

### Настройка VPN сервера WireGuard на Ubuntu

В качестве VPN-протокола был выбран WireGuard — современный протокол с минимальным конфигом и нативной поддержкой как в 
Ubuntu, так и в RouterOS 

Устанавливаем WireGuard на сервер автоматизации:

```bash
sudo apt update && sudo apt install -y wireguard
```

Включаем форвардинг пакетов:

```bash
sudo sed -i 's/#net.ipv4.ip_forward=1/net.ipv4.ip_forward=1/' /etc/sysctl.conf
sudo sysctl -p
```

Генерируем пары ключей для сервера и клиента (CHR):

```bash
cd /etc/wireguard
umask 077
wg genkey | tee server_private.key | wg pubkey > server_public.key
wg genkey | tee chr_private.key | wg pubkey > chr_public.key
```

Создаем конфигурационный файл сервера `/etc/wireguard/wg0.conf`:

```ini
[Interface]
Address = 10.10.10.1/24
ListenPort = 51820
PrivateKey = <server_private.key>

[Peer]
# CHR
PublicKey = <chr_public.key>
AllowedIPs = 10.10.10.2/32
```

Запускаем WireGuard и добавляем в автозагрузку, открываем порт UDP/51820:

```bash
sudo systemctl enable --now wg-quick@wg0
sudo systemctl status wg-quick@wg0
sudo ufw allow 51820/udp
```

### Настройка VPN клиента WireGuard на CHR

На стороне CHR создаем WireGuard-интерфейс с собственным приватным ключом:

```
/interface wireguard
add name=wg-to-vps listen-port=51820 private-key="<chr_private.key>"
```

Назначаем IP-адрес внутри туннельной сети:

```
/ip address add address=10.10.10.2/24 interface=wg-to-vps
```

Добавляем peer (VPN-сервер) с его публичным ключом и адресом endpoint:

```
/interface wireguard peers
add interface=wg-to-vps public-key="<server_public.key>" \
    endpoint-address=<VPS_IP> endpoint-port=51820 \
    allowed-address=10.10.10.0/24 persistent-keepalive=25s
```

Параметр `persistent-keepalive=25s` нужен, чтобы NAT на стороне клиента не закрывал UDP-сессию при отсутствии трафика

### Проверка VPN-туннеля

После применения конфигов проверяем состояние туннеля. На сервере:

```bash
sudo wg show
```

В выводе видно peer'а с актуальным handshake

На CHR:

```
/interface wireguard peers print detail
```

Поля `rx` и `tx` показывают двусторонний обмен трафиком — значит, handshake состоялся и туннель установлен

Проверяем IP-связность в обе стороны:

```
# на CHR
/ping 10.10.10.1

# на VPS
ping 10.10.10.2
```

Пакеты проходят без потерь, а это значит, что VPN-туннель работает

### Настройка CHR для управления через Ansible

Создаем на CHR отдельную группу прав и пользователя для Ansible, ограниченного туннельной сетью:

```
/user group add name=ansible-grp policy=ssh,read,write,test,api,rest-api,!local,!telnet,!winbox,!password,!web,!sniff,!sensitive,!romon
/user add name=ansible password=ansible_pass group=ansible-grp address=10.10.10.0/24
```

Ограничиваем сервисы SSH и API только туннельной сетью:

```
/ip service disable telnet,ftp,www,www-ssl,winbox
/ip service set ssh address=10.10.10.0/24
/ip service set api address=10.10.10.0/24
/ip service enable api
```

### Установка и настройка Ansible

На сервере автоматизации создаем виртуальное окружение Python и устанавливаем Ansible с коллекцией для RouterOS:

```bash
python3 -m venv ~/ansible-env
source ~/ansible-env/bin/activate
pip install ansible ansible-pylibssh
ansible-galaxy collection install community.routeros
```

Библиотека `ansible-pylibssh` нужна для подключения к сетевому оборудованию по SSH через `network_cli`

Создаем inventory-файл `inventory.yml`:

```yaml
all:
  children:
    mikrotik:
      hosts:
        chr:
          ansible_host: 10.10.10.2
          ansible_user: ansible
          ansible_password: ansible_pass
          ansible_network_os: community.routeros.routeros
          ansible_connection: network_cli
          ansible_python_interpreter: /usr/bin/python3
```

И конфиг `ansible.cfg`:

```ini
[defaults]
inventory = inventory.yml
host_key_checking = False
stdout_callback = yaml
```

Проверяем связь с CHR через VPN-туннель:

```bash
ansible mikrotik -m community.routeros.command -a "commands='/system identity print'"
```

Ansible успешно выполняет команду на CHR, в ответ возвращается имя роутера

### Проверка через плейбук

Для демонстрации работы автоматизации создаем плейбук `playbook.yml`, который читает информацию о роутере:

```yaml
- name: Configure CHR via VPN
  hosts: mikrotik
  gather_facts: no
  tasks:

    - name: Get CHR info
      community.routeros.command:
        commands:
          - /system identity print
          - /system resource print
      register: info
```

Запускаем плейбук:

```bash
ansible-playbook playbook.yml
```

В выводе задача выполнена без ошибок

В выводе `note` видно строку `Managed by Ansible via WireGuard`, NTP-клиент включен с сервером `pool.ntp.org`

### Итоговая схема

```
┌─────────────────────┐      WireGuard tunnel       ┌─────────────────────┐
│  VPS (Ubuntu)       │ ◀──────────────────────────▶│  CHR                │
│  Ansible            │   10.10.10.1 ↔ 10.10.10.2   │                     │
└─────────────────────┘                             └─────────────────────┘
```

### Вывод

В ходе выполнения лабораторной работы развернута виртуальная машина в облачном сервисе с установленной системой 
контроля конфигураций Ansible и локально установлен CHR (RouterOS). Между сервером автоматизации и 
локальным CHR поднят VPN-туннель на базе WireGuard, обеспечена IP-связность в обе стороны. Ansible успешно управляет 
CHR через туннель: подключение к API/SSH RouterOS работает