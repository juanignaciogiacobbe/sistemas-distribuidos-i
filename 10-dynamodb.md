---
aliases:
  - DynamoDB
---
# DynamoDB
- Servicio Cloud administrado por AWS
	- Fully Managed (aprovisionamiento, escalado, actualización de software, tolerancia a fallas, replicación, etc)
	- Multi-Tenant -> Sobre una misma infraestructura suelen vivir varios clientes (obviamente separados a nivel lógico) 
	- API con SLA estricto -> Garantía de disponibilidad y latencia
	- Escala "ilimitada"
- Los items tienen atributos, y una Primary Key (se suele usar como Partition Key)
	- La Partition Key es la clave que vamos a usar para shardear.
	- NO usa Consistent Hashing.

![](img/Pasted%20image%2020260618222000.png)

- El RequestRouter es un server que recibe las request, y las forwardea a los StorageNodes
	- Se encarga de la autenticación y validación del request
	- También habla con el Partition Metadata System, que es un componente que guarda la información sobre la ubicación de los distintos shards -> Esto guarda también el map de rango de hashes asignados a cada Replication Group
- Los StorageNodes mantienen un WAL, que es justamente el Log que va a replicar con los integrantes del Replication Group (usa Multi-Paxos).
	- Usan un B-Tree para guardar los datos

## Auto-Admin
- Separa a las distintas operaciones en Control Plane y Data Plane:
	- Control Plane: `CreateTable`, `CreateIndex`, `UpdateTable`, ...
		- Gestión del sistema
	- Data Plane: `PutItem`, `GetItem`, `Delete`, ...
		- Requests en tiempo real sobre los datos
- Tienen distintos requisitos funcionales y no funcionales
	- No es lo mismo el SLA esperado para crear una tabla , que para poner un item (este es más crítico)
	- La disponibilidad NO es uniforme
		- Si el Control Plane se cae, la idea es que el Data Plane siga funcionando.

![](img/Pasted%20image%2020260620145926.png)

### Funcionalidades del Auto-Admin
- Ciclo de vida de Tablas
	- Las operaciones de creación de tablas suelen hacerse de forma async (son tareas lentas debido a la cantidad de nodos que participan de esa operación) -> En general, las tareas del Control Plane suelen ser async.
- Monitoreo de flota (máquinas que se usan)
	- Detección de fallas, y restaura el sistema (reemplaza nodos en caso de failure)
- Scaling y rebalanceo
	- Se encarga de mantener un balance de la cantidad de partitions manejadas por nodo de storage.
- Gestión de backups