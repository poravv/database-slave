# MariaDB con Replicación Maestro-Esclavo Optimizado para Producción

Este proyecto configura un clúster de MariaDB con replicación maestro-esclavo optimizado para entornos de producción. El sistema consta de 1 nodo maestro (client1) y 3 nodos esclavos (client2, client3, client4).

## Requisitos previos

- Docker y Docker Compose instalados
- Al menos 4GB de RAM disponible
- Espacio en disco suficiente para almacenar las bases de datos

## Estructura de archivos

```
docker-compose.yml       # Configuración de los contenedores
Dockerfile.client1       # Dockerfile para el servidor maestro
Dockerfile.client2-4     # Dockerfiles para los servidores esclavos
my1.cnf                  # Configuración optimizada del servidor maestro
my2-4.cnf                # Configuraciones optimizadas de los servidores esclavos
init.sql                 # Script SQL inicial ejecutado en todos los nodos
```

## Configuración inicial

### 1. Configurar variables de entorno

Cree un archivo `.env` en el directorio raíz con las siguientes variables:

```
ROOT_PASSWORD=DbTest2024!
DB_USER=usuario
CLIENT1_PASSWORD=vaveeneil6ohcaiXooGh
CLIENT2_PASSWORD=Iu7iewohquo0YuQu6ahl
CLIENT3_PASSWORD=ThaiLai0Aira4mahngoo
CLIENT4_PASSWORD=Nev3gi2vaixu5Ioth2oa
```

### 2. Iniciar los contenedores

```bash
docker compose up -d
```

## Configuración de la replicación

### 1. Configurar el servidor maestro

Acceda al servidor maestro:

```bash
docker exec -it mariadb_slave-client1-1 mysql -uroot -p${ROOT_PASSWORD}
```

Cree un usuario de replicación:

```sql
CREATE USER 'replica'@'%' IDENTIFIED BY 'vaveeneil6ohcaiXooGh';
GRANT REPLICATION SLAVE ON *.* TO 'replica'@'%';
FLUSH PRIVILEGES;
```

Verifique el estado del maestro para obtener la posición del registro binario:

```sql
SHOW MASTER STATUS;
```

Anote el valor de `File` (por ejemplo, `mysql-bin.000003`) y `Position` (por ejemplo, `154`).

### 2. Obtener la dirección IP del servidor maestro

Puede usar el hostname del contenedor o su dirección IP:

```bash
# Opción 1: Obtener la IP
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' mariadb_slave-client1-1

# Opción 2: Usar el hostname
docker inspect -f '{{.Config.Hostname}}' mariadb_slave-client1-1
```

### 3. Configurar los servidores esclavos

Para cada servidor esclavo (client2, client3, client4), realice los siguientes pasos:

Acceda al servidor esclavo:

```bash
docker exec -it mariadb_slave-client2-1 mysql -uroot -p${ROOT_PASSWORD}
```

Configure la replicación utilizando la IP o el hostname del maestro:

```sql
-- Usando IP (reemplace con la IP real del maestro)
CHANGE MASTER TO 
    MASTER_HOST='172.xx.xx.xx', 
    MASTER_USER='replica', 
    MASTER_PASSWORD='vaveeneil6ohcaiXooGh', 
    MASTER_LOG_FILE='mysql-bin.000003', -- Use el valor obtenido del maestro
    MASTER_LOG_POS=154;                 -- Use el valor obtenido del maestro

-- O usando hostname (reemplace con el hostname real)
CHANGE MASTER TO 
    MASTER_HOST='f530da13e128', 
    MASTER_USER='replica', 
    MASTER_PASSWORD='vaveeneil6ohcaiXooGh', 
    MASTER_LOG_FILE='mysql-bin.000003', -- Use el valor obtenido del maestro
    MASTER_LOG_POS=154;                 -- Use el valor obtenido del maestro
```

Inicie el esclavo:

```sql
START SLAVE;
```

Verifique el estado del esclavo:

```sql
SHOW SLAVE STATUS\G
```

Busque las siguientes líneas, que indican que la replicación está funcionando correctamente:

```
Slave_IO_Running: Yes
Slave_SQL_Running: Yes
```

Y eventualmente verá:

```
Slave has read all relay log; waiting for more updates
```

Repita estos pasos para cada servidor esclavo (client2, client3, client4).

## Verificación del sistema

### 1. Probar la replicación

En el servidor maestro, cree una tabla de prueba:

```sql
USE dbtest;
CREATE TABLE test_replication (id INT PRIMARY KEY, name VARCHAR(50));
INSERT INTO test_replication VALUES (1, 'Test Data');
```

En cada servidor esclavo, verifique que la tabla se ha replicado:

```sql
USE dbtest;
SELECT * FROM test_replication;
```

### 2. Verificar las configuraciones

Para verificar el ID del servidor en cada nodo:

```sql
SHOW VARIABLES LIKE 'server_id';
```

Para verificar el usuario de replicación:

```sql
SELECT user, host FROM mysql.user WHERE user = 'replica';
```

## Mantenimiento y solución de problemas

### Monitoreo de la replicación

Para verificar el estado de la replicación en cualquier momento:

```sql
SHOW SLAVE STATUS\G
```

### Solución de errores comunes

Si la replicación se detiene debido a un error, puede intentar lo siguiente:

1. **Saltar una transacción que está causando un error:**

```sql
STOP SLAVE;
SET GLOBAL SQL_SLAVE_SKIP_COUNTER = 1;
START SLAVE;
```

2. **Reiniciar la replicación desde cero:**

```sql
STOP SLAVE;
RESET SLAVE;
-- Vuelva a configurar el maestro con CHANGE MASTER TO
START SLAVE;
```

3. **Verificar los logs para más detalles:**

```bash
docker exec -it mariadb_slave-client2-1 cat /var/lib/mysql/error.log
```

### Prácticas recomendadas para producción

1. **Copias de seguridad**: Configure respaldos regulares:
   ```bash
   docker exec mariadb_slave-client1-1 mysqldump -u root -p${ROOT_PASSWORD} --all-databases > backup-$(date +%Y%m%d).sql
   ```

2. **Alta disponibilidad**: Considere implementar un servicio de failover automático.

3. **Monitorización**: Implemente una solución de monitoreo como Prometheus y Grafana.

4. **Balanceo de carga**: Considere añadir ProxySQL para distribuir las consultas entre esclavos.

5. **Seguridad**: Revise los archivos de configuración para reforzar la seguridad según sus necesidades.

## Escalamiento

Para añadir más esclavos:
1. Copie y adapte la configuración de un esclavo existente en el `docker-compose.yml`
2. Cree un nuevo archivo de configuración MariaDB
3. Asigne un ID de servidor único
4. Siga los pasos de configuración de esclavos

## Optimizaciones avanzadas (para sistemas de mayor carga)

- Ajuste el tamaño del buffer pool de InnoDB según la RAM disponible
- Configure la replicación semisíncrona para mayor fiabilidad
- Implemente sharding para bases de datos muy grandes
- Configure la replicación de grupo MariaDB para mayor tolerancia a fallos
