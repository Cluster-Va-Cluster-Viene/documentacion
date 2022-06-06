# Monitorización

Para la monitorización de nuestros sistemas vamos a usar el tandem [prometheus](https://prometheus.io/) y [grafana](https://grafana.com/)

### Prometheus

Nos servira para recuperar datos de unos sensores que pondremos en nuestros sistemas y crear alertas por si algo sucede, para empezar vamos a descargamos prometheus

```bash
wget https://github.com/prometheus/prometheus/releases/download/v2.36.0/prometheus-2.36.0.linux-amd64.tar.gzr.gz
```

Descomprimimos

```bash
tar xfz prometheus-*.tar.gz
```

Creamos el usuarios para prometehus

```bash
useradd --no-create-home --shell /usr/sbin/nologin prometheus
```

Creamos las carpetas

```bash
mkdir /etc/prometheus
mkdir /var/lib/prometheus
```

Les asignamos el usuario creado anteriormente

```bash
chown prometheus:prometheus /etc/prometheus
chown prometheus:prometheus /var/lib/prometheus
```

Entramos en la carpeta de prometheus y copiamos los binarios

```bash
cd prometheus-2.36.0.linux-amd64
cp ./prometheus /usr/local/bin/
cp ./promtool /usr/local/bin/
```

asignamos usuario prometheus

```bash
chown prometheus:prometheus /usr/local/bin/prometheus
chown prometheus:prometheus /usr/local/bin/promtool
```

Copiamos la consola y las librerías

```bash
cp -r ./consoles /etc/prometheus
cp -r ./console_libraries /etc/prometheus
```

Damos permisos

```bash
chown -R prometheus:prometheus /etc/prometheus/consoles
chown -R prometheus:prometheus /etc/prometheus/console_libraries
```

Ahora que lo tenemos instalado lo vamos a configurar, para ello creamos el siguiente fichero

```bash
vim /etc/prometheus/prometheus.yml
```

```bash
global:
  scrape_interval:     15s
  evaluation_interval: 15s

rule_files:
  # - "first.rules"
  # - "second.rules"

scrape_configs:
  - job_name: 'prometheus'
    scrape_interval: 5s
    static_configs:
      - targets: ['localhost:9090']
```

Con esto definimos cada cuanto tiene que scrapear los datos de los diferentes nodos que configuraremos más adelante

Asignamos el usuario a la configuración

```bash
chown prometheus:prometheus /etc/prometheus/prometheus.yml
```

Primer arranque de prometehus para ver que esta todo en orden, donde le decimos que configuración tiene que cargar, donde va a guardar los datos (podríamos montarlo en un glusterfs para tener una replica fuera) y la configuración de la versión web.

```bash
sudo -u prometheus /usr/local/bin/prometheus --config.file /etc/prometheus/prometheus.yml --storage.tsdb.path /var/lib/prometheus/ --web.console.templates=/etc/prometheus/consoles --web.console.libraries=/etc/prometheus/console_libraries
```

Para tenerlo en el arranque de manera automática lo vamos a configurar en systemd, para ello creamos el siguiente fichero.

```bash
vim /etc/systemd/system/prometheus.service
```

```vim
[Unit]
  Description=Prometheus Monitoring
  Wants=network-online.target
  After=network-online.target

[Service]
  User=prometheus
  Group=prometheus
  Restart=on-failure
  RestartSec=5s
  Type=simple
  ExecStart=/usr/local/bin/prometheus \
  --config.file /etc/prometheus/prometheus.yml \
  --storage.tsdb.path /var/lib/prometheus/ \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries
  ExecReload=/bin/kill -HUP $MAINPID

[Install]
  WantedBy=multi-user.target
```

Recargamos el demonio de systemd y ya lo tendremos disponible.

```bash
systemctl daemon-reload
```

```bash
systemctl start prometheus
```

#### Nginx + SSL + Autenticación

Una opción para ayudar a proteger nuestro servidor Prometheus es colocarlo detrás de un proxy inverso para que luego podamos agregar SSL y una capa de autenticación sobre la interfaz web predeterminada sin restricciones de Prometheus.

Instalamos nginx

```bash
apt install nginx
```

Vamos a crear un sitio nuevo

```
vim /etc/nginx/sites-enabled/prometheus
```

Agregamos el siguiente contenido

```vim
server {
    listen 80;
    listen [::]:80;
    server_name  tu-dominio.com;

    location / {
        proxy_pass           http://localhost:9090/;
    }
}
```

comprobamos que las configuraciones sean correcta

```bash
nginx -t
```

Reiniciamos nginx y comprobamos su estado

```bash
systemctl start nginx
systemctl status nginx
```

Ahora ya podemos acceder a http://tu-dominio.com

Si accedemos por la ip veremos la web por defecto de nginx si no queremos que esto suceda podemos eliminar el siguiente fichero

```bash
rm /etc/nginx/sites-enabled/default
```

Para activar Let's Encrypt tenemos que instalar nuestro amigo certbot

```bash
apt install certbot python3-certbot-nginx
```

ahora solo tenemos que ejecutar e ir respondiendo las preguntas que nos hace

```bash
certbot --nginx
```

Podemos ver los cambios que realizo certbot en el fichero de nuestro dominio

```bash
vim /etc/nginx/sites-enabled/prometheus
```

```vim
server {
    server_name  tu-dominio.com;

    location / {
        proxy_pass           http://localhost:9090/;
    }

    listen [::]:443 ssl ipv6only=on; # managed by Certbot
    listen 443 ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/tu-dominio.com/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/tu-dominio.com/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot

}
server {
    if ($host = tu-dominio.com) {
        return 301 https://$host$request_uri;
    } # managed by Certbot


    listen 80;
    listen [::]:80;
    server_name  tu-dominio.com;
    return 404; # managed by Certbot


}
```

Como no queremos que todo el mundo tenga acceso a nuestros datos vamos a poner una autentificación y a cerrar el puerto 9090 y 9100 para que no puedan acceder a la información de manera externa y sin autorización

Instalamos apache2-utils

```bash
apt install apache2-utils
```

Creamos un fichero de contraseñas con el usuario admin y ponemos la contraseña que queramos

```bash
htpasswd -c /etc/nginx/.htpasswd admin
```

editamos el fichero de configuración de nuestro prometheus

```bash
vim /etc/nginx/sites-enabled/prometheus
```

Y agregamos lo siguiente para que use el fichero de contraseñas

```bash
server {
    ...

    #addition authentication properties
    auth_basic  "Protected Area";
    auth_basic_user_file /etc/nginx/.htpasswd;

    location / {
        proxy_pass           http://localhost:9090/;
    }

    ...
}
```

comprobamos que las configuraciones sean correcta

```bash
nginx -t
```

Reiniciamos nginx y comprobamos su estado

```bash
systemctl restart nginx
systemctl status nginx
```

Los puertos 9000 y 9100 siguen abiertos por lo que vamos a cerrarlos con iptables

```bash
iptables -A INPUT -p tcp -s localhost --dport 9090 -j ACCEPT
iptables -A INPUT -p tcp --dport 9090 -j DROP
iptables -A INPUT -p tcp -s localhost --dport 9100 -j ACCEPT
iptables -A INPUT -p tcp --dport 9100 -j DROP
iptables -L
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

### node exporter

Descargamos

```bash
wget https://github.com/prometheus/node_exporter/releases/download/v1.3.1/node_exporter-1.3.1.linux-amd64.tar.gz
```

descomprimimos

```bash
tar xvf node_exporter-*.tar.gz
```

Creamos usuarios

```bash
useradd --no-create-home --shell /bin/false node_exporter
```

Entramos a la carpeta de node\_exporter y copiamos el binario

```bash
cd node_exporter-1.3.1.linux-amd64
cp ./node_exporter /usr/local/bin
```

Igual que con prometheus vamos a añadirlo en el systemd para poderlo arrancar en el arranque del sistema de manera facil.

```bash
vim /etc/systemd/system/node_exporter.service
```

```bash
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
```

recargamos systemd

```bash
systemctl daemon-reload
```

Arrancamos el node\_exporter

```bash
systemctl start node_exporter
```

Comprobamos que funciona

```bash
systemctl status node_exporter
```

Lo activamos para arranque de maquina

```bash
systemctl enable node_exporter
```

Vamos a comunicar el node\_exporter con Prometheus para ello vamos a editar la configuración de Prometheus

```bash
vim /etc/prometheus/prometheus.yml
```

Y añadimos lo siguiente en la zona de scrape\_configs

```bash
  - job_name: 'node_exporter'
    scrape_interval: 5s
    static_configs:
      - targets: ['localhost:9100']
```

&#x20;Reiniciamos prometheus

```bash
systemctl restart prometheus
```

Ahora repetimos el proceso con el resto de los nodos y servidores de nuestro HA y ya podemos consultar la información de los diferentes nodos

![Prometheus basico](imgs/prometheus.png)

### Apache exporter

### &#x20;Mysql Exporter

Vamos a monitorizar nuestro cluster de galera, para ello lo primera que hacemos en uno de los nodos crear un usuario

```sql
CREATE USER 'exporter'@'localhost' IDENTIFIED BY 'XXXXXXXX' WITH MAX_USER_CONNECTIONS 3;
GRANT PROCESS, REPLICATION CLIENT, REPLICATION SLAVE, SUPER SELECT ON *.* TO 'exporter'@'localhost
FLUSH PRIVILEGES;
```

Descargamos el mysql exporter en todos los nodos

```bash
wget https://github.com/prometheus/mysqld_exporter/releases/download/v0.14.0/mysqld_exporter-0.14.0.linux-amd64.tar.gz
```

descomprimimos

```bash
tar xvf mysqld_exporter-*.tar.gz
```

Creamos usuarios

```bash
useradd --no-create-home --shell /bin/false mysql_exporter
```

Entramos a la carpeta de mysql\_exporter y copiamos el binario

```bash
cd mysqld_exporter-0.14.0.linux-amd64
cp ./mysqld_exporter /usr/local/bin/
```

Creamos un fichero para guardar el usuario y contraseña que hemos asignado antes

```bash
vim /etc/.mysqld_exporter.cnf
```

```vim
[client]
user=exporter
password=XXXXXXXX
```

Igual que con prometheus vamos a añadirlo en el systemd para poderlo arrancar en el arranque del sistema de manera facil.

```bash
vim /etc/systemd/system/mysql_exporter.service
```

```bash
[Unit]
Description=Prometheus MySQL Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=mysql_exporter
Group=mysql_exporter
Type=simple
Restart=always
ExecStart=/usr/local/bin/mysqld_exporter \
--config.my-cnf /etc/.mysqld_exporter.cnf \
--collect.global_status \
--collect.info_schema.innodb_metrics \
--collect.auto_increment.columns \
--collect.info_schema.processlist \
--collect.binlog_size \
--collect.info_schema.tablestats \
--collect.global_variables \
--collect.info_schema.query_response_time \
--collect.info_schema.userstats \
--collect.info_schema.tables \
--collect.perf_schema.tablelocks \
--collect.perf_schema.file_events \
--collect.perf_schema.eventswaits \
--collect.perf_schema.indexiowaits \
--collect.perf_schema.tableiowaits \
--collect.slave_status \

[Install]
WantedBy=multi-user.target
```

recargamos systemd

```bash
systemctl daemon-reload
```

Arrancamos el mysql\_exporter

```bash
systemctl start mysql_exporter
```

Comprobamos que funciona

```bash
systemctl status mysql_exporter
```

Lo activamos para arranque de maquina

```bash
systemctl enable mysql_exporter
```

Vamos a comunicar el mysql\_exporter con Prometheus para ello vamos a editar la configuración de Prometheus

```bash
vim /etc/prometheus/prometheus.yml
```

Y añadimos lo siguiente en la zona de scrape\_configs

```vim
  - job_name: "node1-mysql"
    scrape_interval: "15s"
    static_configs:
      - targets: ['xxx.xxx.xxx:9104']
```

&#x20;Reiniciamos prometheus

```bash
systemctl restart prometheus
```

Ahora repetimos el proceso con el resto de los nodos de galera pero sin tener que crear el usuario porque ya se replico automaticamente.

### Grafana

```bash
apt-get install -y adduser libfontconfig1
wget https://dl.grafana.com/oss/release/grafana_6.7.3_amd64.deb
dpkg -i grafana_6.7.3_amd64.deb
```

```bash
systemctl daemon-reload
systemctl enable grafana-server
systemctl start grafana-server
```
