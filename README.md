# Proyecto Final - Sistemas Distribuidos (SD)

---

##  Resumen de Nuestro Stack Tecnológico

* **Servidor Web (Contenedor 1):** Next.js (con React y TypeScript).
* **Servidor de Autenticación (Contenedor 4):** Node.js con Express.
* **Bases de Datos (Contenedores 2, 3, 5 y réplicas):** MongoDB (con Sharding y Replica Sets).
* **Gestor Web Incus (Contenedor 6):** Incus UI Canonical.
* **Orquestación de Contenedores:** Incus.

---

##  Plan de Acción 

### Fase 1: Configuración del Entorno Incus
* Instalar Incus en la máquina anfitriona.
* Crear y configurar el contenedor para el gestor web (**Incus UI**) para tener una visión gráfica de nuestro progreso.
* Verificar la configuración de la red puente por defecto (`incusbr0`).

### Fase 2: Arquitectura del Clúster de Sharding (MongoDB)
* **Shards (Almacenes):**a
    * Crear los contenedores para el **Shard 1 (Base de Datos 1)** e inicializar su conjunto de réplicas (mínimo 2 contenedores: primario y réplica).
    * Crear los contenedores para el **Shard 2 (Base de Datos 2)** e inicializar su conjunto de réplicas (mínimo 2 contenedores).
* **Config Servers (Mapa):**
    * Crear el contenedor (o contenedores) para los **Servidores de Configuración** (Config Server Replica Set).
* **Router (Cerebro):**
    * Crear el contenedor para el **Router (`mongos`)**. Este será el *único* punto de entrada para la aplicación web.
* **Inicialización:**
    * Conectarse al `mongos` para inicializar el clúster, añadiendo los servidores de configuración y, finalmente, los *shards* (Shard 1 y Shard 2).
* **Habilitación:**
    * Habilitar la fragmentación en la base de datos y la colección de "productos", especificando nuestra *shard key* (`category`).

### Fase 3: Implementación de la Autenticación
* Crear el contenedor para la base de datos de autenticación (**DB3**), configurada también como un conjunto de réplicas de MongoDB para alta disponibilidad.
* Desarrollar el microservicio de autenticación con Node.js/Express (Contenedor 4) y conectarlo a la DB3.

### Fase 4: Desarrollo de la Aplicación Principal
* Crear el contenedor para la aplicación **Next.js** (Contenedor 1).
* Desarrollar el dashboard y el CRUD de productos.
* Integrar la aplicación con:
    1.  El servicio de autenticación (ej. `http://auth-server.incus`).
    2.  El clúster fragmentado (conectándose *únicamente* al router, ej. `mongodb://mongos-router.incus`).

### Fase 5: Pruebas y Documentación
* Realizar pruebas integrales:
    * **Funcionales:** CRUD de productos, login, registro.
    * **De Resiliencia:** Simular un fallo del nodo primario en un *replica set* y verificar que el sistema sigue operando.
* Construir el documento detallado (paso a paso) del proyecto.

---

##  Decisiones Clave de Arquitectura

### 1. Estrategia de Fragmentación
* **Elección:** Fragmentación Horizontal.
* **Justificación:** El *Sharding* nativo de MongoDB es una implementación de fragmentación horizontal. Elegimos esta estrategia porque el objetivo es escalar horizontalmente el volumen de datos, es decir, poder almacenar un número virtualmente ilimitado de productos distribuyéndolos entre múltiples servidores (nuestros *shards*). La fragmentación vertical (dividir las columnas) no es el requisito principal aquí.

### 2. Estrategia de Red en Incus
* **Elección:** Red puente (bridge) gestionada por Incus (`incusbr0`) con DNS interno.
* **Justificación:** Todos nuestros contenedores se conectarán a esta red. Incus proporciona un DNS interno automático, permitiendo que los contenedores se comuniquen usando sus nombres (ej. `auth-server.incus` o `mongos-router.incus`) en lugar de IPs estáticas. Esto es más robusto y fácil de mantener.

### 3. Clave de Fragmentación (Shard Key)
* **Elección:** El campo `category`.
* **Justificación:** El enunciado del proyecto sugiere dividir por categoría. Usar `category` como *Shard Key* implementa esta lógica directamente. Los productos con la misma categoría se agruparán en el mismo *shard*, lo que puede ser muy eficiente si las consultas suelen filtrar por este campo. Se analizarán las implicaciones de esta elección en la documentación final.



###  Desarrollo de fases

## Fase 1: Configuración del Entorno Incus

Se configuró el entorno de virtualización usando **WSL 2** en Windows para disponer de un sistema Linux completo.  
Dentro de este entorno se instaló y configuró **Incus**, que será el orquestador de los contenedores del proyecto.

### Pasos realizados

- Se habilitó **WSL 2** y se instaló **Ubuntu** como distribución base.  
- Se instaló **Incus** mediante el gestor de paquetes `apt`.  
- Se ejecutó `incus admin init` para la configuración inicial:
- Se verificó que el servicio `incus` estuviera activo y corriendo.
- Se comprobó la instalación con `incus list`, confirmando un entorno limpio y funcional.
- Se creó un contenedor (`incus-ui`) para alojar el gestor web `incus-ui-canonical`.
- Se configuró el contenedor añadiendo los repositorios necesarios e instalando la UI mediante `apt`.
- Se confirmó que el servicio `incusd` está activo y escuchando en el puerto **8443**.  
- Prueba con `curl -k https://<IP_WSL>:8443` devolvió:
  ```json
  {"type":"sync","status":"Success","status_code":200,"operation":"","error_code":0,"error":"","metadata":["/1.0"]}
- Al acceder desde el navegador a https://<IP_WSL>:8443, se visualizó correctamente la interfaz gráfica Incus UI Canonical con la pantalla de login “Login with TLS”.
- Se genero y habilitó el certificado 





## Fase 2: Arquitectura del Clúster de Sharding (MongoDB)

El objetivo de esta fase es la construcción de la infraestructura de base de datos distribuida. Utilizaremos **MongoDB Sharding** sobre contenedores Incus para garantizar escalabilidad horizontal y alta disponibilidad para la gestión de productos.

### 2.1 Creación de Contenedores para el Shard 1 (Base de Datos 1)

El primer paso es crear los contenedores que alojarán el primer *shard* (fragmento) de nuestra base de datos, incluyendo su réplica para garantizar la disponibilidad y tolerancia a fallos. Este conjunto formará nuestro primer *Replica Set*.

#### 2.1.1 Diagnóstico de Imagen del Sistema Operativo

Se implementó la siguiente estrategia de diagnóstico:
1.  Se utilizó el comando `incus image list images: | grep "ubuntu"` para consultar el catálogo del repositorio de imágenes por defecto (`images:`).
2.  Esto reveló que el alias correcto para la versión de Ubuntu 22.04 es **`ubuntu/jammy`**.

#### 2.1.2 Lanzamiento de Contenedores

Con el alias verificado, se procedió a lanzar los dos contenedores que formarán el **Shard 1**.

* **Creación del contenedor principal para el Shard 1:**
    ```bash
    incus launch images:ubuntu/jammy shard1-primary
    ```

* **Creación del contenedor para la réplica del Shard 1:**
    ```bash
    incus launch images:ubuntu/jammy shard1-replica
    ```

#### 2.1.3 Verificación Inicial de Contenedores

Se verificó la correcta creación y ejecución de los contenedores con `incus list`, confirmando que todos están en estado `RUNNING` y han recibido IPs internas en la red `incusbr0`.

```code
+----------------+---------+----------------------+------------------------------------------------+-----------+-----------+
|      NAME      |  STATE  |         IPV4         |                      IPV6                      |   TYPE    | SNAPSHOTS |
+----------------+---------+----------------------+------------------------------------------------+-----------+-----------+
| incus-ui       | RUNNING | 10.138.89.109 (eth0) | fd42:ef06:f622:9a08:1266:6aff:fe0c:28fe (eth0) | CONTAINER | 0         |
| shard1-primary | RUNNING | 10.138.89.122 (eth0) | fd42:ef06:f622:9a08:1266:6aff:fe5f:937e (eth0) | CONTAINER | 0         |
| shard1-replica | RUNNING | 10.138.89.241 (eth0) | fd42:ef06:f622:9a08:1266:6aff:fe41:9c80 (eth0) | CONTAINER | 0         |
+----------------+---------+----------------------+------------------------------------------------+-----------+-----------+
```

## 2.2 Configuración de MongoDB en el Shard 1

### 2.2.1 Instalación en el Nodo Primario (shard1-primary)

Con los contenedores en funcionamiento, el siguiente paso es instalar y configurar MongoDB. El proceso se inició en el nodo designado como primario.

**Acceso al contenedor:**  
Se accedió a la terminal del contenedor con el comando:

```bash
incus exec shard1-primary -- bash
```


Instalación de MongoDB:
Dentro del contenedor, se ejecutaron los comandos estándar para añadir el repositorio oficial de MongoDB 7.0 para Ubuntu 22.04 (Jammy) e instalar los paquetes.

Configuración de Red y Replicación:
Para permitir que el nodo sea accesible desde la red de Incus y que pueda formar parte de un replica set, se modificó el archivo de configuración  `/etc/mongod.conf`:
Se cambió el parámetro bindIp de 127.0.0.1 a 0.0.0.0.
Se añadió la sección replication para definir el nombre del replica set:

```bash
replication: 
   replSetName: "rs-shard1"
```

Arranque del Servicio:
Finalmente, se reinició y habilitó el servicio mongod para aplicar la nueva configuración.

```bash
systemctl restart mongod
systemctl enable mongod
systemctl status mongod
```


La correcta ejecución del servicio fue verificada con systemctl status mongod, confirmando que el servicio estaba active (running).



### 2.2.2 Instalación en el Nodo de Réplica (shard1-replica)

Se repitió el mismo proceso de instalación y configuración en el contenedor shard1-replica.
Se aseguró que la configuración en /etc/mongod.conf fuera idéntica, especialmente manteniendo el mismo nombre de replSetName: "rs-shard1", para que ambos nodos puedan pertenecer al mismo conjunto.






### 2.2.3 Inicialización del Replica Set (rs-shard1)
Con ambos nodos configurados, el paso final fue unirlos formalmente en un conjunto de réplicas.
Se accedió a la consola de MongoDB (mongosh) desde el nodo primario (shard1-primary).
Se ejecutó el comando rs.initiate() especificando los dos miembros del conjunto mediante sus direcciones IP internas:


```bash
rs.initiate({
  _id: "rs-shard1",
  members: [
    { _id: 0, host: "10.138.89.122:27017" },
    { _id: 1, host: "10.138.89.241:27017" }
  ]
})
```

El éxito de la operación se confirmó de dos maneras:
El prompt de la consola cambió a rs-shard1 [primary]>, indicando que el nodo había asumido su rol principal.
El comando rs.status() mostró a ambos miembros en la lista, uno como PRIMARY y el otro como SECONDARY.
Con estos pasos, el Shard 1 queda completamente configurado como un clúster de base de datos con alta disponibilidad.


### 2.3 Creación y Configuración del Shard 2 (rs-shard2)
Siguiendo el mismo procedimiento exitoso del Shard 1, se procedió a construir el segundo fragmento de la base de datos.





### 2.4 Creación de Componentes Centrales del Clúster

Con los shards de datos ya configurados, se procedió a crear los componentes que gestionan y dirigen el clúster.

#### 2.4.1 Configuración de los Servidores de Configuración (Config Servers)

Para almacenar los metadatos del clúster se crearon tres contenedores: configsvr1, configsvr2 y configsvr3.
Se instaló MongoDB en cada uno y se modificó su configuración (/etc/mongod.conf) para asignarles el rol específico de configsvr dentro del clúster de sharding.
Además, se agruparon en su propio replica set llamado rs-config para garantizar la alta disponibilidad de los metadatos.

La inicialización se realizó desde configsvr1 con el siguiente comando:


```bash
rs.initiate({
  _id: "rs-config",
  configsvr: true,
  members: [
    { _id: 0, host: "configsvr1:27017" },
    { _id: 1, host: "configsvr2:27017" },
    { _id: 2, host: "configsvr3:27017" }
  ]
})
```


El éxito se verificó observando el cambio del prompt a:

rs-config [primary]>


lo cual indica que el replica set de configuración quedó correctamente inicializado.

#### 2.4.2 Despliegue del Router (mongos)

El componente final, que actuará como la única puerta de entrada para la aplicación, es el Router (mongos).

Contenedor:
Se lanzó un nuevo contenedor llamado mongos-router.

Instalación:
Se instaló el paquete mongodb-org, que incluye el binario mongos:

`apt update && apt install -y mongodb-org`


Configuración:
A diferencia de los nodos de datos, se creó un archivo de configuración específico en:

`/etc/mongos.conf`


Este archivo define la ubicación del replica set de los Config Servers y habilita la red externa:

```bash
sharding:
  configDB: rs-config/configsvr1:27017,configsvr2:27017,configsvr3:27017

net:
  bindIp: 0.0.0.0
  port: 27017
```

Servicio:
Se creó un servicio de systemd personalizado para gestionar el proceso mongos en:

`/etc/systemd/system/mongos.service`


con el siguiente contenido:

```bash
[Unit]
Description=MongoDB Router (mongos)
After=network.target

[Service]
User=mongodb
ExecStart=/usr/bin/mongos --config /etc/mongos.conf
Restart=always

[Install]
WantedBy=multi-user.target
```


Lanzamiento:
El servicio se inició y habilitó con:

```bash
systemctl start mongos
systemctl enable mongos
```


La correcta ejecución se verificó con:

`systemctl status mongos`

confirmando un estado active (running).




### 2.5 Ensamblaje y Verificación Final del Clúster
Con todos los componentes individuales (shards, config servers y router) en línea, el paso final fue ensamblarlos en un único sistema coherente.
#### 2.5.1 Detección y Corrección de Error de Configuración
La conexión inicial se realizó desde la máquina anfitriona hacia el router mongos. Sin embargo, al ejecutar el comando sh.addShard(...), el sistema arrojó un error crítico:


```bash
MongoServerError[Location50876]: Cannot run addShard on a node started without --shardsvr
```

El diagnóstico fue claro: los nodos de los *shards* (`shard1-primary`, `shard1-replica`, etc.) no habían sido configurados para identificarse a sí mismos como servidores de fragmentación.

La solución consistió en modificar el archivo `/etc/mongod.conf` en los **cuatro** contenedores de los shards para añadir la directiva de rol de clúster:

```yaml
sharding:
  clusterRole: shardsvr
```

Tras añadir esta configuración y reiniciar el servicio mongod en cada uno de los cuatro nodos, el problema quedó resuelto.


#### 2.5.2 Adición de Shards al Clúster


Con la corrección aplicada, se volvió a establecer la conexión con el router mongos y se ejecutaron con éxito los comandos para registrar cada replica set como un shard del clúster:

```bash
sh.addShard("rs-shard1/10.138.89.122:27017")
sh.addShard("rs-shard2/10.138.89.246:27017")
```

La operación fue exitosa, devolviendo el mensaje { "ok": 1 }, confirmando que los shards fueron añadidos correctamente.




### 2.5.3 Habilitación de Sharding en la Base de Datos
Con el clúster completamente ensamblado, el último paso de la configuración fue instruir al sistema sobre cómo distribuir los datos. Se eligió la estrategia de fragmentación horizontal basada en el campo category, tal como se definió en las decisiones de arquitectura.
Desde la consola mongosh conectada al router, se ejecutaron los siguientes comandos:
1. Habilitar Sharding en la Base de Datos: Se designó una nueva base de datos, productsDB, como apta para la fragmentación.

```bash
sh.enableSharding("productsDB")
```

2. Definir la Shard Key: Se especificó que la colección products dentro de productsDB debía ser fragmentada utilizando el campo category como Shard Key.
```bash
sh.shardCollection("productsDB.products", { category: 1 })
```

La confirmación final se obtuvo mediante sh.status(), que mostró la base de datos productsDB con su colección products debidamente configurada para el sharding.

Con este paso, la Fase 2 concluye exitosamente. La infraestructura de la base de datos distribuida está completamente desplegada, configurada y lista para recibir datos.



## Fase 3: Implementación del Sistema de Autenticación
### 3.1 Despliegue de la Base de Datos de Autenticación
La primera etapa de la Fase 3 consistió en construir la base de datos que almacenará la información de los usuarios (credenciales, perfiles, etc.). Para mantener la robustez y alta disponibilidad del sistema, se optó por una arquitectura de replica set también para este componente.
Siguiendo un procedimiento ya establecido, se crearon dos nuevos contenedores: auth-db-primary y auth-db-replica. El proceso de instalación y configuración de MongoDB fue idéntico al de los nodos de shard. La única diferencia clave en la configuración fue la asignación de un nombre de replica set único y descriptivo en el archivo `/etc/mongod.conf`:
```bash
replication:
  replSetName: "rs-auth"
```
Finalmente, se inicializó el conjunto desde el nodo primario. El cambio del prompt de la consola a rs-auth [primary]> confirmó el éxito de la operación. Con esto, se dispone de una base de datos de autenticación independiente, replicada y lista para ser consumida por el microservicio de autenticación.




### 3.2 Implementación del Microservicio de Autenticación
Con la base de datos de usuarios operativa, el siguiente paso fue desarrollar el microservicio encargado de gestionar la lógica de autenticación y registro.
Creación del Contenedor: Se lanzó un nuevo contenedor, auth-server, destinado a alojar la aplicación de Node.js.
Configuración del Entorno: Dentro del contenedor, se configuró el entorno de ejecución necesario:

- Se instaló Node.js (versión 20.x) y su gestor de paquetes NPM utilizando el repositorio oficial de NodeSource.
- Se creó una carpeta para el proyecto y se inicializó con npm init -y.
- Se instalaron las dependencias clave para el servicio: express para el servidor, mongoose para la conexión a la base de datos, bcryptjs para el hashing de contraseñas, jsonwebtoken para la gestión de sesiones y cors para la comunicación entre servicios.
- Desarrollo del Servidor Inicial: Se creó un archivo index.js con la estructura básica de un servidor Express. El punto más importante de esta configuración inicial fue la cadena de conexión a la base de datos:

```bash
const MONGO_URI = 'mongodb://<IP_PRIMARIO_AUTH>:<PORT>,<IP_REPLICA_AUTH>:<PORT>/usersDB?replicaSet=rs-auth';
```

Esta URI se conecta directamente al replica set rs-auth, proporcionando resiliencia automática. Si el nodo primario de la base de datos falla, mongoose gestionará la conexión con el nuevo primario sin necesidad de intervención manual.

Prueba de Conectividad: Se ejecutó la aplicación con node index.js. La aparición de los mensajes Servidor corriendo en el puerto 4000 y ¡Conexión a MongoDB exitosa! en la consola confirmó que el microservicio se inició correctamente y estableció una conexión exitosa con el clúster de la base de datos de autenticación.



### 3.2.2 Implementación de la Lógica de Login y Emisión de Tokens (JWT)
Con el registro de usuarios ya implementado, el paso final fue desarrollar la funcionalidad de inicio de sesión.
Se añadió un nuevo endpoint POST en la ruta /login. La lógica implementada sigue las mejores prácticas de seguridad y autenticación moderna:

- Recepción de Credenciales: El endpoint recibe el email y la password del usuario.
- Verificación de Usuario: Busca en la base de datos un usuario que coincida con el email proporcionado. Si no lo encuentra, devuelve un error genérico de "Credenciales inválidas" para no revelar si un email está registrado o no.
- Comparación de Contraseña: Si el usuario existe, utiliza bcrypt.compare() para comparar de forma segura la password recibida con el hash almacenado en la base de datos.
- Generación de JWT: Si las credenciales son correctas, se genera un JSON Web Token (JWT). Este token contiene información del usuario (su ID) y está "firmado" digitalmente con una clave secreta. El token se devuelve al cliente con un tiempo de expiración (ej. 1 hora), sirviendo como un "pase" temporal que el cliente puede usar para autenticarse en futuras peticiones a rutas protegidas.

Al reiniciar la aplicación, el microservicio de autenticación quedó funcionalmente completo, proporcionando los mecanismos esenciales de registro, inicio de sesión y gestión de sesiones basadas en tokens. Con esto, concluye la Fase 3.


## Fase 4: Desarrollo de la Aplicación Principal con Next.js
### 4.1 Creación y Configuración del Entorno Frontend
El desarrollo del frontend comenzó con la preparación del contenedor que alojará la aplicación web.
- Lanzamiento del Contenedor: Se creó un nuevo contenedor llamado web-app para este propósito.
- Instalación del Entorno: Al igual que con el microservicio de autenticación, se instaló Node.js (versión 20.x) y NPM dentro del contenedor.
- Generación del Proyecto Next.js: Se utilizó la herramienta create-next-app para generar un nuevo proyecto con una configuración moderna y robusta. Se tomó la decisión de incluir:
- TypeScript: para un tipado estático y un código más seguro.
- Tailwind CSS: para un estilizado rápido y eficiente basado en utilidades.
- App Router: el sistema de enrutamiento recomendado por Next.js para arquitecturas modernas.
- Directorio src/: para una mejor organización del código fuente.