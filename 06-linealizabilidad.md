---
aliases:
  - Linealizabilidad
recurso:
---
## Consistencia Eventual
La consistencia eventual es una consecuencia de replicar
- Si un cliente escribe algo y después lo lee, el líder normalmente debería de tener toda la info actualizada. Pero si vamos a un follower, puede no estar actualizado y dar otra cosa.
	- Eventualmente ese follower debería tener la info actualizada.
	- Esto aparece producto de replicar y relajar un poco el tema de consistencia -> con esto podemos ser un cliente e ir a buscar data en otras máquinas que no sean líderes.

## Consistencia Fuerte

> [!QUOTE] Linealizabilidad
> El sistema entero se comporta como si hubiera una sola copia de los datos.

1. El orden de las operaciones respeta el tiempo real.
2. Es válida lógicamente.

### ¿Cómo hacer Raft Linealizable?
- Con las escrituras no habría problema, porque al escribir modificamos el log, y hacemos el baile con los followers para conseguir el quorum.
	- Ahora, con las lecturas podríamos hacer lo mismo (aunque en la realidad no lo hacemos porque directamente llega el req, leemos y devolvemos), y mantener en el log las operaciones de lectura (que sería una "no-operación", porque no muta la data)
		- Esto sería mucho más costoso, porque la mayoría de req van a ser lecturas, y si hacemos todo el baile para responder una lectura, se nos va en costos (aunque aseguraríamos consistencia fuerte).