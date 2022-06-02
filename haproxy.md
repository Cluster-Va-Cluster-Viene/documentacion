# HAProxy

Instalamos HaProxy en su versión 2 que se encuentra en los repositorios

```bash
apt install haproxy
```

Ahora vamos a preparar las diferentes partes de la configuración de HAProxy para lo siguiente:

* Todas las peticiones que tienen certificado se re direcciones automáticamente al puerto 443
* Se añadirá HSTS a todas las peticiones con certificado
* Se reenviara la petición a nuestra granja de WAF
* Si a petición es correcta se devolverá al HaProxy para que lo envié a los nodos

La configuración de HAProxy se encuentra en

```bash
vim /etc/haproxy/haproxy.cfg
```

Ahora vamos a ir viendo las diferentes secciones

Las dos primeras son la global donde configuramos el usuarios, la jaula... y la de por defecto para configurar el modo, los tiempos de conexión y los errores.

```bash
global
        log /dev/log    local0
        log /dev/log    local1 notice
        chroot /var/lib/haproxy
        stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
        stats timeout 30s
        user haproxy
        group haproxy
        daemon

defaults
        log     global
        mode    http
        option  httplog
        option  dontlognull
        timeout connect 5000
        timeout client  50000
        timeout server  50000
        errorfile 400 /etc/haproxy/errors/400.http
        errorfile 403 /etc/haproxy/errors/403.http
        errorfile 408 /etc/haproxy/errors/408.http
        errorfile 500 /etc/haproxy/errors/500.http
        errorfile 502 /etc/haproxy/errors/502.http
        errorfile 503 /etc/haproxy/errors/503.http
        errorfile 504 /etc/haproxy/errors/504.http
```

La siguiente sección se encarga de escuchar las peticiones que vienen del exterior y configurar el SLL, redireccionar a SSL y mandar al WAF o a los nodos directamente

```bash
 frontend clusterVaClusterViene
        bind *:80
        bind *:443 ssl crt-list /etc/haproxy/certs.txt no-tls-tickets ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES128-SHA256:ECDHE-RSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES256-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-GCM-SHA384:AES128-SHA256:AES128-SHA:AES256-SHA256:AES256-SHA:!MD5:!aNULL:!DH:!RC4:@STRENGTH no-tlsv10 no-tlsv11 alpn h2,http/1.1

        #Comprobamos si el dominio esta en la lista de ssl
        acl https_domain hdr(host) -f /etc/haproxy/domains.txt
        redirect scheme https code 301 if !{ ssl_fc } https_domain

        # Distinguimos si la conexión es segura o no
        acl secure dst_port eq 443

        # Marcamos todas las cookies como seguras si se usa ssl
        http-response replace-header Set-Cookie "^((?:(?!; [Ss]ecure\b).)*)\$" "\1; secure" if secure

        # Agregamos HSTS con un año de duracción
        http-response replace-header Strict-Transport-Security:\ max-age=31536000 if secure

        mode http

        # Comprobamos la carga del WAF por si esta saturado
        acl no_waf nbsrv(bk_waf) eq 0
        acl waf_max_capacity queue(bk_waf) ge 1

        # bypass WAF si no esta disponible
        use_backend bk_nodes if no_waf
        # bypass WAF si esta saturado
        use_backend bk_nodes if waf_max_capacity
        #Mandamos la petición al WAF
        default_backend bk_waf
```

Con la configuración actual de SSL conseguimos un A en SSLLabs.

![SSL Labs](imgs/ssl\_test.png)

Configuramos las peticiones a la granja de WAF, se manda una petición a waf\_health\_check a los diferentes nodos para ver si esta disponible esperando un 403 que luego configuraremos en el WAF y a continuación se envia al WAF.

El balance especifica la estrategia de equilibrio de carga. En este caso usamos Round-robin dado que vamos a provechar los pesos, dado que un WAF se encuentra en el mismo CPD que el HAProxy y el otro esta en otro CPD de esta manera reducimos los tiempos de latencia y priorizamos el que tenemos cerca.

```bash
 backend bk_waf
        balance roundrobin
        mode http
        log global
        option forwardfor header X-Client-IP
        option httpchk HEAD /waf_health_check HTTP/1.0
        # Specific WAF checking: a DENY means everything is OK
        http-check expect status 403
        timeout server 25s
        default-server inter 3s rise 2 fall 3
        server waf1 XXX.XXX.XXX.XXX:81 maxconn 100 weight 15 check
        server waf2 XXX.XXX.XXX.XXX:81 maxconn 100 weight 10 check
```

En esta sección lo que hacemos es recibir la petición de vuelta del WAF para mandarlos para los nodos, si solo usáramos una HAProxy podríamos decirle la ip interna que tiene que escuchar en el bind pero si usamos 2 y tenemos una ip flotante interna para que la granja de WAF siempre vuelva a la misma ip tenemos que dejarle el \*, esto supone un fallo de seguridad porque podrían entrar por el puerto 82 saltando el WAF, pero lo solucionaremos mediante Iptables mas adelante.

```bash
frontend ft_web
        bind *:82 name http
        mode http
        log global
        option httplog
        timeout client 25s
        maxconn 1000
        default_backend bk_nodes
```

La ultima sección lo que nos servirá es para enviar ya a los nodos volveremos a usar el tipo de conexión Round-robin con los pesos por el mismo motivo que antes dado que tenemos algunos en el mismo CPD y otros en otro.

```bash
backend bk_nodes
        mode http
        balance leastconn
        log global
        option forwardfor
        cookie SERVERID insert indirect nocache
        default-server inter 3s rise 2 fall 3
        option httpchk HEAD /
        server server1 XXX.XXX.XXX.XXX:80 maxconn 100 weight 15 cookie server1 check
        server server2 XXX.XXX.XXX.XXX:80 maxconn 100 weight 15 cookie server2 check
        server server3 XXX.XXX.XXX.XXX:80 maxconn 100 weight 10 cookie server3 check
```

Con esto tendríamos una configuración minima para poder usar nuestro tipo de infraestructura, podríamos añadir mas cosas al HAProxy como mitigación de DDoS, evitar que los ficheros estaticos como las imagenes tengan que pasar por el waf para liberarlos de carga...

### Keepalived

Podemos instalar keepalived desde código o desde repositorio, la versión de repositiro es un poco antigua para nos vale para nuestro cometido.

```bash
apt install keepalived
```

Las configraciones entre los dos nodos son muy parecidas solo teniendo que cambiar la prioridad y el estado.

#### Nodo MASTER

Creamos la configuración en el siguiente fichero

```
vim /etc/keepalived/keepalived.conf
```

```vim
global_defs {
    enable_script_security
    script_user root
}

vrrp_script chk_haproxy {
        script "/usr/bin/killall -0 haproxy" # Comprobamos que el Haproxy esta vivo
        interval 2 #Cada 2 segundo
        weight 2 # Agregamos 2 puntos de peso si esta OK
}

#Ip flotante dentro del VRack
vrrp_instance VI_2 {
        interface ens4 # Interfaz que monitorizamos
        state MASTER # MASTER en haproxy1, BACKUP en haproxy2
        virtual_router_id 24
        priority 101 # 101 en haproxy1, 100 en haproxy2

        authentication {
                auth_type PASS
                auth_pass PASSWORD
        }

        virtual_ipaddress {
                IP_FLOTANTE dev ens4 # Dirrección virtual
        }
        track_script {
                chk_haproxy
        }
}
```

#### Nodo Backup

Creamos la configuración en el siguiente fichero

```bash
vim /etc/keepalived/keepalived.conf
```

```vim
global_defs {
    enable_script_security
    script_user root
}

vrrp_script chk_haproxy {
        script "/usr/bin/killall -0 haproxy" # Comprobamos que el Haproxy esta vivo
        interval 2 #Cada 2 segundo
        weight 2 # Agregamos 2 puntos de peso si esta OK
}

#Ip flotante dentro del VRack
vrrp_instance VI_2 {
        interface ens4 # Interfaz que monitorizamos
        state MASTER # BACKUP en haproxy1, BACKUP en haproxy2
        virtual_router_id 24
        priority 100 # 101 en haproxy1, 100 en haproxy2

        authentication {
                auth_type PASS
                auth_pass PASSWORD
        }

        virtual_ipaddress {
                IP_FLOTANTE dev ens4 # Dirrección virtual
        }
        track_script {
                chk_haproxy
        }
}
```

### certbot

Vamos a utilizar certbot para que todos los dominios de nuestros cluster tengan por lo menos un certificado ssl gratuito, en este caso tenemos los dominios en OVH (Si el dominio esta en otro proveedor tendriamos que mirar como sacar el certificado para dicho proveedor) por lo primero que tenemos que hacer es generar una api token y los permisos necesarios, para no tener que reconfigurar el HAProxy para que funcione dado que este camino es mas fácil y cómodo.

Para crear los tokens vamos al siguiente enlace:

[API OVH](https://api.ovh.com/createToken/)

Podemos dar permisos globales para funcionar:

* GET /domain/zone/\*
* PUT /domain/zone/\*
* POST /domain/zone/\*
* DELETE /domain/zone/\*

pero seria bastante inseguro por lo que podemos dar solo los permisos necesarios

* GET /domain/zone/
* GET /domain/zone/{domain.ext}/status
* GET /domain/zone/{domain.ext}/record
* GET /domain/zone/{domain.ext}/record/\*
* POST /domain/zone/{domain.ext}/record
* POST /domain/zone/{domain.ext}/refresh
* DELETE /domain/zone/{domain.ext}/record/\*

Reemplace {domain.ext} con su nombre de dominio. Tenga en cuenta que este es siempre el nombre de dominio raíz sin un subdominio.

Después de la validación, deberá crearse un archivo de configuración para que Certbot pueda acceder a los identificadores de API. Puedes guardar este archivo donde quieras y nombrarlo como quieras. Por mi parte `/root/certs/.ovhapi` con el siguiente contenido.

```vim
dns_ovh_endpoint = ovh-eu
dns_ovh_application_key = xxx
dns_ovh_application_secret = xxx
dns_ovh_consumer_key = xxx
```

Obviamente, los reemplaza `xxx` con la información obtenida durante la creación del token. Finalmente, asegúrese de configurar permisos para este archivo con 600, de lo contrario Certbot generara advertencias. `chmod 600 /root/certs/.ovhapi`

Instalamos cerbot en el HA1

```bash
#Instalamos python3-pip
apt install python3-pip

#Instalamos certbot y el complemento de ovh
pip3 install certbot
pip3 install certbot-dns-ovh
```

#### Generando certificados

Una vez que tenemos todo listo, podemos generar nuestros certificados, solo los dominios administrados en su cuenta OVH pueden funcionar.

```bash
# generación del certificado para rotasim.com y * .rotasim.com
# estos son dos certificados separados pero se agruparán en un solo archivo
certbot certonly --dns-ovh --dns-ovh-credentials ~/.ovhapi -d rotasim.com -d *.rotasim.com

# alternativamente, si ejecuta las líneas de comando en dos etapas (por lo tanto, sin -d * .rotasim.com para la anterior)
# obtenemos un archivo por certificado
certbot certonly --dns-ovh --dns-ovh-credentials ~/.ovhapi -d *.rotasim.com

# si quieres hacer un script de la generación
# debemos especificar que no sea interactivo
certbot certonly --dns-ovh --dns-ovh-credentials ~/.ovhapi --non-interactive --agree-tos --email nombre@email.com -d rotasim.com -d *.rotaism.com
```

Vamos a crear el siguiente script en la carpeta `root/certs` para poder tener los certificados de todos nuestros dominios y sincronizados con los dos HaProxys

```bash
 vim /root/certs/certs.sh
```

con el siguiente contenido, el cual comprobara la lista de dominios que tenemos en el ficheros domains\_ssl.txt y si no esta creado el certificado lo solicitara, acordarse de cambiar el email.

```bash
#!/bin/bash
find /etc/letsencrypt/live/* -type d -printf "%f\n" > domains_live.txt
certs=`grep -v -F -x -f domains_live.txt domains_ssl.txt`
for cert in $certs; do
        certbot certonly --dns-ovh --dns-ovh-credentials /root/certs/.ovhapi --dns-ovh-propagation-seconds 60 --non-interactive --agree-tos --email nombre@dominio.com -d $cert --deploy-hook /root/certs/deployhook.sh
done
```

Tendremos un segundo script que se lanzara cuando se acabe de solicitar el dominio para para preprar los certificados para HA y crear la lista de certificados que necesitamos

```bash
vim /root/certs/deployhook.sh
```

```bash
#!/bin/bash
IP=10.0.46.48

cat $RENEWED_LINEAGE/fullchain.pem $RENEWED_LINEAGE/privkey.pem>$RENEWED_LINEAGE/haproxy.pem
if ! grep -q $RENEWED_LINEAGE "/etc/haproxy/certs.txt"
then

        for domain in $RENEWED_DOMAINS; do
           echo "$RENEWED_LINEAGE/haproxy.pem $domain" >>/etc/haproxy/certs.txt
           echo "$domain" >>/etc/haproxy/domains.txt
        done
fi
rsync -rpa --delete /etc/letsencrypt/ $IP:/etc/letsencrypt
scp /etc/haproxy/certs.txt $IP:/etc/haproxy/
scp /etc/haproxy/domains.txt $IP:/etc/haproxy/
ssh $IP "systemctl restart haproxy"
systemctl restart haproxy
```

Esto mismo se podría gestionar desde una base de datos y crear una interfaz web para no tener que entrar por consola si fuera necesario.
