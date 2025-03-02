# Создание роутера для шлюза

1. Создадим в GNS3 роутер Mikrotik RB450G, используя данные с сервера, по последней доступной версии прошивки, с именем "Gateway".
2. Соединим порт `ether1` роутера с облаком "ISP".
3. Проведём настройку роутера:
	1) установим пароль администратора (по умолчанию он пустой);
	2) включим интерфейс `ether1`:
    ```
	interface enable ether1
	```
	3) проверим статус DHCP-клиента:
	  ```
	  ip dhcp-client print
	  ```
	  На интерфейсе `ether1` должен работать DHCP-клиент. Если это не так, то включим его:  
	  ```
	  ip dhcp-client add interface=ether1 use-peer-dns=yes add-default-route=yes
	  ```
	4) включим интерфейс `ether2`:
	  ```
	  interface enable ether2
	  ```
	5) добавим маршрут через шлюз провайдера (шлюз системы гипервизора):
	  ```
	  ip route add gateway=192.168.1.1
	  ```
	6) добавим статический IP-адрес 10.0.0.254/24 на интерфейсе `ether2`:
	  ```
	  ip address add address=10.0.0.254/24 interface=ether2
	  ```
	7) включим NAT из одной сети в другую:
	  ```
	  ip firewall nat add chain=srcnat out-interface=ether1 action=masquerade
	  ```
	8) проверим таблицу маршрутизации:
	  ```
	  ip route print
	  ```
	9) проверим статус NAT:
	  ```
	  ip firewall net print
	  ```