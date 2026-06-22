---
aliases:
  - Dynamo
recurso: "[[recursos/06-dynamo.pdf|06-dynamo]]"
---
# Dynamo
- Es una Base de Datos key-value altamente disponible.
	- Cada objeto es identificado por su clave.
- Populariza el teorema CAP (Consistencia, Availability, Partition Tolerance), y las DBs NoSQL.
- Objetivos:
	1. **Siempre aceptar writes** (les importaba más que la consistencia) -> Pone sus esfuerzos en ser más disponible que consistente. -> Admite Split Brains.
		- Su negocio demandaba estar siempre up, haciendo tradeoff con la consistencia: una pequeña caída podía significar la pérdida de millones.
	2. [Consistencia Eventual](06-linealizabilidad.md#Consistencia%20Eventual) -> No Linealizable.
		- La idea es tener la data disponible al toque, por lo que hicieron que las tareas de replicación corran asíncronamente.
	3.  Permite writes conflictivos (always-writable)
		- El hecho de siempre aceptar escrituras hace que aumente el quilombo de los conflictos de las lecturas.
		- Dynamo se optimiza para cumplir con SLAs medidos en el percentil 99.9 de la distribución de latencia, asegurando una experiencia de usuario consistente incluso bajo carga.
- Contras:
	1. No tiene SQL -> Menos versatilidad para queries.
	2. No tiene transacciones -> ~~ACID~~
		- La experiencia de Amazon es que aplicar estas propiedades tiende a degradar la availability.
- Ventajas:
	- Fácil de [Particionar](03-replicacion-sharding.md#Sharding%20/%20Partición). -> Cada shard se transforma en una "pequeña DB" que tiene los registros completos.

## Principios importantes
- **Escalabilidad incremental:** El sistema debe permitir añadir un host de almacenamiento a la vez con un impacto mínimo tanto para los operadores como para el sistema mismo.
- **Simetría:** No existen nodos maestros o líderes; **todos los nodos tienen las mismas responsabilidades** que sus pares. Esto simplifica enormemente el aprovisionamiento y mantenimiento.

# Arquitectura
### Operaciones
1. `GET (key)`: Encuentra el objeto (o una lista de versiones en conflicto) y su **contexto**
2. `PUT (key, context, object)`: Escribe el objeto en sus réplicas correspondientes.
	- **Contexto:** Metadata opaca (relojes vectoriales) que viaja con el dato y permite al sistema rastrear la **causalidad** y versiones de los objetos.
	- Tanto las claves como los valores son tratados como **blobs**, cuya ubicación en el sistema se determina mediante un hash MD5 de la key.

## Partitioning: Consistent Hashing

![](img/Pasted%20image%2020260511204800.png)

Dynamo usa **consistent hashing** para el particionamiento:
- El rango de salida de la función de hash se trata como un anillo.
- A cada nodo se le asigna un valor aleatorio en este espacio que representa su posición en el anillo.
- Para guardar un objeto, se aplica un hash a su clave para obtener una posición en el anillo. Luego, se recorre el anillo en **sentido horario** hasta encontrar el primer nodo con una posición mayor; ese nodo es el responsable de la clave.
- Al añadir o eliminar un nodo, solo se ven afectados sus vecinos inmediatos en el anillo, mientras que el resto de los nodos permanecen intactos.

### Adaptación para Dynamo
- **Nodos Virtuales (Tokens):** En lugar de que un nodo físico ocupe un solo punto en el anillo, se le asignan múltiples puntos llamados **"nodos virtuales"**.
    - Si un nodo falla, su carga se dispersa entre múltiples nodos disponibles.
    - Se puede asignar un **número distinto de nodos virtuales** a cada servidor según su capacidad física (heterogeneidad).

---
## Replicación
- Los datos se replican entre nodos
	- Cuando se asigna la posición en el anillo, el dato se replica en los n vecinos siguientes en el anillo (siendo $n=r-1$, con $r$ siendo el factor de replicación).
		- La Preference List tiene a los servidores sobre los que se va a replicar el dato.

### Control de orden de Writes de Claves
- Tiene que ser capaz de reconstruir la precedencia de cada operación
	- Mantener un orden de operaciones ("Log").
- Happens Before: encontrar una causalidad entre eventos
	1. Normalmente el envío se realiza antes que el recibimiento del mensaje
	2. En una misma máquina es sencillo encontrar la traza de las operaciones, ya que están marcadas por el clock de la misma
	3. Podemos también encontrar una traza y reconstruir la historia de eventos a partir de ciertas transitividades
		- Hay casos en los cuales no podemos conectar dos eventos(no hay causalidad), y no podemos definir quién ocurrió primero (son operaciones concurrentes).

### Relojes Lógicos (o de Lamport)
1. Los eventos locales incrementan en 1 el reloj lógico
2. Al enviar un mensaje, en el mismo se agrega el valor del reloj lógico del emisor.
	- El receptor actualiza su clock haciendo `MAX(Cq + 1, Tm + 1)`, siendo `Cq` el clock del receptor, y `Tm` el clock que recibe del mensaje.


> [!NOTE] Orden Total VS Orden Parcial
>> [!WARNING] Orden Total
> > Todos los elementos del conjunto se pueden comparar con todos
> 
>> [!WARNING] Orden Parcial
> > Solo algunos pares del conjunto son comparables
>> - Ejemplo: Happens Before es parcial porque hay eventos que no los podemos comparar (cuando pasan en paralelo y no tienen causalidad que los junte).

### Ejemplo: ChatRoom

![](img/Pasted%20image%2020260618205545.png)

## Vector Clocks

![](img/Pasted%20image%2020260618205725.png)
![](img/Pasted%20image%2020260618205942.png)
- Cada vez que hagamos una escritura a Dynamo, a la clave nueva se le va a crear también un reloj lógico, y con esto es capaz de ordenar los eventos.
	- Al hacer un `UPDATE`, se hace un `GET` de la clave, si existe, se devuelve el clock asociado a esa clave -> Al updatear se manda el valor nuevo, y también el mismo clock (esto desencadena el proceso de la escritura).
- Al tener un split brain por ejemplo, puede pasar que empecemos a tener operaciones concurrentes en varios nodos
	- Con los relojes lógicos, se aceptan los eventos siguiendo las reglas de arriba
	- Al realizar un `GET`, Dynamo devuelve varias  versiones que tiene del objeto, y que se hicieron de forma concurrente (lo podemos inferir a partir de los relojes vectoriales).
		- Entonces la responsabilidad de mergear/arreglar los "conflictos" queda del lado del cliente (se fixea a partir de la lógica de negocio que tenga).

---
## Sloppy Quorum
- Recibimos una escritura, y el cliente elige a uno de los nodos de la Preference List como "coordinator" -> este se encarga de pasarle los datos a los demás de la lista
- Al momento de recibir una escritura, puede pasar que alguno de los nodos del Preference List esté caído o no responda
	- En tal caso, se toma otro nodo contiguo dentro del anillo del consistent hashing y se guarda ahí el dato (por eso es sloppy).
		- Ese nodo elegido va a tener la referencia al nodo que en realidad debió ir ese packet en un principio, por lo que cuando este vuelva a funcionar, se le va a forwardear el packet original. -> Se parte de la hipótesis de que "la mayoría de fallas son temporales".

## Sincronización Directa
- Se usan Merkle Trees para comparar tablas entre sí (ejemplo: réplicas de tablas que guardan distintos nodos)
	- A cada row se le calcula un Hash, y luego se van juntando de a pares de hashes
	- Para ver las diferencias, se va recorriendo y comparando los hashes de distintos niveles del árbol.

![](img/Pasted%20image%2020260618214316.png)