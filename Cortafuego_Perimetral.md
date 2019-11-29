## Implementación de un cortafuego perimetral

### Configuración en un solo paso
##### Editamos un fichero y añadimos todas las reglas necesarias para el escenario:

~~~
# Limpiamos las tablas
iptables -F
iptables -t nat -F
iptables -Z
iptables -t nat -Z
# Establecemos la política
iptables -P INPUT DROP
iptables -P OUTPUT DROP
iptables -P FORWARD DROP

# SNAT
echo 1 > /proc/sys/net/ipv4/ip_forward
iptables -t nat -A POSTROUTING -s 192.168.100.0/24 -o eth0 -j MASQUERADE

# DNAT
iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 80 -j DNAT --to 192.168.200.10

# Reglas INPUT/OUTPUT

iptables -A OUTPUT -p tcp -o eth1 -d 192.168.100.0/24 --dport 22 -j ACCEPT
iptables -A INPUT -p tcp -i eth1 -s 192.168.100.0/24 --sport 22 -j ACCEPT

iptables -A INPUT -i lo -p icmp -j ACCEPT
iptables -A OUTPUT -o lo -p icmp -j ACCEPT

iptables -A OUTPUT -o eth0 -p icmp -j ACCEPT
iptables -A INPUT -i eth0 -p icmp -j ACCEPT

iptables -A OUTPUT -o eth1 -p icmp -j ACCEPT
iptables -A INPUT -i eth1 -p icmp -j ACCEPT

# Reglas FORWARD

iptables -A FORWARD -o eth0 -i eth1 -s 192.168.100.0/24 -p icmp -j ACCEPT
iptables -A FORWARD -i eth0 -o eth1 -d 192.168.100.0/24 -p icmp -j ACCEPT

iptables -A FORWARD -i eth1 -o eth0 -s 192.168.100.0/24 -p udp --dport 53 -j ACCEPT
iptables -A FORWARD -o eth1 -i eth0 -d 192.168.100.0/24 -p udp --sport 53 -j ACCEPT

iptables -A FORWARD -i eth1 -o eth0 -s 192.168.100.0/24 -p tcp --dport 80 -j ACCEPT
iptables -A FORWARD -o eth1 -i eth0 -d 192.168.100.0/24 -p tcp --sport 80 -j ACCEPT
iptables -A FORWARD -i eth1 -o eth0 -s 192.168.100.0/24 -p tcp --dport 443 -j ACCEPT
iptables -A FORWARD -o eth1 -i eth0 -d 192.168.100.0/24 -p tcp --sport 443 -j ACCEPT

iptables -A FORWARD -i eth0 -o eth1 -d 192.168.100.0/24 -p tcp --dport 80 -j ACCEPT
iptables -A FORWARD -i eth1 -o eth0 -s 192.168.100.0/24 -p tcp --sport 80 -j ACCEPT
~~~


1. Permite realizar conexiones ssh desde los equipos de la LAN

~~~

~~~

2. Instala un servidor de correos en la máquina de la LAN. Permite el acceso desde el exterior y desde el cortafuego al servidor de correos. Para probarlo puedes ejecutar un telnet al puerto 25 tcp.

~~~

~~~

3. Permite poder hacer conexiones ssh desde exterior a la LAN

~~~

~~~

4. Modifica la regla anterior, para que al acceder desde el exterior por ssh tengamos que conectar al puerto 2222, aunque el servidor ssh este configurado para acceder por el puerto 22.

~~~

~~~

5. Permite hacer consultas DNS sólo al servidor 192.168.202.2. Comprueba que no puedes hacer un dig @1.1.1.1.

~~~

~~~

6. ¿Tendría resolución de nombres y navegación web el cortafuego? ¿Sería necesario? ¿Tendrían que estar esas de reglas de forma constante en el cortafuego?

~~~

~~~