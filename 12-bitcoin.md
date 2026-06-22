---
aliases:
  - Bitcoin
---
# Bitcoin
- Nace como necesidad de quitarse de encima intermediarios al hacer transacciones monetarias.
	- Permissionless: cualquiera puede unirse a la red.
	- Descentralizado: cualquiera puede emitir transferencias -> No hay un actor central que pueda cancelar transacciones o suspender cuentas.
	- Tamper-proof: no se puede falsificar la historia. En Bitcoin, esto se reduce a que no se pueden emitir double-spends.

## Identidad
- Bitcoin vive en un entorno bizantino: todos los nodos son independientes, y nadie confía en nadie.
- No se tiene más autentificación centralizada -> Queremos que una transacción tenga autoría y no falsificable.


> [!NOTE] Clave privada
> Se usa para firmar mensajes
> `Sig(message, secret_key)`


> [!NOTE] Clave pública
> Se usa para verificar mensajes
> `Verify(message, signature, public_key)`

- No existe más el "juan le manda a mari". Existe un mensaje firmado por juan que envía un monto a la clave pública de mari (su identidad online)

```
prev_tx_hash
dst = pubkey_mari
n = cant de btc que mando
s = sig(enc(prev_tx_hash) + enc(dst) + enc(n), secret)
```

- Incluir el hash de la TX anterior hace que la próxima, aunque tenga el mismo destinatario y N de btc, tenga una firma completamente distinta.

### Más tamper proofing 
- Agrupamos las transacciones en bloques -> Se reduce el ancho de banda consumido
- Linkeamos cada bloque al anterior -> Blockchain
- No puedo cambiar historia sin cambiar el hash actual

![](img/Pasted%20image%2020260523193648.png)

### Proof of Work
- Para postear un bloque se tiene que resolver un problema difícil -> La idea es que no se empiecen a spammear identidades/nodos para falsificar transacciones
- `x/hash(x) = 0` es un problema imposible -> `hash(x) < N` es una relajación del problema (trap door).
	- Por ej, podemos buscar Nonce tal que `hash(bloque) < 0x0fffffff`
		- Generar la prueba es probar hashes (lento)
		- Verificar la prueba es fácil (hashear el bloque y chequear leading).
- El que propone un bloque añade una coinbase transaction donde se crea plata y se la envía a sí mismo.
	- Cada 4 años se baja a la mitad la recompensa. Eventualmente, para el año 2140 no se emite más. Quedarán solo fees para los mineros.
	- La recompensa es un mecanismo de emisión y distribución como un incentivo para los mineros.

![](img/Pasted%20image%2020260615115210.png)
- El líder no se elige, aparece
	- El primero en resolver el PoW propaga el bloque, y todos trabajan sobre el último bloque inmediatamente.

![](img/Pasted%20image%2020260615120745.png)

- Puede pasar que dos nodos resuelvan el problema a la vez, y aparezcan ramificaciones
	- En un futuro, va a aparecer algún nodo que resuelva el problema para el siguiente bloque, y entonces quedaría una rama más larga que otra
		- Acá se elige la rama más larga como segundo criterio
			- En términos de CAP, se elige la availability sobre la consistencia, porque si descartamos una rama por otra más larga, posiblemente tengamos problemas de consistencia.

![](img/Pasted%20image%2020260615121427.png)

> [!NOTE] Parámetros
> Dificultad ajustable según tráfico. No hace falta ponerse de acuerdo, es función de los timestamps de los bloques.


### UTXOs: Unspent Transaction Outputs

![](img/Pasted%20image%2020260615124914.png)

- Facilita unicidad del contenido de los mensajes, con lo cual se imposibilita el falsificar mensajes.
- No se procesa el estado de una cuenta, sino que se valida o invalida un conjunto de registros inmutables. Los reorgs son muy fáciles.
- Perfectamente paralelizable: Los UTXOs para una cuenta son independientes entre sí, lo cual permite ejecutar transacciones en paralelo.
