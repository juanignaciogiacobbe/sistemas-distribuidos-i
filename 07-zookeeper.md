---
aliases:
  - Zookeeper
recurso: "[[recursos/04-zookeeper.pdf|04-zookeeper]]"
---
# Zookeeper
- Sistema de coordinación distribuido -> La idea es encapsular la capa de coordinación entre apps en un servicio aparte.
	- Internamente usa un algoritmo de consenso llamado ZAB, que es muy similar a Raft (asumimos que es Raft)
- En realidad consiste en un kernel de coordinación, y su principal objetivo es poder hacer su API extendible a las necesidades de la app que lo utilice sin tener que cambiar el core.
- Hicieron que su API sea muy performance, dado que dejaron de lado las primitivas bloqueantes. -> es un sistema wait-free, y los data objects se guardan en una estructura parecida a un FS. La consistencia aquí es relajada.
- Las operaciones se ejecutan en FIFO a nivel de cliente (es decir, respeta el orden de llegada de operaciones, lo cual nos permite enviar muchísimas tasks de manera async).

## Objetivos NO funcionales
- Sistema dominado por lecturas (de mínima, el doble de lecturas que escrituras, y de máxima, 100 veces más lecturas que escrituras).
	- Se descarta la linealizabilidad -> Se permite la lectura a followers.
--- 
## Overview

![](img/Pasted%20image%2020260511191913.png)

- **Jerarquía de nombres:** Utiliza la notación estándar de UNIX para las rutas (ej. `/app1/p_1`). A diferencia de un sistema de archivos tradicional, los znodes no están pensados para almacenamiento de datos general, sino para **metadatos de coordinación**.
- **Tipos de Znodes:**
    - **Regulares:** Los clientes los crean y eliminan de forma explícita.
    - **Efímeros:** El sistema los elimina automáticamente cuando finaliza la sesión del cliente que los creó (ya sea por cierre voluntario o falla).
    - **Secuenciales:** Al crearlos, se les añade un contador que aumenta de forma monótona, garantizando que el nombre sea único y refleje el orden de creación.
- **Mecanismo de Watches:** No obliga a los clientes a consultar constantemente. Si hubo cambios, ZooKeeper permite configurar "watches". Estos son **disparadores de un solo uso** que envían una notificación al cliente cuando el znode observado cambia.
- **Modelo de datos y Metadatos:** Cada znode almacena metadata (timestamps, version, etc). Esto permite a los clientes rastrear cambios y ejecutar **actualizaciones condicionales** basadas en la versión del nodo.
- **Sesiones de Cliente:** Cuando un cliente se conecta, inicia una sesión con un _timeout_. Las sesiones son persistentes y permiten que un cliente se mueva de manera transparente de un servidor a otro dentro del conjunto de ZooKeeper sin perder su estado
--- 
## Operaciones 

### De Escritura

- `create`: Crea un nodo (**znode**) en una ruta con datos y banderas (como efímero o secuencial).
- `delete`: Elimina un nodo si la **versión** coincide con la del servidor.
- `setData`: Guarda datos en un nodo si la **versión** proporcionada es la correcta.

### De Lectura

- `exists`: Verifica si un nodo existe y permite activar un **watch**.
- `getData`: Devuelve el contenido y los metadatos (como la versión) del nodo.
- `getChildren`: Retorna una lista con los nombres de los nodos hijos.

### De Sincronización

- `sync`: Obliga al servidor a ponerse al día con las actualizaciones pendientes antes de procesar una lectura, garantizando ver el estado más reciente.
---
## Garantías
- Writes Linealizables: Todas las req que actualizan el estado de Zookeeper son serializables y respetan la predecencia.
- Orden FIFO en operaciones de un cliente.

## Modelo de Consistencia
1. Escrituras Linealizables
2. FIFO Client Order
	- Read your Writes
	- Monotonic Reads 

---
## Implementación

![](img/Pasted%20image%2020260511194644.png)

El sistema se organiza en tres capas principales que gestionan el flujo de las peticiones:
1. **Request Processor:** Prepara la solicitud para su ejecución.
2. **Atomic Broadcast:** Un protocolo de acuerdo (Zab) para coordinar las actualizaciones.
	- Con esto se asegura que todas las réplicas se mantengan coherentes.
3. **Replicated Database:** Una base de datos en memoria que contiene todo el árbol de datos (znodes) y que está presente en cada servidor.
	- Como el árbol vive en RAM, los znodes están pensados en pesar como máximo 1MB.
	- Permite respuestas rápidas para las consultas.

### Request Processor
Ayuda a mantener la consistencia entre réplicas:
- el líder transforma cada write en una **transacción idempotente**. Esto significa que capturan el estado final deseado, permitiendo que se apliquen de forma segura incluso si hay fallos.
- Cuando el líder recibe una escritura, calcula cuál será el estado del sistema una vez aplicada (nuevos números de versión, timestamp, etc.). Esto es necesario porque puede haber otras transacciones pendientes que aún no se han reflejado en la base de datos local.
	- La idea es que calcule los cambios teniendo en cuenta también las operaciones que ya estaban encoladas.
- Si una operación falla (por ejemplo, en un `setData` condicional donde la versión no coincide), el sistema genera un **errorTXN** en lugar de simplemente descartar el req, asegurando que todos los servidores registren el mismo flujo de eventos.



### Interacciones Cliente-Servidor

- **Lecturas Locales:** Las req de lectura se procesan **totalmente a nivel local** en el servidor donde el cliente está conectado. No hay actividad de disco ni protocolos de acuerdo involucrados, lo cual es la clave para un rendimiento excelente en aplicaciones donde predominan las lecturas.
- **Ordenamiento mediante ZXID:** Cada lectura se etiqueta con un **zxid**, que es el ID de la última transacción vista por ese servidor. Este identificador define el orden parcial de las lecturas con respecto a las escrituras.
- **Gestión de Watches:** Las notificaciones de los watches se manejan **localmente**. Solo el servidor al que está conectado el cliente rastrea y dispara sus notificaciones.
- **Lease Estricto:** Los servidores procesan las escrituras de forma secuencial y no permiten que otras lecturas o escrituras ocurran concurrentemente durante la aplicación de un cambio. Esto garantiza que las notificaciones de los watches lleguen en un orden estrictamente lógico y coherente.

---
## Ejemplos
### 1. Lock Distribuido con Zookeeper

```python
# Lock
n = create(l + "/lock", EPHEMERAL | SEQUENTIAL)
c = getChildren(l, false)
if n is lowest znode in c, exit
p = znode in c ordered just before n
if exists(p, true) wait for watch event
goto 2

# Unlock
delete(n)
```

1. A medida que llegan los que quieran el lock, crean un archivo en el FS de Zookeeper 
	- Es efímero porque una vez que se libera el lock, hay que liberar el archivo
	- Es secuencial porque la idea es que el lock se otorgue por orden de llegada
2. Estamos viendo en cada iteración si soy el znode hijo del archivo con número de secuencia más chiquito de todos los hijos
	- Si tengo el sequence más pequeño, quiere decir que soy el más longevo, por lo que tengo el lock del recurso
3. Si no tengo el lock, voy a fijarme si el anterior a mí que pidió el lock ya liberó el lock (borró el archivo) -> una vez que me notifican puedo tomar el lock del recurso.
4. Para liberar el lock, simplemente hay que eliminar el archivo efímero que creé al momento de pedir el lock.