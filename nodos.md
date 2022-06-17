# Nodos

Vamos con la configuración de los nodos que contendrán Apache2 + PHP + MySQL Galera Clusters + GlusterFS + memcached

### GlusterFs

El primer problema que nos encontramos cuando queremos utilizar un sistema de nodos es que cada servidor tiene sus discos duros y por lo tanto tenemos que buscar alguna manera de sincronizar los ficheros, para esto vamos a usar [GlusterFS](https://www.gluster.org/)

Tenemos varios métodos de sincronización de ficheros pero nosotros vamos a utilizar el tipo replica

![GlusterFS Replica](.gitbook/assets/GlusterFs\_Replicacion.png)

Lo primero que vamos a realizar es añadir al fichero **hosts** las direcciones de la red interna de los diferentes nodos para facilitarnos la vida en la comunicación con gusterfs

```bash
vim /etc/hosts
```

```vim
XXX.XXX.XXX.XXX   node-1
XXX.XXX.XXX.XXX   node-2
XXX.XXX.XXX.XXX   node-3
```

Para tener la ultima versión de GlusterFS vamos a agregar el respositorio PPA

```bash
add-apt-repository ppa:gluster/glusterfs-10
apt-get update
```

Instalamos el paquete software-properties-common

```bash
apt install software-properties-common
```

Instalamos glusterfs-server en todos los nodos

```bash
apt install glusterfs-server
```

Arrancamos GlusterFs

```bash
systemctl start glusterd.service
```

Y lo habilitamos para que arranque al arrancar el sistema

```bash
systemctl enable glusterd.service
```

Vamos a conectar los nodos entre si, no es necesario conectar el nodo-1 a si mismo por lo que lo obviamos

```bash
gluster peer probe node-2
gluster peer probe node-3
```

Para comprobar que los nodos están conectados disponemos de los siguientes comandos

```bash
gluster peer status
```

```bash
Number of Peers: 2

Hostname: node-2
Uuid: 544725c4-a02b-449b-8464-744b97bf08b6
State: Peer in Cluster (Connected)

Hostname: node-3
Uuid: 3c1b7579-deb7-4900-a1f8-83ee71140ee0
State: Peer in Cluster (Connected)
```

```bash
gluster pool list
```

```bash
UUID                                    Hostname        State
544725c4-a02b-449b-8464-744b97bf08b6    node-2          Connected
3c1b7579-deb7-4900-a1f8-83ee71140ee0    node-3          Connected
d340a141-848d-4ee5-96d7-71605c30a023    localhost       Connected
```

**NOTA**: Para servers en producción es recomendable usar glusterfs en otra partición que no sea la de sistema.

```bash
mkdir -p /gfsvolume/gv0
```

Creamos un volumen con los 3 nodos en modo replica

```bash
gluster volume create clusterVaClusterViene replica 3 node-1:/gfsvolume/gv0/ node-2:/gfsvolume/gv0 node-3:/gfsvolume/gv0
```

Si en el futuro quisiéramos agregar un nuevo nodo para dar más maquinas a nuestro clusters, lo realizaríamos de la siguiente manera

```bash
gluster volume add-brick clusterVaClusterViene replica 4 node-4:/gfsvolume/gv0/
```

Una vez listo el el volumen lo arrancamos

```bash
gluster volume start clusterVaClusterViene
```

Podemos comprobar el estado del volumen

```bash
gluster volume status
```

```bash
Status of volume: clusterVaClusterViene
Gluster process                             TCP Port  RDMA Port  Online  Pid
------------------------------------------------------------------------------
Brick node-1:/gfsvolume/gv0                 49152     0          Y       30787
Brick node-2:/gfsvolume/gv0                 49152     0          Y       30424
Brick node-3:/gfsvolume/gv0                 49152     0          Y       25287
Self-heal Daemon on localhost               N/A       N/A        Y       30808
Self-heal Daemon on node-2                  N/A       N/A        Y       30445
Self-heal Daemon on node-3                  N/A       N/A        Y       25308

Task Status of Volume clusterVaClusterViene
------------------------------------------------------------------------------
There are no active volume tasks
```

Ahora para poder utilizar nuestro volumen vamos a montarlo en la carpeta `/var/www/html/` para ello en cada nodo editamos el **fstab**

```bash
vim /etc/fstab
```

Y agregamos la siguiente linea

```vim
node-1:/clusterVaClusterViene /var/www/html/ glusterfs defaults,_netdev 0 0
```

guardamos y montamos el nuevo punto

```bash
mount -a
```

Ahora cada fichero que se cree modifique o elimine de uno de los nodos se replicara en los otros nodos de manera automática.

{% hint style="danger" %}
Existe un[ bug en glusterfs en ubuntu ](https://bugs.launchpad.net/ubuntu/+source/glusterfs/+bug/876648)si se usan los volúmenes en los propios servidores.
{% endhint %}

Durante el proceso de arranque, GlusterFS tardará un poco en iniciarse. systemd-mount, que maneja los puntos de montaje desde /etc/fstab, se ejecutará antes de que el servicio glusterfs-server termine de iniciarse.

El montaje fallará, por lo que terminará sin su volumen montado después de reiniciar.

La solución es crear el siguiente directorio

```bash
mkdir -p /etc/systemd/system/www.mount.d/
```

y dentro el siguiente fichero

```bash
vim /etc/systemd/system/www.mount.d/override.conf
```

```vim
[Unit]
After=glusterfs-server.service
Wants=glusterfs-server.service
```

recargamos systemd

```bash
systemctl daemon-reload
```

Y en el siguiente arranque ya tendremos el punto de montaje cargado automaticamente.

### Apache + PHP

Ahora que ya nos funciona el sistema de archivos compartidos vamos instalar apache2 + php

```bash
apt install apache2
```

instalamos php y los complementos para mysql

```bash
apt install php libapache2-mod-php php-mysql
```

Agregamos apache al arranque del sistema

```bash
systemctl enable apache2
```

#### Eliminar el banner de la versión del servidor

Apache por defecto nos muestra en las peticiones su versión y el sistema operativo en el que esta corriendo.

Vamos a ocultar las versiones y el sistema operativo de nuestro servidor para dar la menor información posible a los atacantes, para ello editamos el siguiente fichero

```bash
vim /etc/apache2/conf-available/security.conf
```

Dejando los siguiente valores de esta manera

```bash
ServerTokens Prod

ServerSignature Off
```

#### Deshabilitar el listado de directorios en el navegador

Una configuración que trae por defecto apache es listar los ficheros de los directorios por lo que se puede producir una fuga de información si no nos acordamos de poner un fichero index.php o index.html para evitar el listado, para no tener que revisar todas las carpetas.

Editamos el fichero

```bash
vim /etc/apache2/apache2.conf
```

y buscamos donde se encuentre la opcion Indexes y le ponemos un guion delante quedando de la siguiente manera o podemos eliminar directamente la opcion

```vim
<Directory /var/www/>
        Options -Indexes FollowSymLinks
        AllowOverride None
        Require all granted
</Directory>
```

Reiniciamos apache para aplicar los cambios.

```bash
systemctl restart apache2
```

### Mariadb Galera Cluster

Tenemos muchas maneras de montar un sistema de base de datos en modo clúster con [mysql](https://www.mysql.com/products/cluster/), [Percona](https://www.percona.com/software/mysql-database/percona-xtradb-cluster), [galera](https://galeracluster.com/), en nuestro caso lo vamos a realizar con MariaDB.

Nos descargamos el siguiente paquete para configurar los repositorios según nuestro sistema

```bash
wget https://downloads.mariadb.com/MariaDB/mariadb_repo_setup
```

Damos permisos de ejecución

```bash
chmod +x mariadb_repo_setup
```

Ejecutamos el script para que nos agregue los repositorios

{% hint style="danger" %}
Como los repositorios de mariadb no han sido actualizados todavía a "22.04 (Jammy Jellyfish)" No lanzaremos los repositorios oficiales, por lo que las versiones que se nos instalaran son las siguiente:

* mariadb 10.6.7 vs 10.6.8 [_Changelog_](https://mariadb.com/kb/en/mariadb-1068-release-notes/)\_\_
* galeraCuster: 26.4.9 vs 26.4.12 [_Changelog_](https://fromdual.com/galera-cluster-release-notes)\_\_

Cuando se lance el repositorio seria recomendable activarlos.
{% endhint %}

```bash
./mariadb_repo_setup
```

Si no tenemos el siguiente paquete nos dará error a la hora de agregar los repositorios

```bash
apt install apt-transport-https
```

Actualizamos

```bash
apt update
```

Instalamos MariaDB y galera

```bash
apt install mariadb-server mariadb-backup galera-4
```

Ahora vamos a pasar a configurar los nodos, las configuraciones entre ellos son casi idénticas.

De forma predeterminada, MariaDB está configurado para comprobar el directorio /etc/mysql/mariadb.conf.d donde busca configuraciones adicionales desde los archivos que terminan en .cnf. Por lo que vamos a crear el siguiente fichero.

```bash
vim /etc/mysql/mariadb.conf.d/61-clustervaclusterviene.cnf
```

Agregamos la siguiente configuración:

* bind-address: En la ip que escuchara galera.
* wsrep\_cluster\_name: El nombre de nuestro cluster
* wsrep\_cluster\_address: Los nodos del cluster
* wsrep\_node\_address: La ip del nodo
* wsrep\_node\_name: EL nombre del nodo

```vim
[mariadb]
binlog_format=ROW
default-storage-engine=innodb
innodb_autoinc_lock_mode=2
bind-address=xxx.xxx.xxx.xxx

# Galera Provider Configuration
wsrep_on=ON
wsrep_provider=/usr/lib/galera/libgalera_smm.so

# Galera Cluster Configuration
wsrep_cluster_name="clusterVaClusterViene_cluster"
wsrep_cluster_address="gcomm://xxx.xxx.xxx.xxx,yyy.yyy.yyy.yyy,zzz.zzz.zzz.zzz"

# Galera Synchronization Configuration
wsrep_sst_method=rsync

# Galera Node Configuration
wsrep_node_address="xxx.xxx.xxx.xxx"
wsrep_node_name="node-1"
```

Replicamos esta configuración en el resto de los nodos cambiado los diferentes datos.

Activamos mysql para que arranque al comienzo

```bash
systemctl enable mariadb
```

Para configurar el primer nodo, deberá usar una secuencia de comandos de inicio especial. Por la manera en que configuró el clúster, cada nodo que se conecte intentará establecer conexión con al menos otro nodo especificado en su archivo 61-clustervaclusterviene.cnf para obtener su estado inicial. Un `systemctl start mysql` normal fallaría porque no hay en ejecución nodos con los que el primer nodo se pueda conectar.

Ejecutamos en el primer nodo

```bash
galera_new_cluster
```

Una vez terminado podemos comprobar el tamaño del cluster

```bash
mysql -u root -p -e "SHOW STATUS LIKE 'wsrep_cluster_size'"
```

```sql
+--------------------+-------+
| Variable_name      | Value |
+--------------------+-------+
| wsrep_cluster_size | 1     |
+--------------------+-------+
```

En el resto de los nodos arrancamos mysql de manera normal

```bash
systemctl start mariadb
```

Repetimos el comando de antes para ver el aumento del cluster

```bash
mysql -u root -p -e "SHOW STATUS LIKE 'wsrep_cluster_size'"
```

```sql
+--------------------+-------+
| Variable_name      | Value |
+--------------------+-------+
| wsrep_cluster_size | 2     |
+--------------------+-------+
```

Y así sucesivamente entre todos los nodos que tengamos.

Vamos a probar la replicación

```bash
mysql -u root -p -e 'CREATE DATABASE clusterVaClusterViene;
CREATE TABLE clusterVaClusterViene.equipment ( id INT NOT NULL AUTO_INCREMENT, type VARCHAR(50), quant INT, color VARCHAR(25), PRIMARY KEY(id));
INSERT INTO clusterVaClusterViene.equipment (type, quant, color) VALUES ("slide", 2, "blue");'
```

Con este comando crearemos la base de datos clusterVaClusterViene y la table equipment, donde agregaremos un row con los valores, slide, 2, blue.

Ahora vamos a otro nodo para ver que los datos son accesibles y ejecutamos

```bash
mysql -u root -p -e 'SELECT * FROM clusterVaClusterViene.equipment;'
```

```sql
+----+-------+-------+-------+
| id | type  | quant | color |
+----+-------+-------+-------+
|  1 | slide |     2 | blue  |
+----+-------+-------+-------+
```

Insertamos otro dato y repetimos el proceso en el resto de nodos.

```bash
mysql -u root -p -e 'INSERT INTO clusterVaClusterViene.equipment (type, quant, color) VALUES ("swing", 10, "yellow");'
```

Para que la gente no se pueda conectar por la ip publica a los puertos 80 y 443 lo bloqueamos mediante iptables en la interfaz publica

```bash
iptables -A INPUT -i {interface} -p tcp --destination-port 80 -j DROP
iptables -A INPUT -i {interface} -p tcp --destination-port 443 -j DROP
```

Como las iptables si se reinicia el equipo se pierden y no queremos estar agregándolas manualmente vamos a instalar el paquete iptables-persistent

```bash
apt install iptables-persistent
```

Las configuraciones se guardan en estos dos ficheros

```bash
/etc/iptables/rules.v4

/etc/iptables/rules.v6
```

Si hacemos algún cambio a las iptables nos tenemos que acordar de guardarlas

```bash
iptables-save > /etc/iptables/rules.v4

iptables-save > /etc/iptables/rules.v6
```

### Memcached

Ahora que ya solo nos falta poder tener las sesiones compartidas entre los diferentes nodos para ello vamos a usar memcached

```bash
sudo apt install php-memcached memcached
```

Si vamos a la configuración vemos que la ip de escucha es localhost pero para poderlo tener en modo cluster tenemos que decirle que escuche en nuestra red interna.

```bash
vim /etc/memcached.conf
```

```vim
# Specify which IP address to listen on. The default is to listen on all IP addresses
# This parameter is one of the only security measures that memcached has, so make sure
# it's listening on a firewalled interface.
-l 127.0.0.1
```

cambiándola de la siguiente manera en cada nodo

```vim
-l 192.168.10.X
```

Una vez que tenemos configurado reiniciamos los nodos e iremos a configurar php

```bash
systemctl restart memcached
```

Abrimos la configuración de php y buscamos la sección de sesiones

```bash
vim /etc/php/8.1/apache2/php.ini
```

Y la dejamos de la siguiente manera

```vim
session.save_handler = memcache
session.save_path = 'tcp://192.168.10.3:11211,tcp://192.168.10.4:11211,tcp://192.168.10.5:11211'
```

Ahora vamos a configurar el modulo de memcached

```bash
vim /etc/php/8.1/mods-available/memcached.ini
```

```vim
memcache.allow_failover=1
memcache.session_redundancy=4
```

La directiva memcache.session\_redundancy debe ser igual a la cantidad de servidores Memcached + 1 para que la información de la sesión se replique en todos los servidores. Esto se debe a un error en PHP.

Estas directivas permiten la conmutación por error y la redundancia de la sesión, por lo que PHP escribe la información de la sesión en todos los servidores especificados en session.save\_path; similar a una configuración RAID-1.

```bash
systemctl restart apache2
```
