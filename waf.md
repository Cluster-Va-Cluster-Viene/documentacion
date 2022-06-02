# WAF

Vamos a configurar un nodo para los WAF, esta configuración solo la tendremos que replicar con tantos nodos como queramos y añadirlos al HAProxy en la sección **bk\_waf** para que la granja crezca según nuestras necesidades.

Instalamos apache

```bash
apt install apache2
```

Instalamos [ModSecurity](https://modsecurity.org/)

```bash
apt install libapache2-mod-security2
```

Reiniciamos apache para cargar el modulo

```bash
service apache2 restart
```

Comprobamos que el modulo se cargo correctamente

```bash
apachectl -M | grep security
```

Y deberíamos que obtener el siguiente resultado

```bash
security2_module (shared)
```

Ahora que ya tenemos tanto apache como modsecurity instalado y cargado vamos a proceder a configurarlo.

```bash
cp /etc/modsecurity/modsecurity.conf-recommended /etc/modsecurity/modsecurity.conf
```

Editamos la configuración recomendada para que en vez de detectar nos pare los ataques.

```bash
vim /etc/modsecurity/modsecurity.conf
```

Cambiado el valor `SecRuleEngine` de `DetectionOnly` a `On`

```vim
SecRuleEngine = On
```

ModSecurity tiene un conjunto de reglas predeterminado ubicado en el directorio /usr/share/modsecurity-crs . Sin embargo, nosotros vamos a cargar las reglas de OWASP

Hacemos copia de seguridad de las reglas

```bash
mv /usr/share/modsecurity-crs /usr/share/modsecurity-crs.bk
```

Descargamos las nuevas reglas

```bash
git clone https://github.com/SpiderLabs/owasp-modsecurity-crs.git /usr/share/modsecurity-crs
```

Copiamos la configuración por defecto de las reglas descargas

```bash
cp /usr/share/modsecurity-crs/crs-setup.conf.example /usr/share/modsecurity-crs/crs-setup.conf
```

Para que las reglas funcionen tenemos que editar el fichero

```bash
vim /etc/apache2/mods-enabled/security2.conf
```

Y añadir las dos siguientes lineas

```bash
IncludeOptional /usr/share/modsecurity-crs/*.conf
IncludeOptional /usr/share/modsecurity-crs/rules/*.conf
```

Estamos cargando un monto de reglas por defecto que seguramente no necesitaremos por lo que ahora tendríamos que adaptarlos a las aplicaciones que vayan a correr en nuestro cluster.

Una vez que que tenemos nuestro WAF vamos a configurarlo para que funcione como proxy reverso y escuche las peticiones de los HAPRoxys y se las devuelva.

Lo primero que hacemos es cambiar el puerto en el que escuchamos del 80 al 81

```bash
vim /etc/apache2/ports.conf
```

y dejamos el Listen de la siguiente manera

```vim
Listen 81
```

Activamos el módulos de proxy de apache

```bash
a2enmod proxy
```

Ahora vamos a configurar el VirtualHost por defecto para que haga de proxy reverso

```bash
vim /etc/apache2/sites-enabled/000-default.conf
```

Quedando de la siguiente manera, poniendo en IP\_WAF la ip interna de dicho servidor en el VRack y en XXX.XXX.XXX.XXX la ip flotante interna que configuramos en el keepalived

```bash
<VirtualHost IP_WAF:81>

        ServerAlias *
        AddDefaultCharset UTF-8

        ProxyPreserveHost On
        ProxyRequests off
        ProxyVia Off
        ProxyPass / http://XXX.XXX.XXX.XXX:82/
        ProxyPass /server-status !
        ProxyPassReverse / http://XXX.XXX.XXX.XXX:82/

</VirtualHost>
```

Ahora vamos a ocultar las versiones de nuestro servidor para dar la menor información posible a los atacantes, para ello editamos el siguiente fichero

```bash
vim /etc/apache2/conf-available/security.conf
```

Dejando los siguiente valores de esta manera

```bash
ServerTokens Prod

ServerSignature Off
```

Ahora tenemos que agregar una regla personalizada para que el HAProxy sepa que nuestros WAF están vivos, para ello creamos el fichero

```bash
vim /usr/share/modsecurity-crs/rules/aloha.conf
```

con el siguiente contenido

```vim
SecRule REQUEST_FILENAME "/waf_health_check" "phase:2,id:9999,deny,nolog,noauditlog,ctl:auditEngine=Off"
```

Ya solo nos queda reiniciar apache para cargar todos los cambios y nuestro primer WAF de la granja estaría listo

```bash
systemctl restart apache2
```

Para que la gente no se pueda conectar por la ip publica al puerto 81, dado que al no tener dominio luego no sabría donde ir.

```bash
iptables -A INPUT -i {interface} -p tcp --destination-port 81 -j DROP
```

Como las iptables si se reinicia el equipo se pierden y no queremos estar agregándolas manualmente vamos a instalar el paquete iptables-persistent

```bash
apt install iptables-persistent
```

Las configuraciones se guardan en estos dos ficheros

```
/etc/iptables/rules.v4

/etc/iptables/rules.v6
```

Si hacemos algún cambio a las iptables nos tenemos que acordar de guardarlas

```bash
iptables-save > /etc/iptables/rules.v4

iptables-save > /etc/iptables/rules.v6
```
