# otus-linux-day32
## *Статическая и динамическая маршрутизация, OSPF*

# **Prerequisite**
- Host OS: Debian 12.0.0
- Guest OS: CentOS 7.8.2003
- VirtualBox: 6.1.40
- Vagrant: 2.3.2
- Ansible: 2.13.4

# **Содержание ДЗ**

- Развернуть 3 виртуальные машины
- Объединить их разными vlan
  - настроить OSPF между машинами на базе Quagga\FRR;
  - изобразить ассиметричный роутинг;
  - сделать один из линков "дорогим", но что бы при этом роутинг был симметричным

# **Выполнение**

Развёрнуты машины по схеме:
![Screenshot](https://github.com/jimidini77/otus-linux-day32/blob/main/net.png?raw=true)

На все машины подключен репозиторий FRR, установлены FRR и пакеты сетевых утилит:
```yml
    - name: Install FRR RPM repo
      yum:
        name: https://rpm.frrouting.org/repo/frr-stable-repo-1-0.el7.noarch.rpm
        state: present
        validate_certs: no

    - name: install base tools
      yum:
        name:
          - traceroute
          - tcpdump
          - net-tools
          - frr 
          - frr-pythontools
        state: present
        update_cache: true
        validate_certs: no
```

На всех машинах разрешен форвардинг траффика, и отключена Reverse Path фильтрация на интерфейсах между маршрутизаторами:
```yml
    - name: Enable forwarding across routers
      sysctl:
        name: net.ipv4.conf.all.forwarding
        value: '1'
        state: present

    - name: Enable Reverse Path Filtering across routers
      sysctl:
        name: "net.ipv4.conf.{{ item }}.rp_filter"
        value: '0'
        state: present
      with_items:
        - "eth1"
        - "eth2"
```
Выполнено включение OSPF и конфигурирование демона
```yml
    - name: Enable ospfd
      replace:
        path: /etc/frr/daemons
        regexp: '^ospfd=.*$'
        replace: 'ospfd=yes'
      notify: restart frr

    - name: Copy frr config to router
      template:
        src: frr.conf
        dest: /etc/frr/frr.conf
        owner: frr
        group: frr
        mode: 0640
      notify: restart frr
```
Конфиг каждой машины заполнен из шаблона:
```yml
frr version 8.1
frr defaults traditional
hostname {{ frr_hostname }}
log syslog informational
no ipv6 forwarding
service integrated-vtysh-config
!
interface eth1
description {{ eth1_description }}
ip address {{ eth1_ip }}
ip ospf mtu-ignore
{% if ansible_hostname == 'router1' %}
ip ospf cost 1000
{% elif ansible_hostname == 'router2' and symmetric_routing == true %}
ip ospf cost 1000
{% else %}
!ip ospf cost 45
{% endif %}
ip ospf hello-interval 10
ip ospf dead-interval 30
!
interface eth2
description {{ eth2_description }}
ip address {{ eth2_ip }}
ip ospf mtu-ignore
!ip ospf cost 45
ip ospf hello-interval 10
ip ospf dead-interval 30
!
interface eth3
description {{ eth3_description }}
ip address {{ eth3_ip }}
ip ospf mtu-ignore
!ip ospf cost 45
ip ospf hello-interval 10
ip ospf dead-interval 30
!
router ospf
router-id {{ router_id }}
network {{ ospf_net_1 }} area 0
network {{ ospf_net_2 }} area 0
network {{ ospf_net_3 }} area 0
neighbor {{ ospf_nb_1 }}
neighbor {{ ospf_nb_2 }}
!
log file /var/log/frr/frr.log
default-information originate always
```

После запуска FRR выполняется анонс сетей через OSPF:
```sh
[root@router1 vagrant]# ip r
default via 10.0.2.2 dev eth0 proto dhcp metric 100
10.0.2.0/24 dev eth0 proto kernel scope link src 10.0.2.15 metric 100
10.0.10.0/30 dev eth1 proto kernel scope link src 10.0.10.1 metric 101
10.0.11.0/30 proto 188 metric 20
        nexthop via 10.0.10.2 dev eth1 weight 1
        nexthop via 10.0.12.2 dev eth2 weight 1
10.0.12.0/30 dev eth2 proto kernel scope link src 10.0.12.1 metric 102
192.168.10.0/24 dev eth3 proto kernel scope link src 192.168.10.1 metric 103
192.168.20.0/24 via 10.0.10.2 dev eth1 proto 188 metric 20
192.168.30.0/24 via 10.0.12.2 dev eth2 proto 188 metric 20
192.168.56.0/24 dev eth4 proto kernel scope link src 192.168.56.101 metric 104
```
Все сети доступны например с router1:
```
[root@router1 vagrant]# ping -c 2 192.168.10.1
PING 192.168.10.1 (192.168.10.1) 56(84) bytes of data.
64 bytes from 192.168.10.1: icmp_seq=1 ttl=64 time=0.026 ms
64 bytes from 192.168.10.1: icmp_seq=2 ttl=64 time=0.079 ms

--- 192.168.10.1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1003ms
rtt min/avg/max/mdev = 0.026/0.052/0.079/0.027 ms
[root@router1 vagrant]# ping -c 2 192.168.20.1
PING 192.168.20.1 (192.168.20.1) 56(84) bytes of data.
64 bytes from 192.168.20.1: icmp_seq=1 ttl=64 time=0.312 ms
64 bytes from 192.168.20.1: icmp_seq=2 ttl=64 time=1.21 ms

--- 192.168.20.1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1000ms
rtt min/avg/max/mdev = 0.312/0.764/1.217/0.453 ms
[root@router1 vagrant]# ping -c 2 192.168.30.1
PING 192.168.30.1 (192.168.30.1) 56(84) bytes of data.
64 bytes from 192.168.30.1: icmp_seq=1 ttl=64 time=0.476 ms
64 bytes from 192.168.30.1: icmp_seq=2 ttl=64 time=0.821 ms

--- 192.168.30.1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 0.476/0.648/0.821/0.174 ms
[root@router1 vagrant]# ping -c 2 10.0.10.1
PING 10.0.10.1 (10.0.10.1) 56(84) bytes of data.
64 bytes from 10.0.10.1: icmp_seq=1 ttl=64 time=0.024 ms
64 bytes from 10.0.10.1: icmp_seq=2 ttl=64 time=0.043 ms

--- 10.0.10.1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 999ms
rtt min/avg/max/mdev = 0.024/0.033/0.043/0.011 ms
[root@router1 vagrant]# ping -c 2 10.0.11.1
PING 10.0.11.1 (10.0.11.1) 56(84) bytes of data.
64 bytes from 10.0.11.1: icmp_seq=1 ttl=63 time=0.676 ms
64 bytes from 10.0.11.1: icmp_seq=2 ttl=63 time=0.533 ms

--- 10.0.11.1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 0.533/0.604/0.676/0.075 ms
[root@router1 vagrant]# ping -c 2 10.0.12.1
PING 10.0.12.1 (10.0.12.1) 56(84) bytes of data.
64 bytes from 10.0.12.1: icmp_seq=1 ttl=64 time=0.021 ms
64 bytes from 10.0.12.1: icmp_seq=2 ttl=64 time=0.035 ms

--- 10.0.12.1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1000ms
rtt min/avg/max/mdev = 0.021/0.028/0.035/0.007 ms
```

Если погасить один из интерфейсов трассировка идёт по другому маршруту:
```
[root@router1 vagrant]# traceroute 192.168.30.1
traceroute to 192.168.30.1 (192.168.30.1), 30 hops max, 60 byte packets
 1  192.168.30.1 (192.168.30.1)  0.445 ms  0.464 ms  0.429 ms
[root@router1 vagrant]#
[root@router1 vagrant]#
[root@router1 vagrant]# ifconfig eth2 down
[root@router1 vagrant]#
[root@router1 vagrant]#
[root@router1 vagrant]# traceroute 192.168.30.1
traceroute to 192.168.30.1 (192.168.30.1), 30 hops max, 60 byte packets
 1  10.0.10.2 (10.0.10.2)  0.478 ms  0.405 ms  0.273 ms
 2  192.168.30.1 (192.168.30.1)  0.772 ms  0.832 ms  0.780 ms
``` 

Маршруты из интерфейса `vtysh`:
```
[root@router1 vagrant]# vtysh

Hello, this is FRRouting (version 8.3.1).
Copyright 1996-2005 Kunihiro Ishiguro, et al.

router1# sh ip route ospf
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, A - Babel, F - PBR, f - OpenFabric,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup
       t - trapped, o - offload failure

O   10.0.10.0/30 [110/100] is directly connected, eth1, weight 1, 00:21:34
O>* 10.0.11.0/30 [110/200] via 10.0.10.2, eth1, weight 1, 00:03:02
O>* 10.0.12.0/30 [110/300] via 10.0.10.2, eth1, weight 1, 00:02:33
O   192.168.10.0/24 [110/100] is directly connected, eth3, weight 1, 00:21:33
O>* 192.168.20.0/24 [110/200] via 10.0.10.2, eth1, weight 1, 00:20:34
O>* 192.168.30.0/24 [110/300] via 10.0.10.2, eth1, weight 1, 00:03:02
```

## Ассиметричная маршрутизация
Если на `router1` изменить цену интерфейса `eth1`, смотрящего на `router2`,то маршрут к сети 192.168.20.0/24 изменится и пойдёт через `router3` т.е. `eth2`:
```
[root@router1 vagrant]# vtysh

Hello, this is FRRouting (version 8.3.1).
Copyright 1996-2005 Kunihiro Ishiguro, et al.

router1# conf t
router1(config)# int eth1
router1(config-if)# ip ospf cost 1000
router1(config-if)# exit
router1(config)# exit
router1# sh ip route ospf
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, A - Babel, F - PBR, f - OpenFabric,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup
       t - trapped, o - offload failure

O   10.0.10.0/30 [110/300] via 10.0.12.2, eth2, weight 1, 00:00:19
O>* 10.0.11.0/30 [110/200] via 10.0.12.2, eth2, weight 1, 00:00:19
O   10.0.12.0/30 [110/100] is directly connected, eth2, weight 1, 00:02:55
O   192.168.10.0/24 [110/100] is directly connected, eth3, weight 1, 00:29:20
O>* 192.168.20.0/24 [110/300] via 10.0.12.2, eth2, weight 1, 00:00:19
O>* 192.168.30.0/24 [110/200] via 10.0.12.2, eth2, weight 1, 00:02:55
```
При этом машрут со стороны `router2` к `router1` останется с прежней стоимостью:
```
[root@router2 vagrant]# vtysh

Hello, this is FRRouting (version 8.3.1).
Copyright 1996-2005 Kunihiro Ishiguro, et al.

router2# sh ip route ospf
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, A - Babel, F - PBR, f - OpenFabric,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup
       t - trapped, o - offload failure

O   10.0.10.0/30 [110/100] is directly connected, eth1, weight 1, 00:59:16
O   10.0.11.0/30 [110/100] is directly connected, eth2, weight 1, 00:59:16
O>* 10.0.12.0/30 [110/200] via 10.0.10.1, eth1, weight 1, 00:58:36
  *                        via 10.0.11.1, eth2, weight 1, 00:58:36
O>* 192.168.10.0/24 [110/200] via 10.0.10.1, eth1, weight 1, 00:58:41
O   192.168.20.0/24 [110/100] is directly connected, eth3, weight 1, 00:59:16
O>* 192.168.30.0/24 [110/200] via 10.0.11.1, eth2, weight 1, 00:58:36
```

Трассировка между `router1` и `router2` принимает вид:
```
[root@router1 vagrant]# traceroute 192.168.20.1
traceroute to 192.168.20.1 (192.168.20.1), 30 hops max, 60 byte packets
 1  10.0.12.2 (10.0.12.2)  0.400 ms  0.243 ms  0.273 ms
 2  192.168.20.1 (192.168.20.1)  0.582 ms  0.654 ms  0.529 ms


 [root@router2 vagrant]# traceroute 192.168.10.1
traceroute to 192.168.10.1 (192.168.10.1), 30 hops max, 60 byte packets
 1  192.168.10.1 (192.168.10.1)  0.463 ms  0.389 ms  0.276 ms
```
То есть траффик идёт ассиметрично, а не используется тот же интерфейс, с котогого приходят пакеты от соседа. Для восстановления симметрии можно указать для `router2` цену интерфейса как у `router1`

Переменная `symmetric_routing` прописана в `ansible/defaults/main.yml` при значении `true` цена на `router1` и `router2` будет одинаковая после выполнении плейбука, при значении `false` маршрутизация будет ассиметричная.

# **Результаты**

Полученный в ходе работы `Vagrantfile` и плейбук Ansible помещены в публичный репозиторий:

- **GitHub** - https://github.com/jimidini77/otus-linux-day32
