---
aliases:
  - Raft
---
# Raft
- El objetivo es que todas las máquinas se pongan de acuerdo en el orden de las operaciones que van llegando, y que el estado sea consistente.
	- Tiene consistencia fuerte -> [Linealizabilidad](06-linealizabilidad.md)
	- Mediante replicación, no se tiene más un Single Point of Failure como en [MapReduce](02-map-reduce.md) o [Google File System](04-google-file-system.md).
- Si una LogEntry está en una mayoría de lugares, se considera committeada, y no se puede borrar nunca más del log.


> [!NOTE] Mayoría
> Es la mitad +1 de los nodos.
> ![](img/Pasted%20image%2020260512200549.png)


> [!IMPORTANT] Nomenclatura
> N es el número de nodos del sistema (es un parámetro del sistema).
> M es la mayoría de los nodos 
> - Asumimos que N es fijo.
> - $M = (N/2) + 1$


> [!WARNING] Propiedad de los Quorums
> Dos quorums(mayorías) tienen al menos un elemento en común
> > [!DANGER] Corolario
> >  Toda mayoría tiene un servidor actualizado si todas las escrituras fueron en mayoría.


$W = R = M$ (Y $N$ impar)
1. Siempre nos va a quedar una partición con mayorías
2. La partición mayoritaria tiene al menos un nodo actualizado.

## Tolerancia a Fallas
- Se tolera cuando falla una minoría de nodos.
	- La regla sencilla para ver hasta cuántas fallas me banco es $F = N - M$.

Al momento de aceptar una operación de un cliente:
- La operación está persistida en una mayoría,
- y está aplicada en el líder.

## Elección de Líder
- Cada *term* (identificados con un número) tiene un líder, y cada vez que se elije uno nuevo, se inicia un nuevo term y se aumenta el número de term actual.

![](img/Pasted%20image%2020260512210453.png)

![](img/Pasted%20image%2020260512210529.png)

- El líder constantemente está enviando AppendEntries (AE) a los followers, y sirven a modo de HeartBeat
	- Si se pasa un timeout de elección (election timeout) sin recibir este AE, un follower está en condiciones de arrancar una nueva elección (va a pasar a ser un candidato).
		- Se vota a sí mismo, y busca mediante RequestVotes (RV) votos de parte de los demás -> Si obtengo una mayoría de votos, voy a ser nuevo líder.
		- Al arrancar una nueva votación, manda el RequestVote con el term actualizado.

### Respuesta a RequestVotes
- Solo se puede realizar 1 voto por term.
- Solo votar candidatos actualizados
	- El Log (el term) del candidato es más nuevo que el propio (tiene que estar actualizado).


> [!DANGER] Log Matching Property
> Si en dos servers se tienen el mismo LogIndex y el mismo LogTerm, se tiene la garantía de que el dato guardado ahí es el mismo.


#### Ejemplos

| Node ↓ / LogEntryID → | 1   | 2   |
| --------------------- | --- | --- |
| S1                    | 3   |     |
| S2                    | 3   | 3   |
| S3                    | 3   | 3   |

- En este caso, el S1 NO puede ser elegido como líder, ya que no tiene en su log la última entrada que fue committeada (que S2 y S3 por su parte SI la tienen, que es la que se encuentra en el log entry con ID 2).
	- Dato de color, S2 y S3 SI pueden ser elegidos como líder.
- Las celdas son los terms.



| Node ↓ / LogEntryID → | 10  | 11  | 12  | 13  |
| --------------------- | --- | --- | --- | --- |
| S1                    | 3   |     |     |     |
| S2                    | 3   | 3   | 4   |     |
| S3                    | 3   | 3   | 5   | 6   |

- Supongamos que la situación inicial es lo que tenemos en la tabla, y que S3 es el actual líder. 
	- Una optimización de Raft es que se usa el AppendEntries para actualizar el log de los followers.
1. AE que le manda S3 a S2:
	- `Entry = [6], PrevLogIndex = 12, PrevLogTerm = 5`
	- Siguiendo la Log Matching Property, el que recibe el packet (S2), se debe fijar si la anterior entrada es igual a lo que manda el S3 (`PrevLogIndex (S2) == PrevLogIndex (del packet) y PrevLogTerm (S2) == PrevLogTerm (del packet)`)
		- Si no se cumple, se devuelve un failed -> **Le faltan datos**.
2. AE que manda S3 a S2:
	- `Entry = [5, 6], PrevLogIndex = 11, PrevLogTerm = 3`
	- Lo que va a pasar ahora es que S2 se va a fijar en lo que tiene en la entry 11, y va a encontrar que el term de esa celda es igual al que le manda S3, entonces va a aceptar el packet. 
	- IMPORTANTE: S2 va a pisar lo el 4 que había en la entry 12, y lo va a reemplazar por lo que le manda S3.
	
| Node ↓ / LogEntryID → | 10  | 11  | 12          | 13  |
| --------------------- | --- | --- | ----------- | --- |
| S1                    | 3   |     |             |     |
| S2                    | 3   | 3   | ~~4~~ **5** | 6   |
| S3                    | 3   | 3   | 5           | 6   |

---
## Persistencia
- Al revivir, los servers tienen que recordar en qué term estaban, a quién habían votado (si es que lo hicieron), y su Log.
- Todo lo demás se puede reconstruir a partir de lo que se persistió
- **IMPORTANTE**: Primero escribo en el disco, luego respondo.


### Log Compaction
- La idea es que cada cierto tiempo, Raft tome un snapshot del storage que tiene la App de arriba, y que la guarde en el Log. 
	- El Snapshot es una tira de bytes para Raft, no le interesa el formato.
	- Entonces lo que se puede hacer es borrar todo lo que estaba detrás del Snapshot en el Log (ya sabemos que al momento de tomarlo, todo estaba bien persistido).
	- El restore termina siendo rapidísimo, porque se desearializa el snapshot, se lo pasa a la capa de arriba, y se comienzan a aplicar las siguientes entradas del log.