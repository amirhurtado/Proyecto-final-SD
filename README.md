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
Para permitir que el nodo sea accesible desde la red de Incus y que pueda formar parte de un replica set, se modificó el archivo de configuración  `/etc/mongod.conf `:
Se cambió el parámetro bindIp de 127.0.0.1 a 0.0.0.0.
Se añadió la sección replication para definir el nombre del replica set:

```bash
replication: `
` replSetName: "rs-shard1"
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


