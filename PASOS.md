# Para el docker compose 

Necesitas tener .env

```
ROOT_PASSWORD=DbTest2024!
DB_USER=usuario
CLIENT1_PASSWORD=vaveeneil6ohcaiXooGh
CLIENT2_PASSWORD=Iu7iewohquo0YuQu6ahl
CLIENT3_PASSWORD=ThaiLai0Aira4mahngoo
CLIENT4_PASSWORD=Nev3gi2vaixu5Ioth2oa

```
## Levanta con 

```
sudo docker-compose up -d
```

# Configuracion para tener base de datos replicada 

Tener en cuenta que todos los cambios deben pasar por la base de datos maestra 1

## Aplicar docker compose

Una vez culminado acceder al maestro la base de datos 1

## Crear usuario 
```
CREATE USER 'replica'@'%' IDENTIFIED BY 'vaveeneil6ohcaiXooGh';

GRANT REPLICATION SLAVE ON *.* TO 'replica'@'%';

FLUSH PRIVILEGES;
```

Verificar el estado del maester y extraer valores para MASTER_LOG_FILE y MASTER_LOG_POS

```
SHOW MASTER STATUS;
```

Verificar que ip posee el contenedor principal maestro 

```
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' mariadb_slave-client1-1

```

En vez de la ip puede ser el nombre del host que se le asigno al master 

```
docker inspect -f '{{.Config.Hostname}}' mariadb_slave_client1_1

```

Aplicar este ajuste en la base de datos del maestro

```
CHANGE MASTER TO MASTER_HOST='172.27.0.2', MASTER_USER='replica', MASTER_PASSWORD='vaveeneil6ohcaiXooGh', MASTER_LOG_FILE='mysql-bin.000001', MASTER_LOG_POS=543;
```

Aqui se verifica que el maestro este con server id 1 y todos los demas con otros id de server respectivamente sin duplicaciones 

```
SHOW VARIABLES LIKE 'server_id';
```

Verificar el usuario de replica 

```
SELECT user, host FROM mysql.user WHERE user = 'replica';
```

# Configuraciones en los esclavos 

Acceder a cada uno y aplicar el mismo ajuste teniendo en cuenta los datos extraidos anteriormente 

Con ip
```
CHANGE MASTER TO MASTER_HOST='172.27.0.2', 
MASTER_USER='replica', 
MASTER_PASSWORD='vaveeneil6ohcaiXooGh', 
MASTER_LOG_FILE='mysql-bin.000001', 
MASTER_LOG_POS=543;
```

Con host de docker (En mi caso no le renombre al host)

```
CHANGE MASTER TO MASTER_HOST='f530da13e128', 
MASTER_USER='replica', 
MASTER_PASSWORD='vaveeneil6ohcaiXooGh', 
MASTER_LOG_FILE='mysql-bin.000008', 
MASTER_LOG_POS=991;
```

Aplicar esto para iniciar la escucha 

```
START SLAVE;
```

Con esto se verifica el estado 
```
SHOW SLAVE STATUS;
```

Si todo funciona muestra este mensaje en el status 
```
Slave has read all relay log; waiting for more updates
```


## Extras

Para saltar algun error 

```
STOP SLAVE;
SET GLOBAL SQL_SLAVE_SKIP_COUNTER = 1;
START SLAVE;
```