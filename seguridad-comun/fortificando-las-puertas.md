# Fortificando las puertas

## Bloquenado TOR

Si queremos bloquear los nodos TOR para evitar que se conecten desde dicha red que en muchos casos se suele usar para atacar dado el grado de anonimato que proporciona, podemos usar el siguiente script:

```bash
#!/bin/bash
# Block Tor Exit nodes
IPTABLES_TARGET="DROP"
IPTABLES_CHAINNAME="TOR"
if ! iptables -L TOR -n >/dev/null 2>&1 ; then
  iptables -N TOR >/dev/null 2>&1
  iptables -I INPUT 1 -p tcp -j TOR
fi
cd /tmp/
echo -e "\n\tGetting TOR node list from dan.me.uk\n"
wget -q -O - "https://www.dan.me.uk/torlist/" -U SXTorBlocker/1.0 > /tmp/full.tor
wget -q -O - "https://check.torproject.org/cgi-bin/TorBulkExitList.py?ip=1.1.1.1" -U SXTorBlocker/1.0 --no-check-certificate >> /tmp/full.tor
sed -i 's|^#.*$||g' /tmp/full.tor
CMD=$(cat /tmp/full.tor | sort | uniq | grep -o '[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}')
iptables -F TOR
for IP in $CMD; do
  let COUNT=COUNT+1
  iptables -A TOR -s $IP -j DROP
done
  iptables -A TOR -j RETURN
```

luego solo tenemos que poner un crontab para que se actualice periódicamente:

```bash
0 */2 * * * /opt/scripts/./tor-iptables.sh
```

Para que la gente no se pueda conectar por la ip publica al puerto 82 y saltarse el WAF lo cerramos por iptables.

```bash
iptables -A INPUT -i {interface} -p tcp --destination-port 82 -j DROP
```

Como las iptables si se reinicia el equipo se pierden y no queremos estar agregandolas manualmente vamos a instalar el paquete iptables-persistent

```bash
apt install iptables-persistent
```

Las configuraciones se guardan en estos dos ficheros

```bash
/etc/iptables/rules.v4

/etc/iptables/rules.v6
```

Si hacemos algun cambio a las iptables nos tenemos que acordar de guardarlas

```bash
iptables-save > /etc/iptables/rules.v4

iptables-save > /etc/iptables/rules.v6
```

## PortSentry

Una cosa que queremos es bloquear los posibles escaneos de red y de paso dar información no veraz para confundir a nuestros atacantes, para ello disponemos del paquete PortSentry.

```bash
apt install portsentry
```

Lo primero que tenemos que hacer es configurar nuestra ip o rango de trabajo en el fichero que sirve para ignorar los escaneos no sea que nos quedemos fuera.

```bash
vim  /etc/portsentry/portsentry.ignore.static
```

```vim
# Example:

192.168.2.0/24
192.168.0.0/16
192.168.2.1/32
```

Al arrancar el servicio, el contenido del archivo se añadirá al archivo _/etc/portsentry/portsentry.ignore_.

Utilizamos portsentry en modo avanzado para los protocolos TCP y UDP. Para ello, debe modificar

```bash
vim /etc/default/portsentry
```

```vim
TCP_MODE="atcp"
UDP_MODE="audp"
```

Por defecto PortSentry esta en modo detección solo pero a nosotros nos interesa que detecte y bloque los escaneos

```bash
vim /etc/portsentry/portsentry.conf 
```

```vim
##################
# Ignore Options #
##################
# 0 = Do not block UDP/TCP scans.
# 1 = Block UDP/TCP scans.
# 2 = Run external command only (KILL_RUN_CMD)

BLOCK_UDP="1"
BLOCK_TCP="1"
```

Optamos por un bloqueo de personas maliciosas a través de iptables. Por lo tanto comentaremos en todas las líneas del archivo de configuración que comienzan con KILL\_ROUTE excepto la de iptables

```vim
KILL_ROUTE="/sbin/iptables -I INPUT -s $TARGET$ -j DROP"
```

Por defecto PortSentry mete también los escaneos dentro del /etc/hosts.deny, gracias a la siguiente linea si no queremos que esto pase podemos comentarla.

```vim
KILL_HOSTS_DENY="ALL: $TARGET$ : DENY"
```

## Fail2ban

Otra cosa que no queremos es que la gente se ponga a dar martillazos contra nuestras puertas para intentar entrar para ello disponemos de la herramienta Fail2Ban.

```bash
apt install fail2ban
```

La instalación predeterminada de Fail2ban viene con dos archivos de configuración, /etc/fail2ban/jail.conf y /etc/fail2ban/jail.d/defaults-debian.conf. No se recomienda modificar estos archivos, ya que pueden sobrescribirse cuando se actualiza el paquete.

Fail2ban lee los archivos de configuración en el siguiente orden. Cada archivo .local anula la configuración del archivo .conf:

* `/etc/fail2ban/jail.conf`
* `/etc/fail2ban/jail.d/*.conf`
* `/etc/fail2ban/jail.local`
* `/etc/fail2ban/jail.d/*.local`

Copiamos la configuración por defecto que luego modificaremos, podríamos hacernos nosotros la configuración totalmente a mano.

```bash
cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
```

```bash
vim /etc/fail2ban/jail.local
```

Podemos cambiar la cantidad de intentos, el tiempo de baneo o el tiempo que hay entre los máximos intentos.

```vim
# "bantime" is the number of seconds that a host is banned.
bantime  = 10m

# A host is banned if it has generated "maxretry" during the last "findtime"
# seconds.
findtime  = 10m

# "maxretry" is the number of failures before a host get banned.
maxretry = 5
```

Si queremos estar atentos a las ips bloqueadas en todo momento podemos configurar un email para que nos notifique el sistema

```vim
# Destination email address used solely for the interpolations in
# jail.{conf,local,d/*} configuration files.
destemail = root@localhost

# Sender email address used solely for some actions
sender = root@<fq-hostname>

# E-mail action. Since 0.8.1 Fail2Ban uses sendmail MTA for the
# mailing. Change mta configuration parameter to mail if you want to
# revert to conventional 'mail'.
mta = sendmail

```

Por defecto fail2ban tiene un montón de jaulas que podemos activar, por defecto en debian viene activada la del ssh por lo que no es necesario volver a activar, pero si queremos activar cualquier otra solo tenemos que añadir en su bloque de configuración enable=true

```vim
[apache-badbots]
# Ban hosts which agent identifies spammer robots crawling the web
# for email addresses. The mail outputs are buffered.
enabled = true
port     = http,https
logpath  = %(apache_access_log)s
bantime  = 48h
maxretry = 1
```

ahora solo nos queda arrancar el servicio

```bash
systemctl enable fail2ban
systemctl start fail2ban
```
