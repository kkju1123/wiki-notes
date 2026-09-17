---
title: Graph Engineering：从串行 Agent 到并行 Agent 舰队的 14 步路线图
url: https://x.com/angeldot_/status/2081061068516798931
source_type: twitter
author: angeldot_
tags:
- Graph Engineering
- Agent Orchestration
- Parallelism
- Claude Code
- LLM Workflow
summary: 把线性多步 Agent 重构为数据依赖图，删除伪依赖并用 parallel() 并发调度子代理，减少等待与 token 消耗。
fetched_at: '2026-09-17T02:40:33.618686+00:00'
---

# El roadmap de 14 pasos para pasar de  prompts a mover una flota de agentes que trabajen por ti

La mayoría de la gente que monta un agente multi paso acaba con una línea recta.

Paso uno, paso dos, paso tres. Cada uno esperando educadamente a que el anterior termine antes de arrancar.

Y casi nadie se da cuenta de que la mitad de esos pasos nunca tuvo que esperar a nada.

No enrutan. No ramifican. No paralelizan. Solo hacen cola. Una cabeza, un contexto, una cosa a la vez, hasta que la ventana se llena y el agente se olvida de lo que estaba haciendo.

Este es el roadmap de 14 pasos que convierte esa fila india en un grafo. Uno que se abre en abanico sobre una flota entera, verifica sus propios hallazgos y converge en un resultado que un agente solo jamás podría sostener.

Al terminar vas a saber tres cosas:

→ dónde tu sistema actual está esperando sin motivo → cómo repartir el trabajo sin perder la confianza en lo que sale → cómo bajar la factura sin tocar la calidad

## EL CAMBIO DE MARCO QUE NADIE TE EXPLICA

Un prompt es una frase. Un loop es un ciclo. Un harness es el suelo sobre el que se apoya el agente.

Pero la forma del trabajo en sí, qué corre antes de qué, qué puede correr a la vez, qué tiene que esperar a todo lo demás, esa forma es un grafo.

Los nodos piensan. Las aristas transportan resultados.

Y Claude Code ya trae el tooling para construirlos directamente: los workflows dinámicos.

Claude escribe un script de orquestación en JavaScript plano y luego lanza una flota coordinada de subagentes que lo ejecutan. La coordinación en sí cuesta cero tokens de modelo, porque es código, no una conversación.

El roadmap va en cuatro bloques:

→ 01 a 04, ver el grafo que ya tienes → 05 a 08, mover la flota → 09 a 11, confiar en lo que sale → 12 a 14, que no te arruine

# BLOQUE 1 · VER EL GRAFO QUE YA TIENES

## 01. LOS NODOS SON ENCARGOS, LAS ARISTAS SON LO QUE FLUYE

Un grafo tiene exactamente dos cosas, y tenerlas claras arregla casi toda la confusión.

→ Nodo: una unidad de trabajo. Un agente, un encargo acotado, una entrada, una salida. → Arista: una dependencia. Dice que la salida de este nodo alimenta la entrada de aquel. Nada más.

El error es tratar el "y luego" como si fuera una arista.

"Resume este archivo y luego dime el tiempo" no tiene ninguna arista entre las dos partes. El parte meteorológico no consume el resumen. Son dos nodos desconectados que un script lineal encadena sin ninguna necesidad.

La arista solo existe cuando los datos cruzan de verdad.

Dibújalo como cajas y flechas. Una caja es una llamada a agent(). Una flecha es una variable que sale del return de una llamada y entra en el prompt de otra.

Si no puedes dibujar la flecha, si no cruza ninguna variable, esas dos cajas son independientes.

Y esa independencia es lo que vas a explotar durante los 13 pasos siguientes.

→ La regla: por cada "y luego" de tu agente, pregunta si el siguiente paso lee la salida del anterior. Si no, la espera es tiempo tirado.

## 02. TU SCRIPT LINEAL YA ES UN GRAFO, SOLO QUE DEGENERADO

Cuando escribes un agente como "haz A, luego B, luego C, luego D", ya has dibujado un grafo. Una cadena sin ramas. Cada nodo con exactamente una arista de entrada y una de salida.

Funciona. También corre lento y frágil, porque una cadena no tiene redundancia: si C se atasca, D no ocurre nunca, y el trabajo de A se queda atrapado río arriba sin sitio adonde ir.

La primera habilidad real del graph engineering es redibujar la cadena.

Coge tu agente lineal y, por cada flecha, hazte la pregunta del paso 01.

La mayoría de cadenas tiene dos o tres flechas que no transportan datos. Están ahí solo porque es el orden en el que fuiste tecleando.

Corta esas flechas y la cadena colapsa en algo más ancho: unos cuantos nodos independientes que podrían correr todos a la vez, alimentando a un único nodo que sí los necesita todos.

Esto no cuesta dinero, no requiere herramienta nueva, y normalmente elimina más espera que cualquier cosa que puedas comprar.

→ La regla: antes de añadir nada, borra las aristas que no transportan datos.

## 03. DALE UN CONTRATO A CADA NODO

Un nodo sobre el que no puedes razonar es un nodo que no puedes paralelizar.

El arreglo es un contrato: entrada acotada, salida acotada, un solo trabajo.

→ la entrada es lo que el nodo lee, pasado explícitamente, nunca asumido de una ventana compartida → la salida es una forma definida, idealmente validada, para que el siguiente nodo la consuma sin adivinar

En un workflow ese contrato se impone con un schema. Cuando le pasas a Claude una llamada agent() con un JSON schema, el subagente está obligado a devolver datos estructurados válidos. La validación ocurre en la capa de la tool call, así que hay reintento cuando no encaja, en lugar de devolverte texto libre que tú tienes que parsear y rezar para que salga bien.

Esa es la diferencia entre un nodo que Claude puede cablear a un grafo y un nodo que solo funciona cuando lo lee un humano.

→ La regla: si la salida de un nodo no tiene forma, no es un nodo. Es una conversación.

## 04. LA ARISTA TAMBIÉN ES UN CONTRATO, Y ADEMÁS ES GRATIS

Una arista no es "B va después de A". Es una promesa sobre lo que cruza: A produce esta forma, B está construido para consumir esta forma.

Cuando nombras la arista por su dato en vez de por su orden, dos cosas se vuelven fáciles:

→ ves al instante si la arista es real, porque puedes preguntar si algo se mueve por ella → puedes cambiar el nodo de cualquiera de los dos extremos sin romper el grafo, mientras la forma se mantenga

En la práctica, la arista vive en JavaScript plano. El paso de reducción entre el fan out y la síntesis, aplanar, deduplicar, filtrar, es solo código operando sobre las formas que devolvieron tus nodos.

No hace falta ningún agente ahí.

Y esta es una de las victorias silenciosas de pensar en grafos: una cantidad brutal de lo que la gente paga en tokens es en realidad una arista, y las aristas son gratis.

La tentación es lanzar un agente para "combinar los resultados". Resístete.

Si combinar significa aplanar y deduplicar, eso es un flatMap y un Set. Determinista, instantáneo, cero tokens.

→ La regla: los agentes son para el criterio, no para la fontanería. Un grafo donde cada arista es un agente paga alquiler por su propio cableado.

# BLOQUE 2 · MOVER LA FLOTA

## 05. ABRE EN ABANICO CON parallel()

Esta es la jugada que paga todo lo demás.

Cuando tienes N nodos independientes, N fuentes que consultar, N archivos que revisar, N rutas que auditar, no los encadenas. Le dices a Claude que los abra en abanico y los corra a la vez.

En un workflow eso es parallel(): coge un array de thunks, lanza un subagente por thunk, todos ejecutando en concurrencia, y te devuelve el array de resultados.

Dos detalles lo hacen robusto:

→ parallel() es una barrera. Espera a todos los thunks antes de devolver, así que la siguiente etapa ve el conjunto completo. → un thunk que lanza excepción resuelve a null en vez de tumbar el lote entero, así que un agente inestable no puede hundir la ejecución.

Por eso siempre .filter(Boolean) a la salida.

La concurrencia va topada más o menos por tu número de núcleos y el exceso se encola, así que puedes pasar cien thunks y van a terminar todos, solo que de puñado en puñado.

Y aquí está lo importante: el fan out vive en código que escribió Claude, no en una conversación con el modelo.

El contexto propio de Claude nunca sostiene nueve fuentes a la vez. Cada subagente carga la suya, y solo vuelve la respuesta final.

Eso es lo que permite escalar a decenas o cientos de subagentes sin ahogar la sesión. La capa de orquestación cuesta cero tokens porque no es otro turno de Claude pensando.

→ La regla: si N cosas no se leen entre sí, no las encoles. Ábrelas.

## 06. CIERRA EL ABANICO EN UNA BARRERA

Un fan out solo sirve si algo lo recoge.

El fan in es el nodo donde convergen las aristas. Donde un agente, o un trozo de código, ve todos los resultados de arriba a la vez y hace algo que requiere el conjunto entero: deduplicar entre fuentes, ordenar por impacto, salir temprano si no vino absolutamente nada.

Ese es el único sitio donde una barrera se gana su coste en tiempo de reloj.

¿Deduplicar entre todas las fuentes? Barrera, correcto.

¿Solo aplanar una lista? Eso es una arista, hazlo inline.

El test es simple: si escribiste paralelo → transformar → paralelo, y esa transformación del medio no tiene ninguna dependencia entre elementos, tenías que haber usado un pipeline y saltarte la barrera entera.

→ La regla: barrera solo cuando una etapa necesita de verdad todos los resultados anteriores juntos.

## 07. EL DIAMANTE: PARTE, TRABAJA, FUSIONA

Junta fan out y fan in y tienes la topología de trabajo de cualquier grafo de agentes serio: el diamante.

Un nodo parte el encargo. Muchos nodos hacen el trabajo en paralelo. Un nodo lo fusiona.

La forma canónica tiene un nombre que merece memorizarse: fan out → reduce → sintetiza.

→ fan out para ganar amplitud → reduce con código plano para comprimirla → sintetiza con un agente final que escribe la respuesta

Es el esqueleto detrás de un escaneo de mercado, una auditoría de dependencias, una revisión de código y un informe de research. Cambias las fuentes y los prompts y el mismo esqueleto se adapta.

Cuando ves el diamante dejas de preguntarte "cómo hago que mi agente dé más pasos" y empiezas a preguntarte "dónde está el corte y dónde está la fusión". Que es la pregunta que de verdad escala.

→ La regla: no diseñes pasos. Diseña dónde parte y dónde une.

## 08. ENRUTA LA ARISTA EN TIEMPO DE EJECUCIÓN

No todos los grafos son fijos. A veces la arista que se toma depende de lo que encontró un nodo.

Un nodo router inspecciona un resultado y decide qué camino se dispara. Clasifica el ticket y ramifica al handler correcto. Mira el tamaño del diff y elige entre una revisión rápida o levantar una auditoría completa.

En un workflow esto es un simple if o switch de JavaScript sobre la salida validada de un nodo, porque el control de flujo vive en código.

Aquí el determinismo es una feature, no una limitación.

La decisión del router puede venir de Claude, pero el enrutado es código. Corre igual siempre para la misma clasificación.

Criterio del modelo en el nodo, fiabilidad del script en la arista. Sin sorpresas emergentes del tipo "el agente decidió saltarse la auditoría", porque ese salto tendría que estar escrito en el grafo. Y no lo está.

→ La regla: que el modelo decida qué es esto, y que el código decida qué se hace con ello.

> LA CONTRAREGLA (léela antes de seguir)
Un grafo compra amplitud. No compra mejor criterio.Si cada paso de tu trabajo necesita la foto completa del anterior, repartirlo entre agentes no te da una respuesta mejor. Te da la misma respuesta, más cara y más tarde.El grafo empieza a pagar en el momento exacto en el que el trabajo se parte en encargos que nunca leen los resultados de los otros.Antes de añadir un solo agente, la única pregunta que decide tu factura: ¿dónde se parte mi trabajo?Si no se parte, quédate con un agente y ahórrate los 6 pasos que quedan.

# BLOQUE 3 · CONFIAR EN LO QUE SALE

## 09. PON UN VERIFICADOR EN LA ARISTA

El apalancamiento real de un grafo no es tener más agentes. Es la estructura que puedes montar alrededor de ellos para producir confianza.

Un nodo verificador se sienta en la arista antes de que un resultado tenga permiso para bajar. Su único trabajo es intentar matar el hallazgo. Si sobrevive, pasa. Si no, no llega nunca a la respuesta.

Y hay una regla dura debajo de todo esto: nunca dejes que el mismo agente corrija su propio examen. Un modelo revisando su propia salida se pierde la mayoría de sus propios errores, porque está evaluando desde el mismo sitio desde el que se equivocó.

Tres patrones que conviene tener en la mano:

→ Verificación adversarial: por cada hallazgo, lanza N escépticos independientes con la orden de refutarlo. Se queda solo si sobrevive a la mayoría.

→ Verificación de lentes diversas: dale a cada verificador un ángulo distinto. Correcto, seguro, se reproduce. La diversidad caza modos de fallo que N comprobaciones idénticas no encuentran jamás.

→ Panel de jueces: genera N intentos desde ángulos distintos, puntúalos con jueces en paralelo, sintetiza desde el ganador injertando lo mejor de los que quedaron cerca.

Este es el patrón que hay detrás de los proyectos más ambiciosos que se han hecho con agentes, como portar un runtime entero con revisión adversarial metida dentro del propio loop en lugar de al final.

→ La regla: ningún hallazgo viaja sin verificar, y dos verificadores nunca hacen la misma pregunta.

## 10. AÍSLA LOS NODOS PARA QUE UN FALLO NO ENVENENE EL GRAFO

En una cadena, un fallo cascadea. C muere, D no corre nunca, todo se para.

En un grafo, el fallo debería quedarse contenido en su nodo.

Eso ya es parcialmente cierto: un thunk que revienta dentro de parallel() resuelve a null, así que ocho agentes buenos vuelven igual mientras el malo se cae solo. Tu .filter(Boolean) es la contención.

Diseña cada fan in para tolerar entradas ausentes en vez de asumir el conjunto completo.

El fallo más sutil es que los nodos se pisen entre ellos. Cuando varios agentes escriben archivos en paralelo, colisionan.

El arreglo es aislamiento: cada agente corre en su propio worktree de git, hace su trabajo en un sandbox, y luego se fusiona limpio.

Recurre a esto solo cuando los nodos escriben de verdad en paralelo. Es el cinturón de seguridad de la única topología que lo necesita, no un impuesto por defecto sobre cada ejecución.

→ La regla: un solo escritor por archivo. Y todo fan in tolera huecos.

## 11. AÑADE UN CICLO, PERO QUE CONVERJA

A veces no sabes lo grande que es el trabajo hasta que estás dentro. Descubrimiento de tamaño desconocido. Un barrido de bugs donde encontrar uno revela otros tres.

Eso pide un ciclo: una arista controlada de vuelta a un nodo anterior.

El peligro es evidente. Un ciclo que no converge es un bucle infinito lanzando agentes hasta fundir tu presupuesto.

El patrón que sí converge es loop until dry: sigue lanzando buscadores hasta que K rondas consecutivas no saquen nada nuevo, y ahí para.

Y el detalle que lo hace o lo rompe, el error que casi todo el mundo comete la primera vez, es contra qué deduplicas.

Deduplica contra todo lo visto, no contra lo confirmado.

Si no, los hallazgos rechazados reaparecen cada ronda, el bucle no se seca nunca, y has construido una máquina que paga por redescubrir los mismos callejones sin salida para siempre.

→ La regla: todo bucle lleva un tope de rondas, y deduplica contra todo lo visto.

# BLOQUE 4 · QUE NO TE ARRUINE

## 12. ESCALONA LOS MODELOS ENTRE LOS NODOS

No todos los nodos necesitan tu mejor modelo.

Un grafo hace esto obvio de una forma que un agente único nunca consigue. Hay nodos acotados y repetitivos, extrae este campo, clasifica este ticket. Y hay nodos donde vive el criterio de verdad, sintetiza el informe, adjudica el hallazgo.

Corre los aburridos en un modelo barato y gasta los tokens caros donde el criterio importa.

En un workflow, cada subagente hereda el modelo de tu sesión salvo que el script lo sobreescriba. Por defecto, una ejecución grande factura entera a tu nivel de sesión.

La opción model en una llamada agent() concreta enruta solo ese nodo a otro sitio.

Revisa /model antes de una ejecución grande, y luego baja los nodos repetitivos del fan out manteniendo arriba el nodo de fusión.

Esta es la palanca que convierte un grafo hambriento de tokens en algo económico sin tocarle la forma.

→ La regla: modelo caro solo donde hay criterio en juego.

## 13. LA TOPOLOGÍA ES TU COSTE Y TU LATENCIA

La forma del grafo no es cosmética. Es la mayor palanca sobre el tiempo de reloj que existe.

La decisión que hace tropezar a todo el mundo: parallel() contra pipeline().

→ una barrera parallel() hace que todo espere al nodo más lento antes de que empiece la siguiente etapa → un pipeline() hace fluir cada elemento por todas las etapas de forma independiente, sin barrera. El elemento A puede estar en la etapa 3 mientras B sigue en la 1

Los elementos rápidos terminan pronto en vez de quedarse parados detrás de los lentos.

Por defecto, pipeline. Recurre a la barrera solo cuando una etapa necesita de verdad todos los resultados anteriores a la vez: un dedupe contra el conjunto, una salida temprana según el total, un prompt que compara contra "los otros hallazgos".

"Queda más limpio" y "las etapas se sienten separadas" no son razones. La latencia de barrera es tiempo real, medible y desperdiciado.

Separado no es lo mismo que sincronizado.

→ La regla: pipeline por defecto, barrera por excepción justificada.

## 14. DEJA QUE CLAUDE DIBUJE EL GRAFO

La jugada final es dejar de dibujar el grafo a mano para trabajos que no puedes planificar de antemano.

Con workflows dinámicos describes el objetivo y Claude escribe el propio script de orquestación: descompone la tarea, elige el fan out, lanza una flota coordinada de subagentes y sintetiza el resultado.

Consigues un grafo hecho a medida de esa ejecución concreta en lugar de uno fijo que esperabas que encajara.

Hay tres vías de entrada:

→ di la palabra "workflow" en tu prompt y Claude escribe uno para la tarea → ejecuta uno guardado o uno que venga de serie. /deep-research es un grafo real corriendo en producción: acotar → búsqueda paralela → fetch → verificación adversarial → síntesis. Exactamente el esqueleto de este roadmap. → activa el modo que planifica un workflow para cada tarea sustancial de la sesión

Cuando una ejecución sale bien, guarda su script en .claude/workflows/. Versionado, re ejecutable por nombre y lanzable por cualquiera que clone el repo.

› Lanza un workflow que audite todas las rutas de src/routes/ buscando
  auth ausente. Un agente por archivo de ruta, y verifica cada hallazgo
  antes de reportarlo.

● Claude escribió un script de orquestación · lanzando en background…

  /workflows — auth-audit · running
  ✓ Acotar      1/1     2.1k tok · 4s
  ✓ Fan-out     18/18   un agente por archivo de ruta
  ◯ Verificar   11/18   escépticos a 3 votos por hallazgo…
  ○ Sintetizar  0/1     esperando a verificar

  la sesión sigue respondiendo, puedes seguir trabajando mientras corre la flota

→ La regla: si el trabajo cambia de forma en cada ejecución, no dibujes el grafo. Descríbelo.

# SEIS GRAFOS PARA CONSTRUIR ESTA SEMANA

Barrido de seguridad sobre todas las rutas. Un subagente por archivo de ruta, cada uno cazando comprobaciones de auth ausentes, y luego una pasada de verificación que confirma cada hallazgo antes de que llegue al informe. Amplitud que ningún contexto único podría sostener.

Informe con fuentes vía /deep-research. Un grafo que ya viene dentro de Claude Code. Descompone tu pregunta en ángulos distintos, corre búsquedas en paralelo, deduplica fuentes y verifica adversarialmente cada afirmación con escépticos a tres votos antes de escribir.

Portar un módulo, archivo a archivo. Fan out de la traducción sobre los archivos, la suite de tests como puerta en cada uno, y los fallos vuelven al bucle. La revisión adversarial caza lo que una sola pasada habría entregado roto.

Revisión adversarial de un diff. Enruta según el tamaño: un cambio pequeño se lleva una pasada rápida, uno grande dispara una auditoría paralela completa con revisores en lentes distintas, correcto, seguro, rápido, y luego un panel de jueces sintetiza.

Escaneo del ecosistema programado. Se guarda una vez y se re ejecuta para siempre. Consulta muchas fuentes en paralelo, releases, blogs, foros, ordena por impacto en una barrera y escribe el resumen. Versionado en .claude/workflows/, lanzable por nombre.

Descubrimiento de tamaño desconocido. No sabes cuántos bugs hay. Buscadores en paralelo, dedupe de cada hallazgo nuevo contra todo lo visto, verificación de los supervivientes, y sigue el bucle hasta que dos rondas no saquen nada. Entonces para.

# CHECKLIST ANTES DE LANZAR TU PRIMER GRAFO

→ ¿he borrado las aristas que no transportan datos? → ¿el trabajo se parte de verdad, o cada paso necesita el anterior? → ¿cada nodo tiene entrada acotada, salida con forma y un solo encargo? → ¿estoy pagando un agente para hacer algo que es un flatMap? → ¿hay alguna barrera que no necesita el conjunto entero? → ¿algún hallazgo llega al resultado sin que nadie haya intentado matarlo? → ¿dos verificadores hacen la misma pregunta? → ¿todos los bucles tienen tope de rondas? → ¿hay más de un nodo escribiendo el mismo archivo? → ¿los nodos repetitivos están corriendo en el modelo caro?

Si fallas más de tres, no tienes un grafo. Tienes una cadena con más pasos.

# CONCLUSIÓN

El que promptea hace una pregunta. El que arquitecta dibuja un grafo.

El agente lineal nunca fue el techo. Era solo la primera forma, la que todo el mundo coge porque coincide con la manera en la que escribimos. Una línea, una cabeza, una cosa a la vez.

Cuando empiezas a ver los nodos y las aristas dejas de pedirle al agente que haga más pasos y empiezas a pedirle al grafo que lo haga más ancho.

Abanico donde el trabajo es independiente. Puertas en las aristas donde importa la confianza. Modelos más baratos donde no hay criterio en juego.

La mayoría va a seguir encolando pasos en una fila india.

Los que aprendan a dibujar el grafo van a mover una flota entera. Y no van a notar nunca el techo bajo el que está atascado el resto.

Dibuja tu sistema actual esta noche. Solo los encargos y las flechas entre ellos. Cuenta las aristas falsas y bórralas.

Es el primer movimiento de todo el oficio, cuesta cero, y normalmente elimina más espera que cualquier herramienta que puedas comprar.