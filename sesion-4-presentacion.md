---
marp: true
theme: default
paginate: true
header: '![h:34px](nobleprog-logo.png) Tecnología de Punta Formación SL'
footer: 'Sesión 4 — Comunicación, resiliencia y organización'
style: |
  section {
    font-size: 26px;
  }
  header {
    display: flex;
    align-items: center;
    gap: 10px;
    left: 30px;
    top: 18px;
    font-size: 16px;
    font-weight: 600;
    color: #1a5276;
  }
  h1 {
    color: #1a5276;
  }
  h2 {
    color: #1a5276;
  }
  code {
    font-size: 0.85em;
  }
  table {
    font-size: 0.8em;
  }
  section.lead {
    text-align: center;
  }
  section.lead h1 {
    font-size: 1.8em;
  }
---

<!-- _class: lead -->
<!-- _paginate: false -->
<!-- _header: '' -->
<!-- _footer: '' -->

![w:700](logo.png)

# Sesión 4
## Comunicación, resiliencia y organización

**Migración de Monolitos a Microservicios**
Carlos Georg Lübbe · 3 h

<!--
ES: Bienvenida a la sesión final. Las Sesiones 1–3 decidieron POR QUÉ, CÓMO y DÓNDE
cortar, y qué hacer con los datos. Hoy cerramos el círculo: que el sistema resultante
sea robusto de operar (resiliencia, observabilidad) y sostenible en la organización
(Conway, Team Topologies), y juntamos todo en una hoja de ruta de migración. Tono: menos
modelado, más operación y personas.

EN: Welcome to the final session. Sessions 1–3 decided WHY, HOW and WHERE to cut, and what
to do with data. Today we close the loop: make the resulting system robust to operate
(resilience, observability) and sustainable in the organization (Conway, Team Topologies),
and tie it all into a migration roadmap. Tone: less modeling, more operations and people.
-->

---

## Objetivo de hoy

Que el sistema resultante sea **robusto de operar** y esté **alineado con los equipos** — y cerrar con una **hoja de ruta**.

1. **Comunicación:** síncrono vs. asíncrono, eventos, gateway, contratos
2. **Resiliencia:** que el fallo parcial **no se propague**
3. **Observabilidad:** ¿dónde se rompió entre 6 servicios?
4. **Pruebas y entrega** durante la migración
5. **Organización:** la ley de Conway y Team Topologies
6. **Síntesis:** la hoja de ruta y los errores frecuentes

<!--
ES: Recorrido de la sesión, con ejercicios tras cada bloque. Idea transversal: en un
sistema distribuido, los problemas dejan de ser de código y pasan a ser de comunicación,
fallo y organización. La parte técnica (1–4) y la humana (5) son el mismo proyecto.

EN: Roadmap, with exercises after each block. Cross-cutting idea: in a distributed system,
problems stop being about code and become about communication, failure and organization.
The technical part (1–4) and the human one (5) are the same project.
-->

---

<!-- _class: lead -->

# 1. Comunicación entre servicios

---

## Síncrono vs. asíncrono

La decisión clave no es REST vs. gRPC, sino **síncrono vs. asíncrono**:

- **Síncrono** (petición/respuesta): el llamante **espera** la respuesta (HTTP, gRPC)
  - ✅ simple, resultado inmediato, fácil de depurar
  - ⚠️ **acopla disponibilidad y tiempo**: si el llamado está lento, tú también; las cadenas multiplican latencia y riesgo
- **Asíncrono**: el llamante envía y **sigue con su vida** (broker, cola)
  - ✅ **desacopla en el tiempo**: absorbe picos, el receptor puede estar caído ahora
  - ⚠️ más difícil de razonar, consistencia eventual, necesita broker

> La disponibilidad de una cadena síncrona es el **producto** de las disponibilidades: 5 saltos al 99,9 % ≈ 99,5 %.

<!--
ES: El eje de decisión número uno. Síncrono = acoplamiento temporal: el llamante vive y
muere con el llamado, y las cadenas componen latencia y fallo (la cuenta del blockquote
asusta y es real). Asíncrono = desacoplamiento temporal a cambio de complejidad mental
(orden, "¿cuándo se procesó?") y una pieza más que operar. No es "async siempre", es
"async cuando el negocio lo permita".

EN: The number-one decision axis. Sync = temporal coupling: the caller lives and dies with
the callee, and chains compound latency and failure (the blockquote math is scary and
real). Async = temporal decoupling at the cost of mental complexity (ordering, "when was it
processed?") and one more thing to operate. It's not "async always", it's "async when the
business allows".
-->

---

## Mensajes y eventos

Dentro de lo asíncrono, **dos intenciones distintas**:

- **Comando** (mensaje dirigido): *"haz esto"* → a un receptor concreto, por **cola** (un consumidor procesa cada mensaje). El emisor espera que algo ocurra
- **Evento**: *"esto ha ocurrido"* (`PedidoCreado`) → el emisor **anuncia un hecho** sin saber quién escucha, en un **topic** (pub/sub, N consumidores)

> La **inversión de dependencia** es la gran palanca *event-driven*: facturación, fidelidad y analítica se **suscriben** a `PedidoCreado`; el servicio de pedidos **no cambia**.

El precio: el flujo queda **implícito** (coreografía, Sesión 3) y el contrato pasa a ser el **esquema del evento** — versiónalo como una API.

<!--
ES: Comando vs evento no es sintaxis, es dirección de la dependencia. Comando: yo sé a
quién llamo y quiero que pase algo (cola, 1 consumidor). Evento: anuncio un hecho y me da
igual quién reacciona (topic, N consumidores) — el emisor no conoce a sus consumidores, y
ahí está el desacoplamiento. El coste: el proceso global no está escrito en ningún sitio y
el esquema del evento es ahora un contrato que romper.

EN: Command vs event isn't syntax, it's the direction of the dependency. Command: I know
whom I call and want something to happen (queue, 1 consumer). Event: I announce a fact and
don't care who reacts (topic, N consumers) — the publisher doesn't know its consumers, and
that's the decoupling. The cost: the global process is written nowhere and the event schema
is now a contract you can break.
-->

---

## El broker

La pieza central de lo asíncrono: **persiste y entrega** mensajes, gestiona suscripciones y reintentos. *(RabbitMQ, Kafka, colas cloud — como ejemplos, no temario.)*

Conceptos que hay que conocer al elegir/operar uno:

- **Entrega al-menos-una-vez** → obliga a **idempotencia** (Sesión 3): el mismo mensaje puede llegar dos veces
- **Orden** → a menudo garantizado **solo dentro de una partición/clave**, no global
- **Dead Letter Queue (DLQ)**: a dónde van los mensajes que fallan repetidamente — **hay que monitorizarla** (si nadie la mira, ahí mueren pedidos en silencio)

<!--
ES: El broker no es magia: es infraestructura con garantías concretas que condicionan tu
diseño. "Al-menos-una-vez" es la garantía habitual (exactly-once es casi siempre una
ilusión cara) → idempotencia obligatoria. El orden total no existe gratis: se ordena por
clave. Y la DLQ es el cementerio de mensajes: un sistema sin alerta sobre la DLQ pierde
trabajo sin enterarse. Distinguir broker (transporte) de coordinador de saga (lógica),
como vimos en la Sesión 3.

EN: The broker isn't magic: it's infrastructure with concrete guarantees that shape your
design. "At-least-once" is the usual guarantee (exactly-once is almost always an expensive
illusion) → idempotency required. Total ordering isn't free: you order by key. And the DLQ
is the message graveyard: a system without a DLQ alert loses work silently. Distinguish
broker (transport) from saga coordinator (logic), as in Session 3.
-->

---

## Heurística de elección

| Situación | Estilo natural |
|---|---|
| Necesito la respuesta para continuar (precio, validar) | **Síncrono** |
| Notificar hechos a quien le interese | **Evento** (pub/sub) |
| Trabajo que puede hacerse después (email, PDF) | **Comando** por cola |
| Procesos de negocio entre servicios | **Saga** (eventos u orquestador) |
| Cadena síncrona de 4+ saltos | ⚠️ **Rediseñar** (copias por eventos / componer en el borde) |

> **Síncrono en el borde** (la petición del usuario quiere respuesta), **asíncrono en el interior** siempre que el negocio lo permita. Cada llamada síncrona interna evitada es disponibilidad y latencia ganadas.

<!--
ES: La tabla operativa para el aula. La regla de oro es la del blockquote: el borde habla
síncrono porque el usuario espera; las tripas hablan asíncrono porque ahí ganas
disponibilidad. La última fila es una alarma de diseño: una cadena síncrona larga es un
fallo en cascada esperando a ocurrir (lo vemos en el bloque 2).

EN: The operational table for the room. The golden rule is the blockquote: the edge speaks
sync because the user waits; the guts speak async because that's where you gain
availability. The last row is a design alarm: a long sync chain is a cascading failure
waiting to happen (we cover it in block 2).
-->

---

## API Gateway

La **fachada única** por la que entran los clientes externos (web, móvil, terceros):

```
  web ─┐
 móvil ┼─► API Gateway ─┬─► Pedidos
 3.os ─┘   auth · TLS   ├─► Catálogo
           rate limit   └─► Pagos
           enrutado
```

- **Enrutado** al servicio interno (pieza natural del **Strangler Fig**, Sesión 2)
- **Transversales en un punto:** autenticación, rate limiting, TLS, CORS, caché
- **Adaptación al cliente:** agregar llamadas, traducir; *Backend for Frontend* (BFF) = un gateway por tipo de cliente

> El gateway **adapta y protege; no decide negocio.** Riesgos: cuello de botella, "monolito de enrutado", punto único de fallo → redundante.

<!--
ES: El gateway resuelve los transversales en un solo sitio y es el aliado natural del
Strangler (es la fachada que va redirigiendo del monolito a los servicios nuevos). El
peligro es que se llene de lógica de negocio y se convierta en un nuevo monolito, o que
sea un SPOF: protégelo (redundancia) y manténlo tonto. BFF es la variante cuando móvil y
web necesitan agregaciones distintas.

EN: The gateway solves cross-cutting concerns in one place and is the Strangler's natural
ally (it's the façade that gradually redirects from monolith to new services). The danger
is it fills with business logic and becomes a new monolith, or it's a SPOF: protect it
(redundancy) and keep it dumb. BFF is the variant when mobile and web need different
aggregations.
-->

---

## Descubrimiento de servicios

Con instancias que aparecen y desaparecen (escalado, despliegues, fallos): **"¿dónde está el servicio X ahora?"** necesita respuesta automática.

- **Registro de servicios:** las instancias se **registran al arrancar** y se les hace *health check*; los clientes consultan el registro (client-side) o pasan por un balanceador (server-side)
- En plataformas modernas (**Kubernetes**, como ejemplo) lo resuelve la plataforma (DNS interno + servicios) — conceptualmente lo mismo: un registro con health checks
- Clave para la migración: referenciar **por nombre lógico, nunca por IP/host** → puedes mover, escalar y **estrangular** sin tocar a los consumidores

<!--
ES: Problema nuevo que el monolito no tenía: las direcciones dejan de ser fijas. La
solución conceptual es siempre la misma (registro + health checks), te la dé Kubernetes o
te la montes tú. El take-away para la migración es el último punto: si los consumidores
llaman por nombre lógico, puedes reemplazar la implementación por detrás (¡estrangular!)
sin que se enteren. Acoplarse a una IP es atarse las manos.

EN: A new problem the monolith didn't have: addresses stop being fixed. The conceptual
solution is always the same (registry + health checks), whether Kubernetes gives it to you
or you build it. The migration take-away is the last point: if consumers call by logical
name, you can replace the implementation behind it (strangle!) without them noticing.
Coupling to an IP ties your hands.
-->

---

## Contratos y versionado

En el monolito el compilador vigilaba las interfaces. Entre servicios, la interfaz es un **contrato** (OpenAPI, esquemas de evento) y romperlo rompe a otros equipos **en producción**.

- **Cambios compatibles** (añadir campos opcionales): libres. *Tolerant reader:* ignora lo que no conoces
- **Cambios incompatibles** (quitar/renombrar, cambiar semántica): exigen **versionado** y convivencia (v1 y v2 en paralelo)

```
expand–contract:  1. EXPAND   añade lo nuevo junto a lo viejo
                  2. MIGRAR   consumidores pasan a su ritmo
                  3. CONTRACT retira lo viejo cuando nadie lo usa
```

> Sirve para APIs, **eventos** y esquemas de BD. La independencia de despliegue se sostiene exactamente aquí.

<!--
ES: Aquí se juega la independencia de despliegue. Sin compilador, el contrato es lo único
que impide romper a otros en producción. Compatible = añadir; rompedor = quitar/renombrar/
cambiar semántica (¡la semántica es la traicionera!). Expand–contract es el patrón
universal para cambiar sin coordinar un big-bang: añades, migras, retiras. Las pruebas de
contrato (bloque 4) automatizan saber si rompes a alguien.

EN: Deployment independence is decided here. Without a compiler, the contract is the only
thing preventing you from breaking others in production. Compatible = add; breaking =
remove/rename/change semantics (semantics is the treacherous one!). Expand–contract is the
universal pattern to change without a coordinated big-bang: add, migrate, remove. Contract
tests (block 4) automate knowing whether you break someone.
-->

---

<!-- _class: lead -->

# Ejercicio 4.1
## Elegir el estilo de comunicación

*(ver cuaderno de ejercicios)*

<!--
ES: Pausa para ejercicio. 4.1: para cuatro interacciones, elegir síncrono / evento /
comando por cola y justificar. Claves: A (precio en checkout) = síncrono (respuesta
necesaria, es dinero); B (pedido completado → fidelidad+email+analítica) = evento pub/sub
(reacciones independientes, se añaden sin tocar pedidos); C (PDF de bienvenida, 10 s) =
comando por cola (trabajo diferido dirigido); D (envíos lee nombre/dirección miles de
veces) = copia local por eventos (datos de referencia, Sesión 3). ~15 min.

EN: Exercise break. 4.1: for four interactions, choose sync / event / command-by-queue and
justify. Keys: A (price at checkout) = sync (answer needed, it's money); B (order completed
→ loyalty+email+analytics) = event pub/sub (independent reactions, added without touching
orders); C (welcome PDF, 10 s) = command by queue (deferred directed work); D (shipping
reads name/address thousands of times) = local copy fed by events (reference data, Session
3). ~15 min.
-->

---

<!-- _class: lead -->

# 2. Resiliencia

---

## El fallo es normal

En un monolito, una llamada interna no falla "a medias". En un sistema distribuido, **el fallo parcial es el estado normal**: siempre hay algo reiniciándose, una red lenta, un despliegue a medias.

No se diseña para **evitar** el fallo, sino para que **no se propague** y el sistema se **degrade con elegancia** (catálogo sin recomendaciones ≫ nada).

> **Fallo en cascada:** un servicio lento hace esperar a sus llamantes → agotan hilos/conexiones → tumban a *sus* llamantes → cae la plataforma.

**Lento es peor que caído:** un servicio caído falla rápido; uno lento **retiene recursos** de todos sus llamantes.

<!--
ES: El cambio de mentalidad de todo el bloque. En distribuido el fallo no es excepción, es
clima: diseñas para convivir con él. Dos ideas que repetir: (1) degradación elegante = el
producto decide qué mostrar cuando algo falla; (2) "lento es peor que caído" — es
contraintuitivo y es la causa raíz del 90 % de las caídas en cascada. Casi todos los
patrones siguientes existen para cortar esa cascada.

EN: The mindset shift for the whole block. In distributed systems failure isn't an
exception, it's weather: you design to live with it. Two ideas to repeat: (1) graceful
degradation = the product decides what to show when something fails; (2) "slow is worse
than down" — counterintuitive and the root cause of 90% of cascading outages. Almost every
pattern that follows exists to cut that cascade.
-->

---

## Timeouts, reintentos y backoff

- **Timeout:** nunca esperar indefinidamente. Ajustado a la latencia real (p99 + margen), no "30 s por si acaso" — un timeout largo **retiene recursos**
- **Reintentos:** curan fallos transitorios, pero son peligrosos:
  - Solo lo **idempotente** (¿y si el cobro sí se ejecutó y se perdió la respuesta?)
  - Solo errores **transitorios** (un 503 sí; un 400 nunca será válido)
  - **Presupuesto limitado** (2–3) y cuidado con la **amplificación**: 3× en A × 3× en B → C recibe 9×
- **Backoff exponencial + jitter:** esperar cada vez más (1 s, 2 s, 4 s…) y con aleatoriedad — sin jitter, todos reintentan a la vez (*retry storm*)

<!--
ES: La primera línea de defensa. Timeout deliberado = el recurso más importante que
proteges es tu propio pool de hilos. Los reintentos son un arma de doble filo: sin
idempotencia duplican efectos, sin límite amplifican carga (la cuenta 3×3=9 asusta en el
aula), y sin jitter sincronizan a todos los clientes y rematan al servicio que se
recuperaba. Idempotencia (Sesión 3) es lo que los hace seguros.

EN: The first line of defense. A deliberate timeout = the most important resource you
protect is your own thread pool. Retries are double-edged: without idempotency they
duplicate effects, without a budget they amplify load (the 3×3=9 math lands in the room),
and without jitter they synchronize all clients and finish off the recovering service.
Idempotency (Session 3) is what makes them safe.
-->

---

## Circuit breaker

Evita machacar a un servicio que ya falla (y protege al llamante de esperarlo):

```
        fallos > umbral                  prueba OK
CLOSED ───────────────► OPEN ──────► HALF-OPEN ─────► CLOSED
(todo pasa,        (falla rápido SIN   (deja pasar
 cuenta fallos)     llamar; espera)     unas de prueba)
                                  └── prueba falla ──► OPEN
```

- **Open:** las llamadas **fallan al instante** sin tocar la red → el llamante no malgasta recursos (corta la cascada) y el servicio enfermo respira
- Obliga a la pregunta sana: **¿qué hago cuando está abierto?** (el *fallback*): valor por defecto, dato cacheado, función degradada, mensaje honesto

<!--
ES: El patrón estrella contra la cascada. La idea: si algo falla repetidamente, deja de
llamarlo un rato — beneficia a los dos lados (tú no esperas, él se recupera). Half-open es
el mecanismo de auto-curación: prueba con cuidado antes de volver a confiar. Lo importante
de aula: el circuit breaker te FUERZA a diseñar el fallback, que es decisión de producto,
no de infraestructura. "¿Qué ve el usuario cuando recomendaciones está caído?" es la
pregunta correcta.

EN: The star pattern against cascades. The idea: if something fails repeatedly, stop
calling it for a while — it helps both sides (you don't wait, it recovers). Half-open is
the self-healing mechanism: probe carefully before trusting again. Classroom point: the
circuit breaker FORCES you to design the fallback, which is a product decision, not an
infrastructure one. "What does the user see when recommendations is down?" is the right
question.
-->

---

## Bulkheads (mamparos)

Como los **compartimentos estancos de un barco**: particionar recursos para que el agotamiento de uno no hunda el resto.

- **Pools separados por dependencia:** si recomendaciones se vuelve lento, agota *su* pool — las llamadas a pagos siguen teniendo el suyo
- **Instancias/colas separadas** por tipo de carga: el *batch* no roba capacidad al tráfico interactivo
- **Límites de concurrencia por cliente**

> Sin mamparos, una sola dependencia degradada **agota recursos compartidos** y se lleva por delante funciones que ni la usaban (p. ej. el checkout cae por culpa de recomendaciones).

<!--
ES: El complemento del circuit breaker. Si el breaker corta la propagación "hacia
adelante", el bulkhead evita la propagación "hacia los lados" por recursos compartidos. La
imagen del barco vende sola. El ejemplo del blockquote es exactamente el Ejercicio 4.2: el
checkout no usa recomendaciones, pero comparte pool/instancias → cae con ellas. Aislar el
flujo crítico (compra) es la aplicación más rentable.

EN: The circuit breaker's complement. If the breaker cuts forward propagation, the bulkhead
prevents sideways propagation through shared resources. The ship image sells itself. The
blockquote example is exactly Exercise 4.2: checkout doesn't use recommendations, but
shares pool/instances → it falls with them. Isolating the critical flow (purchase) is the
most profitable application.
-->

---

<!-- _class: lead -->

# Ejercicio 4.2
## Autopsia de un fallo en cascada

*(ver cuaderno de ejercicios)*

<!--
ES: Pausa para ejercicio. 4.2: TiendaTotal — recomendaciones responde en 28 s, la página
de producto la llama síncrona sin timeout (default 60 s), y a los minutos cae todo el
sitio incluido el checkout (que no usa recomendaciones). Tareas: reconstruir la cadena
causal (timeout → pool agotado → recursos compartidos → checkout); por qué los reinicios no
curan (avalancha / retry storm); y prescribir los patrones (timeout, bulkhead, circuit
breaker, fallback, observabilidad). Mensaje: el fallo existe (inevitable) ≠ el fallo se
propaga (evitable). ~20 min.

EN: Exercise break. 4.2: TiendaTotal — recommendations responds in 28 s, the product page
calls it sync with no timeout (default 60 s), and minutes later the whole site falls
including checkout (which doesn't use recommendations). Tasks: reconstruct the causal chain
(timeout → pool exhaustion → shared resources → checkout); why restarts don't cure
(avalanche / retry storm); prescribe the patterns (timeout, bulkhead, circuit breaker,
fallback, observability). Message: failure exists (inevitable) ≠ failure propagates
(avoidable). ~20 min.
-->

---

<!-- _class: lead -->

# 3. Observabilidad

---

## El problema nuevo

En el monolito, un error producía **un** stack trace en **un** log. Ahora una petición atraviesa **6 servicios**: ¿dónde se rompió? ¿dónde se fue el tiempo?

> Sin inversión explícita en observabilidad, cada incidente se convierte en una **partida de Cluedo entre equipos**.

Los tres pilares:

- **Logs** — eventos discretos: *¿qué pasó?*
- **Métricas** — series numéricas agregadas: *¿cómo va el sistema?*
- **Trazas** — el viaje de una petición: *¿dónde se fue el tiempo?*

<!--
ES: El precio oculto de distribuir: pierdes la observabilidad gratis que daba el monolito
(un proceso, un log, un stack trace). La frase del Cluedo resume el dolor real: sin
herramientas, los incidentes degeneran en reuniones de "no soy yo, es tu servicio". Los
tres pilares son complementarios, no alternativos: responden preguntas distintas. La
práctica clave (siguiente slide) es construirlos DURANTE la migración.

EN: The hidden cost of distributing: you lose the free observability the monolith gave (one
process, one log, one stack trace). The Cluedo line captures the real pain: without tools,
incidents degenerate into "not me, it's your service" meetings. The three pillars are
complementary, not alternatives: they answer different questions. The key practice (next
slide) is to build them DURING the migration.
-->

---

## Logs, métricas y trazas

- **Logs:** **centralizados** (nadie hace `ssh` a 40 máquinas), **estructurados** (JSON con campos, no prosa) y con **ID de correlación** en cada línea
- **Métricas:** baratas, para **dashboards y alertas**. Señales doradas: **tráfico, errores, latencia, saturación**. Alerta sobre **síntomas del usuario** (error del checkout), no causas (CPU 80 %)
- **Trazado distribuido:** un **trace ID** se **propaga** en cada llamada; cada servicio registra sus *spans*:

```
trace=abc  ├─────────── checkout 420ms ───────────┤
 gateway     ├ 12ms ┤
 pedidos          ├────── 380ms ──────┤
 pagos              ├─ 310ms ─┤  ← aquí se fue el tiempo
```

> Propaga el contexto **también por colas y eventos**, o la traza se corta donde empieza lo difícil.

<!--
ES: Los tres pilares con su "cómo" práctico. Logs: sin centralizar+estructurar+correlación
son inútiles en distribuido. Métricas: alerta sobre síntomas (lo que sufre el usuario), no
sobre causas (ruido que quema al on-call). Trazado: es el Gantt de una petición — te dice
en qué servicio se fue el tiempo sin adivinar. El error clásico: no propagar el contexto a
través de eventos/colas, y entonces la traza muere justo en la parte asíncrona, que es la
difícil. OpenTelemetry como estándar de ejemplo.

EN: The three pillars with their practical "how". Logs: without centralize+structure+
correlation they're useless in distributed systems. Metrics: alert on symptoms (what the
user suffers), not causes (noise that burns out on-call). Tracing: it's the Gantt of a
request — it tells you in which service time went without guessing. Classic mistake: not
propagating context across events/queues, so the trace dies right at the async part, which
is the hard one. OpenTelemetry as the example standard.
-->

---

<!-- _class: lead -->

# 4. Pruebas y entrega

---

## Estrategia de pruebas en transición

La pirámide clásica sigue valiendo (muchas unitarias, menos integración, **poquísimas E2E**), con dos matices:

- Las **E2E que cruzan muchos servicios** son carísimas y frágiles → redúcelas a *smoke tests* de los flujos de oro. Su papel lo asumen **pruebas de contrato + observabilidad** en producción
- **Tests de caracterización** sobre el monolito: capturan el comportamiento **actual** (correcto o no) de la pieza a extraer, escritos **antes** de extraer

> Son la **red de seguridad** de la extracción y la base contra la que validar el servicio nuevo (el pariente barato del *Parallel Run*).

<!--
ES: La pirámide no cambia, su interpretación sí. Las E2E completas en microservicios son un
pozo de fragilidad: poquísimas, solo los flujos de oro. El arma específica de la migración
son los tests de caracterización: NO comprueban que el código esté bien, comprueban que el
servicio nuevo se comporta IGUAL que el viejo (incluidas sus rarezas) — exactamente lo que
necesitas para extraer sin cambiar comportamiento. Es el Parallel Run de los pobres.

EN: The pyramid doesn't change, its interpretation does. Full E2E in microservices is a
fragility pit: very few, golden flows only. The migration-specific weapon is
characterization tests: they DON'T check the code is correct, they check the new service
behaves the SAME as the old one (quirks included) — exactly what you need to extract without
changing behavior. It's the poor man's Parallel Run.
-->

---

## Pruebas de contrato

Verifican que proveedor y consumidor están de acuerdo **sin levantar los dos sistemas juntos**:

- El **consumidor** declara qué usa (*"llamo a `GET /clientes/{id}` y uso `nombre` y `email`"*)
- Esa expectativa se verifica **contra el proveedor en su CI** (*consumer-driven contracts*; Pact, como ejemplo)
- Si el proveedor intenta quitar `email`, **su build falla antes de desplegar**, señalando qué consumidor se rompería

> Es la **malla de seguridad que sustituye al compilador** del monolito — la que permite desplegar de forma independiente con confianza.

<!--
ES: La respuesta automatizada a "¿romperé a alguien?" del bloque 1. La magia: no necesitas
un entorno con los dos servicios; el consumidor publica sus expectativas y el CI del
proveedor las verifica. Así recuperas, en distribuido, lo que el compilador te daba gratis
en el monolito: te avisa antes de desplegar y te dice a quién romperías. Sin esto, la
independencia de despliegue es una apuesta a ciegas.

EN: The automated answer to "will I break someone?" from block 1. The magic: you don't need
an environment with both services; the consumer publishes its expectations and the
provider's CI verifies them. So you recover, in distributed systems, what the compiler gave
you for free in the monolith: it warns before deploy and tells you whom you'd break.
Without this, deployment independence is a blind bet.
-->

---

## CI/CD e independencia de despliegue

- **Un pipeline por servicio:** cada uno se construye, prueba y despliega **solo**. Si liberar exige "el pipeline de la release" global → no hay independencia (*release train* = síntoma del monolito distribuido, Sesión 1)
- **Separar despliegue de liberación:** desplegar código inactivo y activarlo por **configuración** (*feature flags*, canary) — ya visto en *Branch by Abstraction*
- Durante la migración, **el monolito también** necesita buen pipeline (se redesplegará sin parar al quitarle piezas) — invertir en su CI/CD es el **andamiaje de la obra**

<!--
ES: La independencia de despliegue es el objetivo de toda la migración, y aquí se hace
operativa. Un pipeline por servicio o no hay autonomía. Separar deploy de release (feature
flags) desacopla "poner código en producción" de "activar la función" — clave para
desplegar sin miedo. Y el recordatorio práctico: no descuides el pipeline del monolito; va
a ser la pieza que más se toca durante años, invertir ahí no es trabajo perdido.

EN: Deployment independence is the goal of the whole migration, and here it becomes
operational. One pipeline per service or there's no autonomy. Separating deploy from release
(feature flags) decouples "ship code" from "enable the feature" — key to deploy without
fear. And the practical reminder: don't neglect the monolith's pipeline; it'll be the most-
touched piece for years, investing there isn't wasted work.
-->

---

<!-- _class: lead -->

# Ejercicio 4.3
## ¿Romperá a los consumidores?

*(ver cuaderno de ejercicios)*

<!--
ES: Pausa para ejercicio. 4.3: el equipo de clientes propone 4 cambios en GET /clientes/
{id}; clasificar compatible/rompedor y hacerlo seguro. A (añadir campo) = compatible; B
(renombrar email) = rompedor → expand–contract; C (telefono nunca-nulo pasa a nullable) =
rompedor TRAICIONERO (cambio de semántica que ningún validador pilla); D (cambiar el
esquema del evento) = rompedor (el evento es contrato igual que la API). Extra: pruebas de
contrato dirigidas por el consumidor cazan B y D antes de desplegar. ~15-20 min.

EN: Exercise break. 4.3: the customers team proposes 4 changes to GET /clientes/{id};
classify compatible/breaking and make it safe. A (add field) = compatible; B (rename email)
= breaking → expand–contract; C (never-null phone becomes nullable) = TREACHEROUS breaking
(semantic change no validator catches); D (change event schema) = breaking (the event is a
contract like the API). Extra: consumer-driven contract tests catch B and D before deploy.
~15-20 min.
-->

---

<!-- _class: lead -->

# 5. Aspectos organizativos

---

## La ley de Conway

> *"Las organizaciones diseñan sistemas que copian su estructura de comunicación."* — M. Conway, 1968

No es metáfora, es una **fuerza física** de la ingeniería:

```
 3 equipos:   Frontend │ Backend │ DBA
                     ▼ (Conway empuja)
 sale esto:    UI/web  │ web-api │ datos     (capas, NO dominios)
```

- Migrar la arquitectura **sin tocar la organización** → monolito distribuido
- **Conway inverso:** diseña primero los equipos que *reflejan* la arquitectura deseada (equipos por **dominio**) y deja que la fuerza empuje a favor

<!--
ES: La idea organizativa más importante del curso. Conway no es un consejo, es física: la
arquitectura SIEMPRE acaba reflejando la estructura de comunicación de los equipos. Si
tienes equipos por capa, saldrán servicios por capa por mucho que el diagrama diga
"dominios". La maniobra inversa es la palanca: cambia los equipos ANTES (a dominios) y la
misma fuerza que te perjudicaba ahora te ayuda. Sin esto, la migración técnica fracasa.

EN: The most important organizational idea of the course. Conway isn't advice, it's
physics: architecture ALWAYS ends up mirroring the teams' communication structure. If you
have layer teams, you'll get layer services no matter what the diagram says "domains". The
inverse maneuver is the lever: change teams FIRST (to domains) and the same force that hurt
you now helps. Without this, the technical migration fails.
-->

---

## Team Topologies

Vocabulario de Skelton & Pais para lo que una migración necesita:

- **Stream-aligned:** equipo alineado a un dominio (≈ bounded context), dueño de extremo a extremo. El tipo **por defecto**; los demás existen para servirle
- **Platform:** ofrece la plataforma interna (CI/CD, observabilidad, plantillas) **como producto** autoservicio → baja la **carga cognitiva**
- **Enabling:** especialistas **temporales** que enseñan (DDD, contratos) y se retiran
- **Complicated-subsystem:** custodia una pieza de saber raro (motor de tarificación)

> **Carga cognitiva** como límite de diseño: un equipo no debe poseer más sistema del que **cabe en su cabeza**.

<!--
ES: Le da nombres útiles a la estructura objetivo. El 90 % deben ser stream-aligned (por
dominio, autónomos). El platform team es el que hace escalable la migración: sin plataforma
como producto, cada equipo reinventa Kubernetes y te ahogas en infraestructura. Enabling =
andamiaje humano temporal (acompaña y se va). Y la carga cognitiva es un criterio de
tamaño tan válido como el acoplamiento: si no cabe en la cabeza del equipo, es demasiado.

EN: It gives useful names to the target structure. 90% should be stream-aligned (by domain,
autonomous). The platform team makes the migration scalable: without platform-as-a-product,
every team reinvents Kubernetes and you drown in infrastructure. Enabling = temporary human
scaffolding (accompanies and leaves). And cognitive load is a sizing criterion as valid as
coupling: if it doesn't fit in the team's head, it's too much.
-->

---

## Propiedad de los servicios

- **Cada servicio tiene exactamente un equipo dueño** (un equipo puede tener varios servicios; un servicio **no** puede tener dos dueños — la propiedad compartida diluye responsabilidad y calidad)
- Propiedad = ciclo de vida completo: construir, desplegar, **operar**, llevar el busca (*you build it, you run it*)
- **¿Y el monolito?** Peligro de "tierra de nadie" mientras siga sirviendo a la mayoría. Opciones: un equipo dueño con la misión de **encogerlo**, o propiedad por zonas alineada con los futuros contextos

> Lo inaceptable es que el monolito **no tenga dueño**.

<!--
ES: Propiedad única o no hay responsabilidad. "You build it, you run it" cierra el bucle de
incentivos: quien sufre el busca a las 3am escribe mejor código. El punto sensible de la
migración es el monolito: como sigue funcionando, nadie lo quiere, y se pudre. Hay que
asignarle dueño explícito con la misión de encogerlo — o trocearlo por zonas entre los
equipos de dominio. Tierra de nadie = muerte lenta.

EN: Single ownership or there's no accountability. "You build it, you run it" closes the
incentive loop: whoever carries the 3am pager writes better code. The migration's sensitive
point is the monolith: since it still works, nobody wants it, and it rots. It needs an
explicit owner with the mission to shrink it — or carve it by zones across the domain teams.
No-man's-land = slow death.
-->

---

## La migración como cambio organizativo

La migración técnica y la organizativa son **el mismo proyecto**:

- **Reorganizar** equipos hacia dominios (Conway inverso) y crear la **plataforma**
- **Formar:** operar distribuido es una habilidad nueva (on-call, debugging distribuido, diseño de contratos)
- **Gestionar la resistencia:** el experto del monolito que "pierde su feudo" necesita un papel valioso — es **quien más sabe de los dominios**
- **Patrocinio de dirección** imprescindible: una migración de años no sobrevive como proyecto de guerrilla

<!--
ES: El cierre del bloque humano. Reorganizar no es el "después" de migrar, es parte de
migrar. Tres frentes que suelen olvidarse: formación (operar distribuido no se sabe por
ciencia infusa), gestión del cambio (el gurú del monolito puede ser tu mejor aliado o tu
peor saboteador — dale un papel), y patrocinio (sin cobertura ejecutiva, la migración
muere en la primera reorganización de prioridades). Es tan importante como cualquier
patrón técnico.

EN: The close of the human block. Reorganizing isn't the "after" of migrating, it's part of
migrating. Three fronts often forgotten: training (operating distributed isn't innate),
change management (the monolith guru can be your best ally or worst saboteur — give them a
role), and sponsorship (without executive cover, the migration dies at the first priority
reshuffle). As important as any technical pattern.
-->

---

<!-- _class: lead -->

# Ejercicio 4.4
## Conway en acción

*(ver cuaderno de ejercicios)*

<!--
ES: Pausa para ejercicio. 4.4: BancoÁgil — equipos por capa (Frontend, Backend, DBA, QA) y
arquitectura objetivo por dominios (cuentas, pagos, préstamos, onboarding). El plan del CTO
mantiene los equipos como están. Tarea 1: predecir con Conway qué saldrá (servicios con
nombre de dominio pero flujo que cruza 3-4 equipos; DBA = BD compartida de facto; QA de
release conjunta = release train). Tarea 2: estructura alternativa (4 stream-aligned por
dominio + 1 platform que absorbe DBAs + enabling temporal con QA seniors; monolito con
dueño explícito). ~20 min.

EN: Exercise break. 4.4: BancoÁgil — layer teams (Frontend, Backend, DBA, QA) and a target
architecture by domains (accounts, payments, loans, onboarding). The CTO's plan keeps teams
as-is. Task 1: predict with Conway what emerges (services with domain names but a flow
crossing 3-4 teams; DBA = de-facto shared DB; joint-release QA = release train). Task 2:
alternative structure (4 stream-aligned by domain + 1 platform absorbing DBAs + temporary
enabling with senior QA; monolith with explicit owner). ~20 min.
-->

---

<!-- _class: lead -->

# 6. Síntesis y hoja de ruta

---

## La hoja de ruta tipo

**Fase 0 — Decidir y preparar** *(S1, S3, S4)*
Objetivo + métricas con línea base + criterio de parada · bosquejo de bounded contexts · equipos hacia dominios + plataforma + CI/CD del monolito

**Fase 1 — Primeras extracciones** *(S2, S3)*
1–2 candidatos por valor×facilidad · Strangler + patrón que toque + **ACL en toda frontera** · observabilidad y contratos **desde el primer servicio** · **borrar** lo extraído · medir

**Fase 2 — Industrializar**
Candidatos más centrales + plantillas de servicio · **datos en serio** (propiedad, esquemas, sagas/outbox) · resiliencia por defecto

**Fase 3 — Converger y parar**
Genérico → comprar/SaaS · lo estable y de bajo valor **se queda** (documentado) · revisar criterio de parada → **declarar el final**

<!--
ES: El destilado de las 4 sesiones en una secuencia. Las ideas que no pueden faltar:
medir antes de empezar (sin línea base no sabes si fue bien); organización ANTES de las
extracciones (Conway); observabilidad y contratos desde el PRIMER servicio (no "para
después"); borrar el legacy en cada paso (o pagas doble para siempre); y un criterio de
parada explícito — la migración termina deliberadamente, no se difumina. "Parar" incluye
dejar a propósito lo que no compense migrar.

EN: The distillate of all 4 sessions in one sequence. Must-have ideas: measure before
starting (no baseline, no way to know it went well); organization BEFORE extractions
(Conway); observability and contracts from the FIRST service (not "for later"); delete the
legacy at each step (or you pay double forever); and an explicit stopping criterion — the
migration ends deliberately, it doesn't fade out. "Stop" includes deliberately keeping what
isn't worth migrating.
-->

---

## Errores frecuentes (el top 10)

1. Migrar sin **objetivo medible** ("modernizar") → sin prioridad ni parada
2. Dividir por **capas o entidades** → monolito distribuido
3. Dividir el código y **no los datos** → acoplamiento subterráneo
4. **Big-bang** de un sistema grande y vivo → objetivo móvil, cero valor
5. Cortar **demasiado fino** demasiado pronto → fusiones carísimas
6. Olvidar **borrar** el legacy → híbrido eterno, coste doble
7. Cadenas síncronas largas sin timeouts/breakers → **cascada**
8. Asumir entrega **exactamente-una-vez** → duplicados sin idempotencia
9. Observabilidad y contratos "para después" → incidentes indepurables
10. Migrar la arquitectura **sin** la organización → Conway gana

<!--
ES: El checklist de la vergüenza: cada error mapea a una lección del curso. Útil como
autoevaluación ("¿en cuántos estamos cayendo?"). Los tres más mortales en la práctica: el
3 (código sin datos = la migración que parece hecha y no lo está), el 6 (no borrar = pagar
dos sistemas para siempre) y el 10 (ignorar Conway = todo lo técnico se deshace solo). Si
solo recuerdan diez frases del curso, que sean estas diez en negativo.

EN: The wall of shame: each mistake maps to a course lesson. Useful as self-assessment
("how many are we making?"). The three deadliest in practice: #3 (code without data = the
migration that looks done and isn't), #6 (not deleting = paying for two systems forever),
and #10 (ignoring Conway = all the technical work undoes itself). If they remember only ten
sentences from the course, make it these ten in the negative.
-->

---

<!-- _class: lead -->

# Ejercicio 4.5
## Síntesis: la hoja de ruta de SaludPlus

*(ejercicio final integrador — usa las 4 sesiones)*

<!--
ES: Ejercicio final integrador. SaludPlus (clínicas, monolito de 14 años): facturación a
aseguradoras que cambia cada mes (release mensual completa), citas que se saturan los
lunes, historiales clínicos regulados y sin cambios en 3 años, y quieren lanzar
telemedicina. Tarea: esbozar la hoja de ruta (objetivo+métricas, qué se extrae y en qué
orden y con qué patrón, datos, organización, y qué NO se migra). Claves: facturación
primero (máximo valor, parallel run); telemedicina como Bubble Context (greenfield, core);
citas quizá re-platforming; historiales NO se migran (estable+regulado=riesgo alto/valor
nulo); informes por eventos. ~30-40 min, puesta en común amplia. Cierra el curso.

EN: Final integrative exercise. SaludPlus (clinics, 14-year monolith): insurer billing that
changes monthly (full monthly release), appointments that saturate on Mondays, regulated
clinical records unchanged in 3 years, and they want to launch telemedicine. Task: sketch
the roadmap (goal+metrics, what to extract and in what order and with which pattern, data,
organization, and what NOT to migrate). Keys: billing first (max value, parallel run);
telemedicine as Bubble Context (greenfield, core); appointments maybe re-platforming;
records NOT migrated (stable+regulated = high risk/zero value); reports via events. ~30-40
min, broad discussion. Closes the course.
-->

---

## Resumen de la sesión

- **Comunicación:** síncrono solo cuando la respuesta es imprescindible; eventos para hechos, colas para trabajo diferido; gateway como fachada; contratos versionados (**expand–contract** + pruebas de contrato)
- **Resiliencia:** el fallo parcial es lo normal; timeouts, retries idempotentes con backoff+jitter, **circuit breakers** con fallback, bulkheads — que el fallo **no se propague**. *Lento es peor que caído*
- **Observabilidad:** logs estructurados centralizados, métricas sobre síntomas, trazado con propagación de contexto — **durante** la migración, no después
- **Organización:** **Conway gana siempre** — equipos por dominio, plataforma como producto, propiedad única, monolito nunca sin dueño
- **Hoja de ruta:** medir → extraer por valor×facilidad con ACLs → industrializar → converger y **parar**

<!--
ES: El destilado de la sesión y casi del curso. Si solo se llevan cuatro ideas: (1) async
en el interior compra disponibilidad; (2) diseña para que el fallo no se propague, no para
evitarlo; (3) observabilidad y contratos son inversión de día uno, no de "después"; (4)
Conway es física: sin reorganizar, lo técnico se deshace. Y la migración termina
deliberadamente.

EN: The distillate of the session and almost of the course. If they take away four ideas:
(1) async on the inside buys availability; (2) design so failure doesn't propagate, not to
avoid it; (3) observability and contracts are day-one investment, not "later"; (4) Conway
is physics: without reorganizing, the technical work undoes itself. And the migration ends
deliberately.
-->

---

<!-- _class: lead -->

# ¿Preguntas?

**Fin de la formación** — Migración de Monolitos a Microservicios
*Decidir · Extraer con patrones · Encontrar límites y partir datos · Operar y organizar*

🌱 *¡Gracias y buenas migraciones!*

<!--
ES: Cierre de las cuatro sesiones. Abrir turno de preguntas y recordar dónde está todo el
material y los cuadernos. Puente narrativo final: el curso ha recorrido el ciclo completo
— por qué y cuándo migrar (S1), cómo mover con seguridad (S2), dónde cortar y qué hacer con
los datos (S3), y cómo operar y organizar el resultado (S4). El mensaje de despedida:
migrar es una decisión de negocio ejecutada con disciplina técnica y organizativa, y que
termina a propósito. Agradecer y cerrar.

EN: Close of the four sessions. Open the floor and remind them where all the material and
workbooks are. Final narrative bridge: the course has covered the full cycle — why and when
to migrate (S1), how to move safely (S2), where to cut and what to do with data (S3), and
how to operate and organize the result (S4). Farewell message: migrating is a business
decision executed with technical and organizational discipline, and it ends on purpose.
Thank them and close.
-->
