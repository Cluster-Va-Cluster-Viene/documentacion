# Visión general

Se va a construir un cluster para mejorar la disponibilidad de los sistemas, optimizar los tiempos de carga y mejorar la seguridad, todos los servidores se montan sobre Ubuntu 22.04 LTS (Jammy Jellyfish), se construirá en maquinas virtuales en VirtualBox

## Objetivos

* Alta disponibilidad entre balanceadores de carga para tener siempre un punto de entrada disponible
* Disponibilidad de un WaF para mejorar la seguridad de las aplicaciones.
* Alta disponibilidad de los ficheros mediante GlusterFs teniendo replicación en todos los nodos.
* Alta disponibilidad de base de datos con Mysql Galera

## Funcionamiento general

* Las peticiones entrar por los balanceadores de carga, donde se finaliza el SSL para poder realizar el análisis de las conexiones.
* Se remiten al WaF que revisar la seguridad de la petición y mitigar los intentos de hackeos como SQLi, XSS…
* Una vez analizada la petición es devuelta al balanceador de carga, si la petición es buena la envía a los nodos GLAMP, si la petición es mala se informa al balanceador para que a la décima petición se corte la conexión contra el cluster.

![Estructura servidores](imgs/estructura\_servidores.png)

## Comparte <a href="#feedback" id="feedback"></a>

La obre se distribuye bajo Licencia Creative Commos, por lo que eres libre de compartirla como quieras y realizar modificaciones u obras derivadas de ella.

![](https://i.creativecommons.org/l/by-sa/4.0/88x31.png)

Cluster va Cluster viene por [Rafael Otal Simal](https://legacy.gitbook.com/book/goldrak/seguridad-web-101/details) se distribuye bajo una [Licencia Creative Commons Atribución-CompartirIgual 4.0 Internacional](http://creativecommons.org/licenses/by-sa/4.0/)
