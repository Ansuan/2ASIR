# IPTables
## Visualizar reglas
```sh
iptables -L -nv
```
## Reglas por defecto
```sh
iptables -P [INPUT,OUTPUT,FORWARD] [ACCEPT,DROP]
```
## Añadir reglas
```sh
iptables -A [INPUT,OUTPUT,FORWARD] [-i,-o] interfaz -p protocolo [-s, -d] red [--sport, --dport] puerto -j [ACCEPT,DROP]

iptables -A FORWARD -i enp0s3 -o enp0s8 -s 192.168.200.2/32 -d 192.168.100.2/32 -p tcp --dport 3306 -j ACCEPT
```