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

![Este obra está bajo una licencia de Creative Commons Reconocimiento-CompartirIgual 4.0 Internacional](https://i.creativecommons.org/l/by-sa/4.0/88x31.png)
