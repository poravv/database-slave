# MariaDB con Replicación Master-Slave Optimizado para Producción

Este proyecto configura un sistema de replicación de MariaDB con un nodo maestro (client1) y tres nodos esclavos (client2, client3, client4), optimizado para entornos productivos con alta disponibilidad y rendimiento.

## Requisitos previos

- Docker Engine v20.10+ y Docker Compose v2.x+
- Al menos 4GB de RAM disponible para todos los contenedores
- Espacio en disco suficiente para almacenar las bases de datos y sus logs

## Estructura del proyecto

```
├── docker-compose.yml      # Configuración de contenedores optimizada
├── Dockerfile.client1      # Configuración del servidor maestro
├── Dockerfile.client2      # Configuración del primer esclavo
├── Dockerfile.client3      # Configuración del segundo esclavo
├── Dockerfile.client4      # Configuración del tercer esclavo
├── init.sql               # Script SQL inicial para todos los nodos
├── my1.cnf                # Configuración optimizada del servidor maestro
├── my2.cnf                # Configuración optimizada del primer esclavo
├── my3.cnf                # Configuración optimizada del segundo esclavo
├── my4.cnf                # Configuración optimizada del tercer esclavo
└── .env                   # Archivo de variables de entorno
```

## Configuración inicial

### 1. Configurar variables de entorno

Crea un archivo `.env` en el directorio raíz con el siguiente contenido:

```
ROOT_PASSWORD=ContraseñaRootSegura!
DB_USER=usuario_app
CLIENT1_PASSWORD=ContraseñaSeguraMaster!
CLIENT2_PASSWORD=ContraseñaSeguraEsclavo1!
CLIENT3_PASSWORD=ContraseñaSeguraEsclavo2!
CLIENT4_PASSWORD=ContraseñaSeguraEsclavo3!
```

> **IMPORTANTE:** Usa contraseñas seguras diferentes a las de este ejemplo en tu entorno de producción.

### 2. Iniciar los contenedores

```bash
docker compose up -d
```

Este comando levantará todos los contenedores con las configuraciones optimizadas. El sistema incluye:
- Comprobaciones de salud (healthchecks)
- Dependencias entre contenedores (los esclavos esperan a que el maestro esté listo)
- Configuraciones de rendimiento optimizadas
- Aliases de red para facilitar la conexión

## Configuración de la replicación

### 1. Acceder al servidor maestro (client1)

```bash
docker exec -it mariadb_slave-client1-1 mysql -uroot -p
```

> Ingresa la contraseña ROOT_PASSWORD que configuraste en el archivo .env

### 2. Crear usuario de replicación en el maestro

```sql
CREATE USER 'replica'@'%' IDENTIFIED BY 'ContraseñaReplicaSegura!';
GRANT REPLICATION SLAVE ON *.* TO 'replica'@'%';
FLUSH PRIVILEGES;
```

### 3. Obtener información de estado del maestro

```sql
SHOW MASTER STATUS;
```

> **IMPORTANTE:** Anota los valores de `File` y `Position` mostrados. Los necesitarás para configurar los esclavos.
> Ejemplo: File = mysql-bin.000003, Position = 1234

### 4. Configurar cada servidor esclavo

Para cada servidor esclavo (client2, client3, client4), ejecuta los siguientes pasos:

a. Acceder al servidor esclavo:
```bash
docker exec -it mariadb_slave-client2-1 mysql -uroot -p
# Para los demás nodos, cambiar client2 por client3 o client4
```

b. Configurar la replicación usando el nombre del host (recomendado):
```sql
CHANGE MASTER TO 
    MASTER_HOST='master', 
    MASTER_USER='replica', 
    MASTER_PASSWORD='ContraseñaReplicaSegura!', 
    MASTER_LOG_FILE='mysql-bin.000XXX', 
    MASTER_LOG_POS=YYYY;
```

> Reemplaza 'mysql-bin.000XXX' y YYYY con los valores exactos que obtuviste del comando SHOW MASTER STATUS.

c. Iniciar el proceso de replicación:
```sql
START SLAVE;
```

d. Verificar el estado de la replicación:
```sql
SHOW SLAVE STATUS\G
```

El sistema está configurado correctamente si ves:
- `Slave_IO_Running: Yes`
- `Slave_SQL_Running: Yes`
- `Seconds_Behind_Master: 0` (o un valor pequeño)

## Verificación del funcionamiento

### 1. Prueba de escritura en el maestro

Conéctate al maestro y crea una tabla de prueba:

```sql
USE dbtest;
CREATE TABLE prueba (id INT AUTO_INCREMENT PRIMARY KEY, texto VARCHAR(50));
INSERT INTO prueba (texto) VALUES ('Prueba de replicación');
SELECT * FROM prueba;
```

### 2. Verificación en los esclavos

Conéctate a cualquier esclavo y verifica que los datos se han replicado:

```sql
USE dbtest;
SELECT * FROM prueba;
```

Deberías ver exactamente los mismos datos que en el maestro.

## Resolución de problemas

### Error en la replicación

Si encuentras errores en la replicación, puedes verificar y solucionar de la siguiente manera:

```sql
-- En el esclavo con problemas:
SHOW SLAVE STATUS\G

-- Si necesitas saltar un error específico:
STOP SLAVE;
SET GLOBAL SQL_SLAVE_SKIP_COUNTER = 1;
START SLAVE;
```

### Verificar configuración de server-id

Para asegurarte de que no hay duplicación de server-id:

```sql
SHOW VARIABLES LIKE 'server_id';
```

Cada servidor debe tener un ID único:
- client1 (maestro): 1
- client2 (esclavo): 2
- client3 (esclavo): 3
- client4 (esclavo): 4

## Monitoreo del sistema

Para monitorear el estado del sistema de replicación:

```bash
# Ver logs de un contenedor específico
docker logs mariadb_slave-client1-1

# Ver estado de los contenedores
docker compose ps

# Entrar a un contenedor para diagnóstico
docker exec -it mariadb_slave-client1-1 bash
```

## Consideraciones para producción

1. **Respaldos**: Configura respaldos regulares del maestro usando mysqldump o herramientas como Percona XtraBackup.
2. **Monitoreo**: Implementa herramientas como Prometheus + Grafana para monitorear el rendimiento.
3. **Escalabilidad**: Para mayor escalabilidad, considera implementar un proxy como ProxySQL.
4. **Alta disponibilidad**: Para mayor disponibilidad, configura un sistema de conmutación automática por error (failover).

## Arquitectura del sistema

```
client1 (MAESTRO)
    ↓
    ├─────→ client2 (ESCLAVO)
    ├─────→ client3 (ESCLAVO)
    └─────→ client4 (ESCLAVO)
```

Todos los cambios deben realizarse en el nodo maestro (client1) y se replicarán automáticamente a todos los esclavos.
