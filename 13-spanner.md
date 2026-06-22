---
aliases:
  - Spanner
---
# Spanner
- DB distribuida que soporta transacciones (ACID).
- Se tienen tablas shardeadas por rangos de PKs.
	- Los shards se replican y están en distintos datacenters. -> Parecido a lo que hace [DynamoDB](10-dynamodb.md).
	- Usa MultiPaxos.

# Transacciones
- Read/Write $T_X$ -> Usan 2PC + 2PL (Locking pesimista) + Paxos groups
- Read Only $T_X$  -> Sin Locks (usan MVCC). -> Snapshot Isolation, True Time

## Read/Write $T_X$ 
- Al arrancar, el cliente elige a uno de los participantes ($P_i$, que son los grupos de replicación de los shards) como Transaction Coordinator (TC).

```mermaid
sequenceDiagram
    autonumber

    participant C as Cliente
    participant P1 as P1
    participant P2 as P2

    Note over C,P2: Read / Write Transaction

    C->>P1: READ(k1)
    Note right of P1: READ LOCK
    P1-->>C: valor k1 + versión/LSN

    C->>P2: READ(k2)
    Note right of P2: READ LOCK
    P2-->>C: valor k2 + versión/LSN

    Note over C: Buffer de writes<br/>todavía no aplicados

    C->>P1: ELIGE TC
    Note right of P1: P1 queda como<br/>Transaction Coordinator

    C->>P1: WRITE(k1, v1)
    Note right of P1: WRITE LOCK
    P1-->>C: write buffered

    C->>P2: WRITE(k2, v2)
    Note right of P2: WRITE LOCK
    P2-->>C: write buffered

    C->>P1: COMMIT(tx)

    rect rgb(230, 255, 230)
        Note over P1,P2: 2 Phase Commit - Prepare
        P1->>P1: PREPARE local
        P1->>P2: PREPARE(tx)
        P2-->>P1: OK
    end

    rect rgb(230, 255, 230)
        Note over P1,P2: Todos OK → Commit
        P1->>P1: COMMIT local
        P1->>P2: COMMIT(tx)
        P2-->>P1: COMMIT OK
    end

    P1-->>C: COMMIT OK

    Note right of P1: libera locks
    Note right of P2: libera locks
```

- 2PL garantiza serializabilidad
- Lento -> 10 - 100ms de tiempos de ejecución -> No importa tanto porque este tipo de transacciones son minoría.

---
## Read Only $T_X$
- Representan el 99.9% de las transacciones.
- La baja latencia se logra en parte leyendo de réplicas locales (Snapshot Isolation).
	- Tampoco lockean -> No hay 2PL
- Correctitud (External Consistency): La transacción tiene que ver a aquellas transacciones que commitearon en el pasado
	- La transacciones son serializables.
	- [Linealizabilidad](06-linealizabilidad.md)

### Multi-Version Concurrency Control (MVCC)
- Guardamos varias versiones del registro

| (k, ts) | V  |
|---------|----|
| (x, 10) | v1 |
| (y, 9)  | v2 |
| (y, 12) | v3 |
- Si llega una $T_{X, \, R/O} \, @11$:
	- $R_X=10$
	- $R_Y=9$

---
## Réplicas desactualizadas
- Si tengo un grupo de réplicas, puede llegar a pasar de tener alguna de ellas desactualizadas temporalmente, y puede pasar que una transacción caiga a leer valores a esa réplica, y obtendría un valor desactualizado (violando linealizabilidad)

```mermaid
flowchart LR
    subgraph R1["Follower"]
        R1Data["(x, 10)<br/>(x, 11)"]
    end

    subgraph R2["Líder"]
        R2Data["(x, 10)<br/>(x, 11)"]
    end

    subgraph R3["Follower"]
        R3Data["(x, 10)"]
    end

    Tx["Tx R/O @ 12<br/><br/>Lee valor viejo<br/>(x, 10)"]

    R2Data --> R1Data
    R2Data --> R3Data
    Tx --> R3Data
```


### Safe Time
- Las réplicas guardan el timestamp más actualizado que tienen
	- Al llegar una transacción pidiendo algo con timestamp más grande que el que tiene la réplica, espera a tener un timestamp mayor al que pide la transacción
		- De esta forma no vamos a darle cosas del pasado 

---
## Lector Atrasado
- Va a leer cosas del pasado
	- Esto es inconsistente (NO linealizable)
- Hay que sincronizar relojes de alguna forma

### True Time
- True Time (`TT`) es un TDA que representa a un intervalo
	- `[earliest, latest]` -> Intervalo de Confianza
	- `TT.now()` nos devuelve el intervalo
	- `TT.after(t)` -> true si ya pasó `t`.
	- `TT.before(t)` -> true si `t` es anterior.

### Elegir el Timestamp de la $T_X$ de Lecturas a partir de `[Earliest, Latest]`

![](img/Pasted%20image%2020260615150714.png)

- Si elegimos a 9 como el timestamp de $T_2$, vamos a estar ocultando la $T_1$ (porque $T_1$ tiene timestamp 10)
	- Entonces elegimos siempre `tt.now().latest`

### Elegir el Timestamp de la $T_X$ de Escrituras a partir de `[Earliest, Latest]`

![](img/Pasted%20image%2020260615151151.png)

- En este caso, la escritura no puede tomar como timestamp a 12 (latest), porque sino estaríamos ocultando en $T_2$ cosas del pasado.
- Si tomamos el timestamp de la escritura en 9 (osea, en el pasado), corremos el riesgo de reescribir la historia

#### Reglas para solucionar el problema
1. Start Rule -> Tomar `tt.now()latest` cuando arranca el commit de la transacción -> Ese va a ser el TS de la transacción
2. Commit Wait -> Esperar a que TS pase luego de `now()`

![](img/Pasted%20image%2020260615151923.png)