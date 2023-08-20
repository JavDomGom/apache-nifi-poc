# Escenario 1
Ingestar cada poco tiempo un log disponible en un servidor SFTP y leer solo las nuevas líneas, como si se tratara de un `tail -f`. Finalmente llevaremos esas líneas nuevas del log a un topic de Kafka configuraco como `log-compaction` ([doc](https://docs.confluent.io/kafka/design/log_compaction.html)). Esto último en realidad no es necesario, pero si recomendable por si por cualquier motivo hubiera alguna duplicidad de información en el origen de datos. 

## Inicio del stack

Primero levantamos el stack con `docker-compose`:
```commandline
~$ docker-compose up
[+] Running 6/0
 ✔ Network poc-net      Created                                                                                                                                                                               0.1s 
 ✔ Volume "nifi_data"   Created                                                                                                                                                                               0.0s 
 ✔ Volume "kafka_data"  Created                                                                                                                                                                               0.0s 
 ✔ Container kafka      Created                                                                                                                                                                               0.0s 
 ✔ Container nifi       Created                                                                                                                                                                               0.0s 
 ✔ Container sftp       Created                                                                                                                                                                               0.0s 
Attaching to kafka, nifi, sftp
...
```

## Servicio SFTP

Entramos en el contenedor `sftp`:
```commandline
~$ docker exec -it sftp bash
```

Creamos un script que nos va a servir para generar logs a nuestro antojo.
```commandline
root@sftp:/# echo "echo \"\$(date  +'%Y-%m-%d %H:%M:%S') [INFO] Lorem ipsum dolor sit amet, consectetur adipiscing elit\" >> upload/test.log" > /home/usutest/generate_log.sh
```

Le damos permisos para el usuario `usutest` y de ejecución:
```commandline
root@sftp:/# chmod 750 /home/usutest/generate_log.sh
root@sftp:/# chown 1000:100 /home/usutest/generate_log.sh
```

Nos cambiamos de sesión a nuestra cuenta del usuario `usutest` y comprobamos que todo está OK:
```commandline
root@sftp:/# su - usutest
$ bash
usutest@sftp:~$ ls -lrt
total 8
drwxr-xr-x 2 usutest users 4096 Aug 20 15:52 upload
-rwxr-x--- 1 usutest users  119 Aug 20 15:57 generate_log.sh
```

En nuestro host local, desde otra shell nos conectamos al servicio `sftp` con el usuario `usutest` e introducimos la password `passwdtest`:
```commandline
~$ sftp -P 2222 usutest@localhost
usutest@localhost's password:
```

Comprobamos que no tienen ningún archivo subido a la carpeta `/upload`:
```commandline
sftp> ls -lrt upload/
sftp>
```

Desde la shell del contenedor `sftp` ejecutamos nuestro script para generar nuestro primer log en un archio `temp.log`:
```commandline
usutest@sftp:~$ ./generate_log.sh
```

Volvemos a comprobar si hay contenido dentro del directorio `/upload` de nuestro servicio SFTP:
```commandline
sftp> ls -lrt upload/
-rw-r--r--    1 1000     100            83 Aug 20 16:06 test.log
```

## Apache Kafka

Entramos en el contenedor `kafka`:

```commandline
~$ docker exec -it kafka bash
```

Creamos un topic nuevo llamado `topic-nifi`:
```commandline
I have no name!@kafka:/$ kafka-topics.sh --create --bootstrap-server kafka:9092 --topic topic-nifi
Created topic topic-nifi.
```

Listamos los topics existentes, solo para comprobar que ya existe nuestro topic:
```commandline
I have no name!@kafka:/$ kafka-topics.sh --list --bootstrap-server kafka:9092
topic-nifi
```

Ejecutamos un consumer para ver los mensajes que van entrando en el topic `topic-nifi`. esto dejará la sheel abierta:
```commandline
I have no name!@kafka:/$ kafka-console-consumer.sh --bootstrap-server kafka:9092 --topic topic-nifi --from-beginning --group group-a
```

## Apache NiFi

1. Entramos en el frontal de Apache NiFi desde un navegador en la URL: https://localhost:8443/nifi/
2. Importamos la plantilla `poc-sftp2kafka.xml` y la abrimos. En el processor `GetSFTP` hay que establecer de nuevo la contraseña, ya que esta no se guarda al exportar la plantilla.
3. Iniciamos todos los processors y empezaremos a ver cómo entran los primeros logs.
4. Desde la shell del servicio `sftp` puedes ejecutar el script `./generate_log.sh` tantas veces como quieras, que irá generando logs en el archivo `test.log`, este estará disponible en el servidor SFTP y desde NiFi se ingestará. El resto de processors ya leerán solo las nuevas líneas.
