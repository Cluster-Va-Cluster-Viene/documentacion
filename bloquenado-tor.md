# Bloquenado TOR

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

luego solo tenemos que poner un crontab para que se actualice periodicamente:

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

```
/etc/iptables/rules.v4

/etc/iptables/rules.v6
```

Si hacemos algun cambio a las iptables nos tenemos que acordar de guardarlas

```bash
iptables-save > /etc/iptables/rules.v4

iptables-save > /etc/iptables/rules.v6
```
