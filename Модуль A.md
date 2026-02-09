# Сетевые настройки
## Оглавление
- [IP forward](#ip-forward)
- [NAT](#nat)
- [DNAT](#dnat)
- [Сохранение IPTABLES](#сохранение-iptables)
- [Динамическая маршрутизация](#динамическая-маршрутизация)
- [Туннели GRE+IPSec](#туннели-greipsec)
- [Отказоустойчивое решение (два провайдера в одной внутренней подсети) - VRRP](#отказоустойчивое-решение-два-провайдера-в-одной-внутренней-подсети---vrrp)
- [DHCP-сервер](#dhcp-сервер)
- [IPTABLES](#iptables)
## IP forward
Нужно его "включить":
```
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
sysctl -p
```
## NAT
Динамический NAT (***эта зараза может подвести и отвалиться***):
```
iptables -t nat -A POSTROUTING -s *NET-INT* -o *ens3* -j MASQUERADE
```
Статический NAT:
```
iptables -t nat -A POSTROUTING -s *NET-INT* -o *ens3* -j SNAT --to-source *IP-EXT*
```
## DNAT
```
iptables -t nat -A PREROUTING -d 89.101.0.5 -p tcp --dport 8443 -j DNAT --to-destination 192.168.1.10:8443
-d - на какой IP прокидываем, --dport - на какой порт прокидываем, --to destination - откуда прокидываем
```
## Сохранение IPTABLES
Ставим пакет:
```
apt install -y iptables-persistent
```
Затем после каждого изменения в iptables выполняем:
```
iptables-save > /etc/iptables/rules.v4
```
## Динамическая маршрутизация
Ставим пакет **frr**:
```
apt update
apt install -y frr
```
Для настроек запускаем:
```
vtysh
```
После каждой настройки в **conf t** делаем сохранения:
```
do write
```
### OSPF:
Включаем OSPF в конфиге:
```
nano /etc/frr/daemons
	ospfd=1
systemctl restart frr
```
Определение зон, настройка маршрутов и Router-ID:
```
conf t
	router ospf
		ospf router-id 10.5.5.1
		network 10.6.6.0/30 area 0
		network 10.7.7.0/30 area 0
		neighbor 10.5.5.2 - указываем соседа
	exit
```
Аутентификация:
```
interface eth1
	ip ospf authentication message-digest
	ip ospf message-digest-key 1 md5 PASSWORD
exit
```
### RIP:
Включаем RIP в конфиге:
```
nano /etc/frr/daemons
	ripd=1
systemctl restart frr
```
Настраиваем маршруты:
```
conf t
	router rip
		version 2
		network 10.10.10.0/24
		network 172.16.0.0/30
	exit
```
Пассивные интерфейсы:
```
router rip
	passive-interface default - по дефолту все пассивные
	no passive-interface eth0 - делаем активным нужный
exit
```
Включаем только RIPv2: ??? (вроде только version 2 надо указать)
```
router rip
	distribute-list prefix RIP-V2 in
	distribute-list prefix RIP-V2 out
exit
ip prefix-list RIP-V2 seq 10 permit 0.0.0.0/0 le 32
```
Аутентификация:
```
key chain RIP-KEY
	key 1
		key-string GreenSkillsRIP
	exit
exit
interface eth0
	ip rip authentication mode md5
	ip rip authentication key-chain RIP-KEY
exit
do write
```
## Туннели GRE+IPSec
### GRE:
На "левом" роутере:
```
nano /etc/network/interfaces
	auto gre-1
	iface gre-1 inet static
	address *IP-LOCAL-TUN*
	netmask 255.255.255.0
	pre-up ip tunnel add *gre-1* mode gre remote *IP-REMOTE-EXT* local *IP-LOCAL-EXT* ttl 255
	post-up ip route add *NET-REMOTE-INT* via *IP-REMOTE-TUN*
	pre-down ip tunnel del *gre-1* mode gre remote *NET-REMOTE-EXT* local *IP-LOCAL-EXT* ttl 255
```
На "правом" роутере делаем зеркальные адреса и маршруты.
### IPSec:
Ставим нужный пакет:
```
apt install strongswan
```
На "левом" роутере:
```
nano /etc/ipsec.conf
	config setup
		charondebug="all"
		uniqueids=no
		strictcrlpolicy=no
	conn gre-msk
		keyexchange=ikev2 - явное использование IKEv2
		authby=psk - пассфраза
		left=*IP-LOCAL-EXT*
		right=*IP-REMOTE-EXT*
		leftprotoport=gre
		rightprotoport=gre
		type=tunnel
		esp=aes128-sha1 - как шифруются реальные данные: 1 - шифрование, 2 - проверка целостности
		ike=aes128-sha256-modp3072 - как стороны договариваются о защите: 1 - шифрование данных, 2 - аутентификация и проверка целостности, 3 - для сессионных ключей (modp - диффи-хелман)
		auto=start
```
На "правом" роутере делаем зеркальные адреса.
Пишем пассфразу:
```
nano /etc/ipsec.secrets
	IP-LOCAL-EXT IP-REMOTE-EXT : PSK "Pass"
```
На "правом" опять зеркально.
Перезапускаем, проверяем статус:
```
systemctl restart strongswan-starter - перезапуск
ipsec start - запуск
ipsec status - проверяем подключения
```
## Отказоустойчивое решение (два провайдера в одной внутренней подсети) - VRRP
Качаем пакет:
```
apt install keepalived
```
На **основном** роутере:
```
nano /etc/keepalived/keepalived.conf
	vrrp_instance VI_1 {
		state BACKUP
		interface ens4
		virtual_router_id 51
		priority 100
		advert_int 1
		virtual_ipaddress {
			10.15.10.1/24
		}
	}
	vrrp_instance VI_2 {
		state MASTER
		interface ens4
		virtual_router_id 52
		priority 150
		advert_int 1
		virtual_ipaddress {
			10.15.10.1/24
		}
	}
systemctl restart keepalived
```
На **запасном** роутере:
```
nano /etc/keepalived/keepalived.conf
	vrrp_instance VI_1 {
		state MASTER
		interface ens4
		virtual_router_id 51
		priority 150
		advert_int 1
		virtual_ipaddress {
			10.15.10.1/24
		}
	}
	vrrp_instance VI_2 {
		state BACKUP
		interface ens4
		virtual_router_id 52
		priority 100
		advert_int 1
		virtual_ipaddress {
			10.15.10.1/24
		}
	}
systemctl restart keepalived
```
Получается зеркально настраиваем.
## DHCP-сервер
Ставим пакет:
```
sudo apt install isc-dhcp-server
```
Редактируем конфиг:
```
nano /etc/default/isc-dhcp-server:
	DHCPDv4_CONF=/etc/dhcp/dhcpd.conf
	INTERFACESv4="ens4" - внутренний интерфейс, который будет раздавать адреса
```
Настройки DHCP:
```
nano /etc/dhcp/dhcpd.conf:
	subnet 192.168.1.0 netmask 255.255.255.0 {
	    range 192.168.1.50 192.168.1.100; - диапазон адресов для сети
	    option routers 192.168.1.1; - адрес DHCP
	    option domain-name-servers 192.168.1.2, 77.88.8.1;
	    option domain-name "company";
	    default-lease-time 600;
	    max-lease-time 7200;
	}
sudo systemctl restart isc-dhcp-server
```
## IPTABLES
**DROP всегда в конце, иначе все откинет, и ACCEPT не пройдут.**

Если политика DROP, то в самом начале надо разрешить ответы:
```
iptables -A FORWARD -m state --state ESTABLISHED,RELATED -j ACCEPT
```
Поменять политику:
```
iptables -P FORWARD DROP|ACCEPT
```
Запретить весь трафик из сети A в сеть B (вместо сети можно указывать только IP, если надо только для одного):
```
iptables -A FORWARD -s *NET-A* -d *NET-B* -j DROP
```
Запретить только конкретный порт:
```
iptables -A FORWARD -s *NET-A* -d *NET-B* -p tcp --dport 22 -j DROP
```
Запретить несколько конкретных портов:
```
iptables -A FORWARD -s *NET-A* -d *NET-B* -p tcp -m multiport --dports 80,443 -j DROP
```
Запретить весь трафик, кроме одного конкретного порта:
```
iptables -A FORWARD -s *NET-A* -d *NET-B* -p tcp --dport 22 -j ACCEPT
iptables -A FORWARD -s *NET-A* -d *NET-B* -j DROP
```
Запретить весь трафик, кроме нескольких конкретных портов:
```
iptables -A FORWARD -s *NET-A* -d *NET-B* -p tcp -m multiport --dports 80,443 -j ACCEPT
iptables -A FORWARD -s *NET-A* -d *NET-B* -j DROP
```
Запретить только ICMP трафик:
```
iptables -A FORWARD -s *NET-A* -d *NET-B* -p icmp -j DROP
```
Разрешить только ICMP:
```
iptables -A FORWARD -s *NET-A* -d *NET-B* -p icmp -j ACCEPT
iptables -A FORWARD -s *NET-A* -d *NET-B* -j DROP
```
Логирование:
```
-j LOG --log-prefix "DROP A->B: "
```
