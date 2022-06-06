# Analizando ficheros

Vamos a configurar diferentes medidas de seguridad que nos permitan vigilar los ficheros del sistema para luchar contra el malware, rootkits y diferentes bichos.

### CalmAV

ClamAV es un excelente antivirus gratis Open Source que luego nos servira para darle mas potencia a maldet

Instalamos los paquetes

```bash
apt install clamav clamav-daemon clamav-freshclam 
```

Si queremos escanear por ejemplo el directorio actual lo podemos hacer de la siguiente manera

```bash
clamscan -v .
```

Vamos a configurar CalmAV para que cada vez que se acceda a un fichero se analice, el escaneo on-access se instalará en el núcleo de GNU/Linux, en el kernel.

Para configurar el demonio de clamav vamos a editar el fichero

```bash
vim /etc/clamav/clamd.conf
```

Este demonio del sistema usa la librería del ClamAV para escanear ficheros. Es la mejor opción para realizar escaneos en paralelo, sin afectar demasiado al rendimiento del sistema. Este servicio es el que usará el clamonacc para engancharse a los accesos al sistema de ficheros que realiza el kernel de Linux.

Ya podemos lanzar el comando del sistema clamonacc, pero antes conviene configurarlo, para ello añadimos las siguientes lineas, OnAccessIncludePath podemos agregar tantas como sean necesarias.

```
OnAccessIncludePath /home/geekshub/
OnAccessExcludeUname clamav
OnAccessExcludeUname root
OnAccessPrevention yes
```

Lo ejecutamos para probarlo si todo va bien lo vamos a demonizar.

```bash
sudo clamonacc
```

```bash
vim /etc/systemd/system/clamonacc.service
```

```vim
[Unit]
Description=ClamAV On Access
Requires=clamav-daemon.service
After=clamav-daemon.service syslog.target nss-lookup.target network.target

[Service]
Type=simple
User=root
ExecStart=/usr/sbin/clamonacc -F --fdpass --log=/var/log/clamav/clamonacc.log --move=/opt/quarantine
Restart=on-failure
RestartSec=20s

[Install]
WantedBy=multi-user.target

```

y lo activamos

```bash
systemctl daemon-reload
systemctl enable clamav-daemon
systemctl start clamav-daemon
systemctl enable clamonacc
systemctl start clamonacc
```

para probarlo podemos tirar vamos a usar eicar

```bash
wget https://secure.eicar.org/eicar.com
```

Si todo va bien tendremos el fichero en la carpeta de cuarentena.&#x20;

### Maldet

Maldet es un antimalware que usamos desde la terminal de Linux y que utiliza un potente escáner y una serie de patrones o firmas para detectar malware en el sistema que puede utilizar el motor de clamav para ser mas rápido y potente.

Lo descargamos

```bash
wget http://www.rfxn.com/downloads/maldetect-current.tar.gz
```

Descomprimimos&#x20;

```bash
tar -xvf maldetect-current.tar.gz
```

Lo instalamos

```bash
cd maldetect-1.6.4/
./install.sh
```

Ahora vamos a configurarlo

```bash
vim  /usr/local/maldetect/conf.maldet
```



* **email\_alert:** Nos permite poner un 0 o un 1, si ponemos 1 activaremos los avisos de Maldet mediante correo electrónico.
* **email\_addr:** Nos permite introducir una dirección de correo electrónico a donde se enviaran los avisos de detección.
* **email\_ignore\_clean:** Nos permite que nos lleguen correos de limpieza de malware además de los correos de detección, permite poner 0 o 1, si ponemos 0 nos llegaran los correos.
* **scan\_max\_depth:** Nos permite especificar la profundidad máxima al escanear, normalmente trae 15, pero lo hemos cambiado a 30.
* **scan\_min\_filesize:** Nos permite especificar el tamaño mínimo de archivo a escanear, descartando en los análisis los archivos más pequeños de lo especificado en esta variable.
* **scan\_max\_filesize:** Nos permite especificar el tamaño máximo del archivo a escanear, descartando en los análisis los archivos más grandes de lo especificado en esta variable, nosotros lo subimos a más de 300 MB, es decir, más de 300000k.
* **quarantine\_hits:** Por defecto solo alerta pero si lo ponemos a 1 lo movera a cuarentena y los eliminara.
* **quarantine\_clean**: Intentara limpiar el fichero de la infección tiene que tener activo quarantine\_hits
* **default\_monitor\_mode**: Configuramos la ruta que queremos monitorizar, por ejemplo /var/www/?/public\_html

### Anti-RootKits

Un rooktit es en esencia un programa o conjunto de programas que generalmente se utilizan para camuflar otros procesos maliciosos, archivos, puertas traseras o cambios de registro sospechosos en el sistema.

#### Chkrootkit

Instalamos el paquete

```bash
apt install chkrootkit
```

ahora solo tenemos que ejecutarlo

```bash
chkrootkit -q
```

#### Rkhunter

Instalamos el paquete

```bash
apt install rkhunter
```

Realizamos el escaneo

```bash
rkhunter --check
```

Cuando acabemos el escaneo hay que recordar que puede salir algun falso positivo, como por ejemplo PortSentry segun la configuración que tengamos realizada.
