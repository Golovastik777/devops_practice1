# Отчет по лабораторной работе №1

## Задание

Вам необходимо сделать трехуровневую сеть связи классического предприятия, изображенную на рисунке 1, в ContainerLab. Необходимо создать все устройства, указанные на схеме ниже, и соединения между ними, правила работы с СontainerLab можно изучить по ссылке.

* Помимо этого вам необходимо настроить IP адреса на интерфейсах и 2 VLAN-a для `PC1` и `PC2`, номера VLAN-ов вы вольны выбрать самостоятельно.
* Также вам необходимо создать 2 DHCP сервера на центральном роутере в ранее созданных VLAN-ах для раздачи IP адресов в них. `PC1` и `PC2` должны получить по 1 IP адресу из своих подсетей.
* Настроить имена устройств, сменить логины и пароли.

![Схема из задания](https://itmo-ict-faculty.github.io/introduction-in-routing/education/labs2023_2024/lab1/3tiernetwork.png)

## Описание работы


### Топология 
В файле `lab.yaml` описана топология сети. Она включает маршрутизатор `R1`, три коммутатора (`SW1`, `SW2`, `SW3`), а также два конечных устройства (`PC1` и `PC2`). Каждый узел имеет свой файл конфигурации (в папке `configs`, который загружается при старте контейнера.

```
name: enterprise-network

mgmt:
  network: static
  ipv4-subnet: 172.20.20.0/24

topology:
  nodes:
    R1:
      kind: linux
      image: alpine:latest
      mgmt-ipv4: 172.20.20.11
      startup-config: /configs/r1_linux.sh
      binds:
        - ./configs:/configs

    SW1:
      kind: linux
      image: alpine:latest
      mgmt-ipv4: 172.20.20.12
      startup-config: /configs/sw1_linux.sh
      binds:
        - ./configs:/configs

    SW2:
      kind: linux
      image: alpine:latest
      mgmt-ipv4: 172.20.20.13
      startup-config: /configs/sw2_linux.sh
      binds:
        - ./configs:/configs

    SW3:
      kind: linux
      image: alpine:latest
      mgmt-ipv4: 172.20.20.14
      startup-config: /configs/sw3_linux.sh
      binds:
        - ./configs:/configs

    PC1:
      kind: linux
      image: alpine:latest
      mgmt-ipv4: 172.20.20.15
      startup-config: /configs/pc1_linux.sh
      binds:
        - ./configs:/configs

    PC2:
      kind: linux
      image: alpine:latest
      mgmt-ipv4: 172.20.20.16
      startup-config: /configs/pc2_linux.sh
      binds:
        - ./configs:/configs

  links:
    - endpoints: ["R1:eth1", "SW1:eth1"]
    - endpoints: ["SW1:eth2", "SW2:eth1"]
    - endpoints: ["SW1:eth3", "SW3:eth1"]
    - endpoints: ["SW2:eth2", "PC1:eth1"]
    - endpoints: ["SW3:eth2", "PC2:eth1"]

```


### Настройка маршрутизатора R1

На маршрутизаторе `R1` созданы VLAN 10 и VLAN 20, каждый для своего сегмента (PC1 и PC2). Для обоих VLAN настроены DHCP-сервера, которые выдают IP-адреса из заданных диапазонов. Также добавлен новый пользователь и изменено имя устройства.


Команды для настройки `R1`:
```
#!/bin/sh
sleep 10
ip link set dev eth1 up
ip link add link eth1 name eth1.10 type vlan id 10
ip link add link eth1 name eth1.20 type vlan id 20
ip link set dev eth1.10 up
ip link set dev eth1.20 up
ip addr add 10.10.10.1/24 dev eth1.10
ip addr add 10.10.20.1/24 dev eth1.20
echo 1 > /proc/sys/net/ipv4/ip_forward
echo "R1 configured with VLANs 10 and 20"
tail -f /dev/null

```


### Настройка коммутатора SW1

На `SW1` добавлены VLAN-интерфейсы для разных портов и созданы мосты для VLAN 10 и VLAN 20, обеспечивающие передачу трафика между соответствующими интерфейсами. Для каждого VLAN настроен клиент DHCP.



Команды для настройки `SW1`:
```
#!/bin/sh
sleep 10
ip link set dev eth1 up
ip link set dev eth2 up
ip link set dev eth3 up
ip link add name br0 type bridge
ip link set dev br0 up
ip link set dev eth1 master br0
ip link set dev eth2 master br0
ip link set dev eth3 master br0
echo "SW1 bridge configured"
tail -f /dev/null

```

### Настройка SW2 и SW3

Конфигурация аналогична.





### Настройка ПК

`PC1` и `PC2` настраиваются для работы в своих VLAN, создаются подинтерфейсы и задаются IP-адреса.

Пример для `PC1`:


Пример настройки `PC1`:

```
#!/bin/sh
sleep 10
ip link set dev eth1 up
ip link add link eth1 name eth1.10 type vlan id 10
ip link set dev eth1.10 up
ip addr add 10.10.10.100/24 dev eth1.10
ip route add default via 10.10.10.1
echo "PC1 configured in VLAN 10"
tail -f /dev/null

```

Пример настройки `PC2`:
```
#!/bin/sh
sleep 10
ip link set dev eth1 up
ip link add link eth1 name eth1.20 type vlan id 20
ip link set dev eth1.20 up
ip addr add 10.10.20.100/24 dev eth1.20
ip route add default via 10.10.20.1
echo "PC2 configured in VLAN 20 with IP 10.10.20.100/24"
tail -f /dev/null
```

### Пример работы

После настройки всех устройств, были выполнены тесты на подключение и маршрутизацию. В частности, `PC1` и `PC2` успешно получили IP-адреса через DHCP, что подтверждается скриншотом ниже:

<img width="788" height="485" alt="image" src="https://github.com/user-attachments/assets/4ecf6490-ab13-4ea5-ac40-d5e7ba234409" />


Также был выполнен тест с помощью команды `ping`, подтверждающий успешное взаимодействие между устройствами в сети:

<img width="788" height="304" alt="image" src="https://github.com/user-attachments/assets/feddf023-a120-49e6-8551-2280e58757f0" />

## Заключение
В результате выполнения лабораторной работы была успешно настроена трехуровневая сеть с VLAN-ами и DHCP-серверами. Все устройства функционируют корректно, и конечные устройства получают IP-адреса согласно настройкам DHCP.
