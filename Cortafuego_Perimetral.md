## Implementación de un cortafuego perimetral

### Configuración en un solo paso
###### Editamos un fichero y añadimos todas las reglas necesarias para el escenario:

~~~
sudo su

# Limpiamos las tablas
iptables -F
iptables -t nat -F
iptables -Z
iptables -t nat -Z

# Conexión ssh
iptables -A INPUT -s 172.22.0.0/16 -p tcp -m tcp --dport 22 -j ACCEPT
iptables -A OUTPUT -d 172.22.0.0/16 -p tcp -m tcp --sport 22 -j ACCEPT

iptables -A INPUT -s 172.23.0.0/16 -p tcp -m tcp --dport 22 -j ACCEPT
iptables -A OUTPUT -d 172.23.0.0/16 -p tcp -m tcp --sport 22 -j ACCEPT

# Establecemos la política
iptables -P INPUT DROP
iptables -P OUTPUT DROP
iptables -P FORWARD DROP

# SNAT
echo 1 > /proc/sys/net/ipv4/ip_forward
iptables -t nat -A POSTROUTING -s 192.168.100.0/24 -o eth0 -j MASQUERADE

# DNAT
iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 80 -j DNAT --to 192.168.100.10

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

exit
clear
~~~

> Para listar las reglas de IPTABLES:
~~~
sudo iptables -L -nv --line-numbers
~~~
> Para eliminar las reglas de IPTABLES:
~~~
sudo iptables -D INPUT <number>
sudo iptables -D OUTPUT <number>
~~~
> Para listar las reglas de NAT
~~~
sudo iptables -t nat -L -nv --line-numbers
~~~

### 1. Permite realizar conexiones ssh desde los equipos de la LAN

##### Reglas

~~~
sudo iptables -t nat -I POSTROUTING -s 192.168.100.0/24 -o eth0 -p tcp --dport 22 -j MASQUERADE

sudo iptables -A FORWARD -i eth1 -o eth0 -p tcp --dport 22 -j ACCEPT
sudo iptables -A FORWARD -i eth0 -o eth1 -p tcp --sport 22 -j ACCEPT
~~~

##### Comprobación

~~~
debian@lan:~$ ssh moralg@172.23.0.54
moralg@172.23.0.54's password:
Linux padano 4.19.0-6-amd64 #1 SMP Debian 4.19.67-2+deb10u2 (2019-11-11) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
You have new mail.
Last login: Wed Dec  4 14:23:43 2019 from 172.22.201.77
~~~

### 2. Instala un servidor de correos en la máquina de la LAN. Permite el acceso desde el exterior y desde el cortafuego al servidor de correos. Para probarlo puedes ejecutar un telnet al puerto 25 tcp

##### Reglas

~~~
sudo iptables -t nat -I PREROUTING -i eth0 -p tcp --dport 25 -j DNAT --to 192.168.100.10
~~~
###### Para onectarse desde mi equipo:
~~~
sudo iptables -A FORWARD -i eth0 -o eth1 -d 192.168.100.10/24 -p tcp --dport 25 -j ACCEPT
sudo iptables -A FORWARD -i eth1 -o eth0 -p tcp --sport 25 -j ACCEPT
~~~
###### Para conectarse desde el router:
~~~
sudo iptables -A OUTPUT -o eth1 -d 192.168.100.10/24 -p tcp --dport 25 -j ACCEPT
sudo iptables -A INPUT -i eth1 -p tcp --sport 25 -j ACCEPT
~~~

##### Comprobación

###### Desde mi equipo:
~~~
moralg@padano:~$ telnet 172.22.201.64 25
Trying 172.22.201.64...
Connected to 172.22.201.64.
Escape character is '^]'.
220 lan.novalocal ESMTP Postfix (Debian/GNU)
~~~

###### Desde el router:
~~~
debian@router-fw:~$ telnet 192.168.100.10 25
Trying 192.168.100.10...
Connected to 192.168.100.10.
Escape character is '^]'.
220 lan.novalocal ESMTP Postfix (Debian/GNU)
~~~

### 3. Permite poder hacer conexiones ssh desde exterior a la LAN

##### Reglas

~~~
sudo iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 22 -j DNAT --to 192.168.100.10

sudo iptables -A FORWARD -i eth0 -o eth1 -p tcp --dport 22 -j ACCEPT
sudo iptables -A FORWARD -i eth1 -o eth0 -p tcp --sport 22 -j ACCEPT
~~~

##### Comprobación

###### Hacemos ssh a la 172.22.201.64 y se nos rediregirá a la LAN ya que tenemos hecho un DNAT al puerto 22 desde el router
~~~
moralg@padano:~$ ssh debian@172.22.201.64
Linux lan 4.19.0-6-cloud-amd64 #1 SMP Debian 4.19.67-2+deb10u1 (2019-09-20) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Thu Dec  5 17:23:53 2019 from 172.23.0.54
debian@lan:~$
~~~

### 4. Modifica la regla anterior, para que al acceder desde el exterior por ssh tengamos que conectar al puerto 2222, aunque el servidor ssh este configurado para acceder por el puerto 22

##### Reglas

~~~
sudo iptables -t nat -I PREROUTING -i eth0 -p tcp --dport 2222 -j DNAT --to 192.168.100.10:22

sudo iptables -I FORWARD -i eth0 -o eth1 -p tcp --dport 22 -j ACCEPT
sudo iptables -I FORWARD -i eth1 -o eth0 -p tcp --sport 22 -j ACCEPT
~~~

##### Comprobación

###### Igual que en el ejercicio 3 pero indicandole el puerto 2222
~~~
moralg@padano:~$ ssh -p 2222 debian@172.22.201.64
Linux lan 4.19.0-6-cloud-amd64 #1 SMP Debian 4.19.67-2+deb10u1 (2019-09-20) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Thu Dec  5 17:34:34 2019 from 172.23.0.54
debian@lan:~$
~~~

### 5. Permite hacer consultas DNS sólo al servidor 192.168.202.2. Comprueba que no puedes hacer un dig @1.1.1.1

##### Reglas

~~~
sudo iptables -t nat -A POSTROUTING -s 192.168.100.0/24 -o eth0 -p udp --dport 53 -j MASQUERADE

sudo iptables -I FORWARD -i eth0 -o eth1 -p udp --sport 53 -j ACCEPT
sudo iptables -I FORWARD -i eth1 -o eth0 -d 192.168.202.2 -p udp --dport 53 -j ACCEPT
~~~

##### Comprobación

~~~
debian@lan:~$ dig @1.1.1.1

   ; <<>> DiG 9.11.5-P4-5.1-Debian <<>> @1.1.1.1
   ; (1 server found)
   ;; global options: +cmd
   ;; connection timed out; no servers could be reached
~~~

### 6. ¿Tendría resolución de nombres y navegación web el cortafuego?

###### No tendría resolución de nombres ni de navegación.

### ¿Sería necesario?

###### No sería necesario porque pondríamos en riesgo la seguridad.

### ¿Tendrían que estar esas de reglas de forma constante en el cortafuego?

###### No, añadiriamos las reglas individualmente dependiendo de la demanda.