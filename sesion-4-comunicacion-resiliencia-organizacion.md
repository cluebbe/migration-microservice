# Sesión 4 — Comunicación, resiliencia y organización

**Duración:** 3 h · **Objetivo:** hacer que el sistema resultante sea robusto de operar y esté alineado con los equipos que lo mantienen; cerrar la formación con una hoja de ruta de migración.

---

## 1. Comunicación entre servicios

### 1.1 Síncrona vs. asíncrona

La decisión de comunicación más importante no es REST vs. gRPC, sino **síncrono vs. asíncrono**:

- **Síncrono (petición/respuesta):** el llamante envía y **espera** la respuesta para continuar (HTTP/REST, gRPC). 
  - ✅ Simple de entender, el resultado llega de inmediato, fácil de depurar paso a paso.
  - ⚠️ **Acopla disponibilidad y tiempo:** si el llamado está caído o lento, el llamante también; las cadenas de llamadas multiplican la latencia y el riesgo (la disponibilidad de la cadena es el producto de las disponibilidades).
- **Asíncrono:** el llamante envía y **sigue con su vida**; la comunicación pasa por un intermediario (broker, cola) o se resuelve más tarde.
  - ✅ **Desacopla en el tiempo:** el receptor puede estar caído ahora y procesar después; absorbe picos (la cola amortigua); permite añadir consumidores sin tocar al emisor.
  - ⚠️ Más difícil de razonar (¿cuándo se procesó?, ¿en qué orden?), consistencia eventual, necesita broker (una pieza más que operar), depuración no lineal.

### 1.2 Mensajes y eventos (message-driven / event-driven)

Dentro de lo asíncrono conviene distinguir dos intenciones:

- **Comando (mensaje dirigido):** "haz esto" — va dirigido a un receptor concreto, normalmente por una **cola** (un solo consumidor procesa cada mensaje). El emisor *espera que algo ocurra*.
- **Evento:** "esto ha ocurrido" (`PedidoCreado`) — el emisor **anuncia un hecho** sin saber ni importarle quién escucha; se publica en un **topic** (publish/subscribe, N consumidores). La dependencia se invierte: el emisor no conoce a sus consumidores.

Esta inversión es la gran palanca de desacoplamiento de las arquitecturas **event-driven**: cuando facturación, fidelidad y analítica quieren reaccionar a un pedido, el servicio de pedidos no cambia — solo se suscriben. El precio: el flujo del proceso queda implícito (lo vimos en coreografía de sagas, Sesión 3) y el contrato pasa a ser **el esquema del evento**, que hay que versionar con el mismo cuidado que una API.

**El broker** (RabbitMQ, Kafka, colas cloud — como ejemplos, no como temario) es la pieza central: persiste y entrega mensajes, gestiona suscripciones y reintentos. Conceptos que conviene conocer al elegir/operar uno: entrega al-menos-una-vez (→ idempotencia, Sesión 3), orden (a menudo solo garantizado por partición/clave), y *dead letter queues* (a dónde van los mensajes que fallan repetidamente — deben monitorizarse).

### 1.3 Heurística de elección

| Situación | Estilo natural |
|---|---|
| Necesito la respuesta para continuar (consultar precio, validar) | Síncrono |
| Notificar hechos a quien le interese | Evento (pub/sub) |
| Trabajo que puede hacerse después (enviar email, generar PDF) | Comando por cola |
| Procesos de negocio entre servicios | Saga (eventos u orquestador) |
| Cadena síncrona de 4+ saltos | ⚠️ Rediseñar: copias locales por eventos o componer en el borde |

Regla práctica: **síncrono en el borde** (la petición del usuario quiere respuesta), **asíncrono en el interior** siempre que el negocio lo permita — cada llamada síncrona interna que se evita es disponibilidad y latencia que se gana.

### 1.4 API Gateway

El **API Gateway** es la fachada única por la que los clientes externos (web, móvil, terceros) acceden al sistema:

- **Enrutado** hacia el servicio interno adecuado (es, de hecho, la pieza natural para el Strangler Fig de la Sesión 2).
- **Transversales en un solo punto:** autenticación/autorización de entrada, rate limiting, TLS, logging de acceso, CORS, caché.
- **Adaptación al cliente:** agregar varias llamadas internas, traducir protocolos; la variante *Backend for Frontend* (BFF) da un gateway adaptado a cada tipo de cliente (uno para móvil, otro para web).

Riesgos: convertirse en cuello de botella o en "monolito de enrutado" lleno de lógica de negocio (el gateway adapta y protege; **no decide negocio**), y ser un punto único de fallo (debe ser redundante).

### 1.5 Descubrimiento de servicios

Con instancias que aparecen y desaparecen (escalado, despliegues, fallos), "¿dónde está el servicio X ahora mismo?" necesita respuesta automática:

- **Registro de servicios:** las instancias se registran al arrancar (y se les hace *health check*); los clientes consultan el registro (client-side) o pasan por un balanceador que lo consulta (server-side).
- En plataformas modernas (Kubernetes, como ejemplo) el descubrimiento viene resuelto por la plataforma (DNS interno + servicios); conceptualmente sigue siendo lo mismo: un registro con health checks.
- Idea clave para la migración: los servicios se referencian **por nombre lógico**, nunca por IP/host concretos — eso permite mover, escalar y reemplazar (¡estrangular!) sin tocar a los consumidores.

### 1.6 Contratos y versionado

En el monolito, el compilador vigilaba las interfaces internas. Entre servicios, la interfaz es un **contrato** (OpenAPI para REST, esquemas de eventos…) y romperlo rompe a otros equipos *en producción*:

- **Cambios compatibles** (añadir campos opcionales, nuevas operaciones): libres. Regla de tolerancia del consumidor (*tolerant reader*): ignora lo que no conoces, no valides más de lo que usas.
- **Cambios incompatibles** (quitar/renombrar campos, cambiar semántica): exigen **versionado** y convivencia de versiones (v1 y v2 en paralelo) hasta que los consumidores migren — la independencia de despliegue se sostiene exactamente aquí.
- **Expand–contract** (el patrón universal de cambio compatible): *expandir* (añadir lo nuevo junto a lo viejo) → migrar consumidores a su ritmo → *contraer* (retirar lo viejo cuando nadie lo usa). Sirve para APIs, eventos y también esquemas de BD.
- Las **pruebas de contrato** verifican automáticamente que no rompes a tus consumidores (ver §4).

---

## 2. Resiliencia

### 2.1 El fallo es normal

En un monolito, una llamada interna no falla "a medias". En un sistema distribuido, **el fallo parcial es el estado normal**: en todo momento alguna instancia está reiniciándose, alguna red va lenta, algún despliegue está a medias. No se diseña para evitar el fallo; se diseña para que el fallo **no se propague** y el sistema se **degrade con elegancia** (mostrar el catálogo sin recomendaciones es infinitamente mejor que no mostrar nada).

El patrón de catástrofe a evitar es el **fallo en cascada**: un servicio lento hace esperar a sus llamantes, que agotan sus hilos/conexiones, que hacen esperar a *sus* llamantes… y un servicio degradado tumba la plataforma entera. Casi todos los patrones siguientes existen para cortar esa cadena. Atención: **lento es peor que caído** — un servicio caído falla rápido; uno lento retiene recursos de todos sus llamantes.

### 2.2 Timeouts, reintentos y backoff

- **Timeout:** nunca esperar indefinidamente. Todo cliente de red necesita un timeout explícito y deliberado (los defaults suelen ser absurdos: infinito o minutos). Ajustado a la latencia real esperada (p. ej. p99 + margen), no un "30 segundos por si acaso" — un timeout largo es un retenedor de recursos.
- **Reintentos:** los fallos transitorios (un paquete perdido, una instancia reiniciándose) se curan reintentando. Pero los reintentos son peligrosos:
  - Solo reintentar lo **idempotente** (¿y si el cobro *sí* se ejecutó y solo se perdió la respuesta? — reintentar duplica el cobro). La idempotencia (Sesión 3) es lo que hace seguros los reintentos.
  - Solo errores **transitorios** (un 503 sí; un 400 no — reintentar una petición inválida nunca la hará válida).
  - **Presupuesto limitado** (2–3 intentos), y cuidado con la **amplificación**: si A reintenta 3× y B reintenta 3× la llamada a C, C recibe 9× — los reintentos en cada capa se multiplican.
- **Backoff exponencial con jitter:** esperar entre reintentos cada vez más (1 s, 2 s, 4 s…) y con aleatoriedad (jitter). Sin jitter, todos los clientes que fallaron a la vez reintentan a la vez: la *retry storm* remata al servicio que intentaba recuperarse.

### 2.3 Circuit breaker

El **cortacircuitos** evita machacar a un servicio que ya está fallando (y proteger al llamante de esperarlo):

```
        fallos > umbral                  prueba OK
CLOSED ────────────────► OPEN ──────► HALF-OPEN ─────► CLOSED
(normal: las llamadas    (falla rápido SIN     (deja pasar
 pasan; se cuentan        llamar; tras un       unas pocas
 los fallos)              tiempo → half-open)   de prueba)
                                    └── prueba falla ──► OPEN
```

- **Closed:** todo pasa; se monitoriza la tasa de fallos.
- **Open:** superado el umbral, las llamadas **fallan inmediatamente** sin tocar la red. Doble beneficio: el llamante no malgasta recursos esperando (corta la cascada) y el servicio enfermo recibe un respiro para recuperarse.
- **Half-open:** pasado un tiempo, se dejan pasar llamadas de prueba; si van bien se cierra, si no, vuelve a abrirse.

El circuit breaker obliga a la pregunta sana: **¿qué hago cuando está abierto?** (el *fallback*): valor por defecto, dato cacheado, funcionalidad degradada, mensaje honesto al usuario. Esa respuesta es diseño de producto, no solo de infraestructura.

### 2.4 Bulkheads (mamparos)

Como los compartimentos estancos de un barco: **particionar los recursos** para que el agotamiento de uno no hunda el resto. Ejemplos: pools de conexiones/hilos separados por dependencia (si el servicio de recomendaciones se vuelve lento, agota *su* pool y las llamadas a pagos siguen teniendo el suyo); instancias o colas separadas para tipos de carga distintos (el procesamiento batch no roba capacidad al tráfico interactivo); límites de concurrencia por cliente.

### 2.5 Idempotencia (recapitulación transversal)

Aparece en todas partes porque es lo que hace **seguros** los mecanismos anteriores: reintentos (el mismo request dos veces), entrega al-menos-una-vez de los brokers, compensaciones de sagas repetidas, relays de outbox. Técnicas: claves de idempotencia en las peticiones (el cliente genera un ID; el servidor deduplica), restricciones únicas por clave de negocio, operaciones naturalmente idempotentes (asignar en vez de incrementar). **Regla de diseño: toda operación expuesta entre servicios debería ser idempotente o declarar explícitamente que no lo es.**

---

## 3. Observabilidad

### 3.1 El problema nuevo

En el monolito, un error producía **un** stack trace en **un** log. Ahora una petición atraviesa 6 servicios: ¿dónde se rompió?, ¿dónde se fue el tiempo? Sin inversión explícita en observabilidad, la migración convierte cada incidente en una partida de Cluedo entre equipos. Los tres pilares:

### 3.2 Logs, métricas y trazas

- **Logs:** el registro de eventos discretos. En microservicios exigen: **centralización** (todos los servicios envían a un sistema común consultable — nadie hace `ssh` a 40 máquinas), **formato estructurado** (JSON con campos, no prosa: para poder filtrar y correlacionar) y **IDs de correlación** en cada línea (ver trazas).
- **Métricas:** series numéricas agregadas (tasa de peticiones, errores, latencias —p50/p95/p99—, profundidad de colas, estado de circuit breakers). Baratas de almacenar, ideales para **dashboards y alertas**. Las "señales doradas": tráfico, errores, latencia, saturación. Alertar sobre **síntomas que afectan al usuario** (tasa de error del checkout), no sobre causas internas (CPU al 80 %).
- **Trazado distribuido:** reconstruye el viaje de **una petición concreta** a través de los servicios. Cada petición recibe un **trace ID** que se **propaga** en cada llamada (cabeceras HTTP, metadatos de mensajes); cada servicio registra sus tramos (*spans*) con duración. El resultado es el diagrama de Gantt de la petición: dónde se fue el tiempo, qué servicio falló, qué se llamó en paralelo o en secuencia. (OpenTelemetry, como ejemplo de estándar.) La propagación del contexto debe incluirse **también a través de colas y eventos**, o las trazas se cortan justo donde empieza lo difícil.

Práctica esencial para la migración: **construir la observabilidad antes o durante las primeras extracciones, no después** — los primeros servicios extraídos son los que más vigilancia necesitan (¡y un parallel run sin métricas no es nada!).

---

## 4. Pruebas y entrega durante la migración

### 4.1 Estrategia de pruebas para un sistema en transición

La pirámide clásica (muchas unitarias, menos de integración, pocas E2E) sigue valiendo, con dos matices propios de microservicios y migración:

- Las pruebas **E2E que atraviesan muchos servicios** se vuelven carísimas y frágiles (entornos completos, datos coordinados, fallos intermitentes). Deben ser poquísimas (smoke tests de los flujos de oro) — su papel lo asumen las pruebas de contrato + la observabilidad en producción.
- Durante la migración hay un arma especial: **tests de caracterización** sobre el monolito — tests que capturan el comportamiento *actual* (sea correcto o no) de la pieza a extraer, escritos *antes* de extraer. Son la red de seguridad de la extracción, y la base contra la que validar el servicio nuevo (el pariente barato del Parallel Run).

### 4.2 Pruebas de contrato

Verifican que proveedor y consumidor están de acuerdo **sin levantar los dos sistemas juntos**:

- El **consumidor** declara qué usa del contrato ("llamo a `GET /clientes/{id}` y uso los campos `nombre` y `email`").
- Esa expectativa se verifica automáticamente **contra el proveedor** en su CI (*consumer-driven contracts*; Pact, como ejemplo).
- Resultado: si el proveedor intenta quitar el campo `email`, su build falla *antes de desplegar*, señalando qué consumidor se rompería. Es la malla de seguridad que sustituye al compilador del monolito y la que permite desplegar de forma independiente con confianza.

### 4.3 CI/CD e independencia de despliegue (conceptual)

- **Un pipeline por servicio:** cada servicio se construye, prueba y despliega solo. Si para liberar un servicio hay que ejecutar "el pipeline de la release" global, no hay independencia (el release train es el síntoma C del monolito distribuido, Sesión 1).
- **Separar despliegue de liberación (release):** desplegar código inactivo y activarlo por configuración (feature flags) — ya lo vimos como pieza clave de Branch by Abstraction; canary releases y despliegues graduales son la misma idea a nivel de tráfico.
- Durante la migración, el monolito **también** necesita un buen pipeline (se va a redesplegar constantemente a medida que se le quitan piezas); invertir en CI/CD del monolito no es trabajo perdido, es el andamiaje de la obra.

---

## 5. Aspectos organizativos

### 5.1 La ley de Conway

> "Las organizaciones diseñan sistemas que copian su estructura de comunicación." — M. Conway, 1968

No es una metáfora; es una fuerza física de la ingeniería: si hay tres equipos (frontend, backend, BD), saldrá una arquitectura de tres capas — independientemente de lo que diga el diagrama objetivo. Consecuencias para la migración:

- Migrar la arquitectura **sin tocar la organización** produce el monolito distribuido: equipos por capas → servicios por capas → cada cambio cruza todos los equipos.
- La **maniobra de Conway inversa**: diseñar primero la estructura de equipos que *refleje* la arquitectura deseada (equipos por dominio de negocio) y dejar que la fuerza de Conway empuje a favor. Equipos alineados con bounded contexts → servicios alineados con bounded contexts.

### 5.2 Team Topologies (vocabulario útil)

El modelo de Skelton & Pais da nombres a lo que una migración necesita:

- **Stream-aligned teams:** equipos alineados a un flujo de valor/dominio (≈ bounded context) con propiedad de extremo a extremo. Son el tipo por defecto; los demás existen para servirles.
- **Platform team:** ofrece la plataforma interna (CI/CD, observabilidad, plantillas de servicio, infraestructura) **como producto** con autoservicio — reduce la **carga cognitiva** de los stream-aligned (sin plataforma, cada equipo reinventa Kubernetes y la migración se ahoga en infraestructura).
- **Enabling team:** especialistas temporales que enseñan (p. ej. un equipo que acompaña a los demás en DDD o en pruebas de contrato durante la migración) y se retiran.
- **Complicated-subsystem team:** custodia una pieza que exige saber raro (el motor de tarificación matemático).

Y la noción de **carga cognitiva** como límite de diseño: un equipo no debe poseer más sistema del que cabe en su cabeza — criterio tan válido como el acoplamiento para decidir cuántos servicios y de qué tamaño.

### 5.3 Propiedad de los servicios

- **Cada servicio tiene exactamente un equipo dueño** (un equipo puede poseer varios servicios; un servicio no puede tener dos dueños — la propiedad compartida diluye la responsabilidad y la calidad).
- Propiedad = ciclo de vida completo: construir, desplegar, operar, llevar el busca ("*you build it, you run it*") — es lo que cierra el bucle de incentivos de la calidad.
- ¿Y el monolito durante la migración? Peligro de que se convierta en tierra de nadie ("eso es legacy, no es de nadie") mientras siga sirviendo a la mayoría de los clientes. Opciones: un equipo dueño del monolito con la misión explícita de encogerlo, o propiedad por zonas alineada con los futuros contextos. Lo inaceptable es que no tenga dueño.

### 5.4 La migración como cambio organizativo

La migración técnica y la organizativa son **el mismo proyecto**: reorganizar equipos hacia dominios, crear la plataforma, formar a la gente (operar sistemas distribuidos es una habilidad nueva: on-call, debugging distribuido, diseño de contratos), y gestionar la resistencia natural (el experto del monolito que pierde su feudo necesita un papel valioso en el mundo nuevo — es la persona que más sabe de los dominios). Patrocinio de dirección imprescindible: una migración de años no sobrevive como proyecto de guerrilla.

---

## 6. Síntesis y hoja de ruta

### 6.1 Todo junto: la hoja de ruta tipo

**Fase 0 — Decidir y preparar** *(Sesión 1)*
1. Objetivo, métricas con línea base, criterio de parada. ¿De verdad hay que dividir? (re-platforming / modularización como alternativas).
2. Bosquejo de bounded contexts (EventStorming con negocio) y clasificación core/supporting/generic *(Sesión 3)*.
3. Organización: equipos hacia dominios (Conway inverso), plataforma mínima, CI/CD también para el monolito *(Sesión 4)*.

**Fase 1 — Primeras extracciones** *(Sesiones 2–3)*
4. Elegir 1–2 candidatos del cuadrante valor×facilidad (módulos hoja, dolor concreto).
5. Fachada delante del monolito (Strangler); extraer con el patrón que toque (borde → Strangler; entrañas → Branch by Abstraction; remodelado → Bubble Context); **ACL en toda frontera**; Parallel Run donde el riesgo lo pida.
6. Observabilidad (logs centralizados, métricas, trazas) **desde el primer servicio**; pruebas de contrato desde el primer contrato.
7. **Borrar** el código extraído del monolito. Medir contra la línea base. Aprender.

**Fase 2 — Industrializar** 
8. Repetir con candidatos cada vez más centrales; plantillas de servicio ("el segundo servicio debe costar la mitad").
9. Atacar los datos en serio: propiedad de tablas, esquemas separados, romper cruces, sagas/outbox donde haya procesos *(Sesión 3)*.
10. Resiliencia como estándar: timeouts, retries con backoff, circuit breakers, idempotencia por defecto *(Sesión 4)*.

**Fase 3 — Converger y parar**
11. Decidir el destino del resto: lo genérico se reemplaza por compra/SaaS; lo estable y de bajo valor **se queda** en el monolito modularizado, deliberada y documentadamente.
12. Revisar el criterio de parada. Declarar el final. Celebrarlo.

### 6.2 Errores frecuentes (el top 10 de la formación)

1. Migrar sin objetivo medible ("modernizar") → sin criterio de prioridad ni de parada.
2. Dividir por capas técnicas o por entidades → monolito distribuido.
3. Dividir el código y **no** los datos → acoplamiento subterráneo permanente.
4. Big-bang de un sistema grande y vivo → objetivo móvil, cero valor entregado.
5. Cortar demasiado fino demasiado pronto → fusiones carísimas después.
6. Olvidar el paso de **borrar** el legacy → híbrido eterno, coste doble para siempre.
7. Cadenas síncronas largas sin timeouts/circuit breakers → fallos en cascada.
8. Asumir entrega exactamente-una-vez → duplicados sin idempotencia.
9. Observabilidad y pruebas de contrato "para después" → incidentes indepurables y roturas entre equipos.
10. Migrar la arquitectura sin migrar la organización → Conway gana siempre.

---

## Ejercicios

### Ejercicio 4.1 — Elegir el estilo de comunicación

Para cada interacción, decide: síncrono, evento (pub/sub) o comando por cola — y justifica:

- A. El checkout necesita el precio vigente del artículo antes de confirmar.
- B. Al completarse un pedido, hay que: actualizar fidelidad, enviar email de confirmación y actualizar el dashboard de analítica.
- C. Al registrarse un usuario hay que generarle un PDF de bienvenida (tarda ~10 s) que se le enviará por email.
- D. El servicio de envíos muestra en su pantalla el nombre y la dirección del cliente (datos del servicio de clientes) en cada paquete que procesa, miles de veces al día.

<details>
<summary>💡 Solución</summary>

- **A — Síncrono.** El checkout **no puede continuar** sin el dato y debe ser fresco (es dinero). Petición/respuesta con timeout corto y un fallback decidido (¿precio cacheado de hace segundos? ¿rechazar? — decisión de negocio). Es el caso legítimo de llamada síncrona: respuesta necesaria para continuar.

- **B — Evento (`PedidoCompletado`, pub/sub).** Tres reacciones independientes a un mismo hecho, ninguna necesaria para que el pedido se complete. Pedidos publica el evento sin conocer a los consumidores; fidelidad, notificaciones y analítica se suscriben. Mañana se añade un cuarto consumidor sin tocar pedidos — esa es la señal de que era un evento. Llamarlos síncronamente acoplaría la finalización del pedido a la disponibilidad de tres servicios secundarios.

- **C — Comando por cola** (`GenerarPDFBienvenida`). Es trabajo dirigido ("genera *este* PDF") que tarda y nadie espera en línea: el registro responde al usuario de inmediato y encola el trabajo. La cola da reintentos si el generador falla y absorbe picos de registros. (También sería modelable como reacción al evento `UsuarioRegistrado`; la diferencia conceptual: comando = quiero que ocurra, dirigido; evento = ocurrió, abierto. Cualquiera de los dos es defendible aquí si se justifica.)

- **D — Copia local alimentada por eventos.** Miles de lecturas al día de datos casi estáticos (nombre, dirección): llamar síncronamente al servicio de clientes en cada paquete acoplaría la operación de almacén a la disponibilidad de clientes y generaría tráfico inútil. Envíos mantiene una réplica local de los campos que necesita, actualizada por `ClienteActualizado`. Desfase de segundos: irrelevante para una etiqueta de envío. Es el patrón "datos de referencia por eventos" de la Sesión 3.

</details>

---

### Ejercicio 4.2 — Autopsia de un fallo en cascada

Viernes, 18:03. En **TiendaTotal**: el servicio de `recomendaciones` empieza a responder en 28 s (un despliegue suyo degradó una consulta). La página de producto llama síncronamente a `recomendaciones` sin timeout configurado (default: 60 s). A las 18:09 la página de producto no responde; a las 18:14 cae todo el sitio, incluido el checkout, que **no usa recomendaciones para nada**. Los reinicios de `recomendaciones` no curan: cada instancia recién levantada vuelve a saturarse en segundos.

**Preguntas:**
1. Reconstruye la cadena causal: ¿por qué cae la página de producto? ¿por qué cae el *checkout*?
2. ¿Por qué los reinicios no curan?
3. Prescribe los patrones de la sesión que habrían evitado o mitigado cada eslabón, y qué quedaría como comportamiento visible para el usuario con ellos puestos.

<details>
<summary>💡 Solución</summary>

**1. Cadena causal:**
- `recomendaciones` no está caído, está **lento** — el peor caso. Cada petición de la página de producto se queda **60 s** esperando (timeout por defecto).
- Los hilos/conexiones de la página de producto se quedan retenidos esperando; con el tráfico de un viernes por la tarde, el pool se agota en minutos → la página de producto deja de atender **cualquier** petición (18:09).
- El checkout cae (18:14) sin usar recomendaciones porque **comparte recursos** con la página de producto: mismo pool de conexiones/hilos del frontal, mismo balanceador saturado de conexiones colgadas, posiblemente mismas instancias. Sin mamparos, el agotamiento de recursos provocado por una dependencia se lleva por delante a funcionalidades que no dependen de ella. Es el fallo en cascada de libro: un servicio degradado → retención de recursos → caída de la plataforma.

**2. Los reinicios no curan** porque al levantarse cada instancia se encuentra la **avalancha completa**: todas las peticiones acumuladas + los usuarios recargando + (si hay reintentos automáticos) la tormenta de reintentos. Sin nada que limite la entrada, la instancia nueva se satura en segundos (*retry storm* / efecto estampida). Para recuperar haría falta cortar o limitar el tráfico entrante (abrir el circuito, rate limit, apagar la funcionalidad) y dejar respirar al servicio.

**3. Prescripción por eslabón:**

| Eslabón | Patrón que lo corta |
|---|---|
| Espera de 60 s por petición | **Timeout explícito** acorde a la latencia real (p. ej. 300 ms para unas recomendaciones) |
| Pool agotado por una dependencia | **Bulkhead:** pool propio y acotado para llamadas a recomendaciones; al agotarse, fallan *esas* llamadas, no la página |
| Machacar al servicio lento | **Circuit breaker:** tras el umbral de fallos/lentitud, abrir y fallar rápido sin llamar — y dar respiro a recomendaciones para recuperarse (también arregla el punto 2) |
| ¿Y el usuario? | **Fallback / degradación elegante:** página de producto **sin** el carrusel de recomendaciones (sección vacía o "los más vendidos" cacheados) |
| Checkout arrastrado | El bulkhead ya lo aísla; además, recursos/instancias separadas para el flujo crítico de compra |
| Detección a los 6 minutos por caída | **Observabilidad:** alerta por latencia p99 de recomendaciones y por estado del circuit breaker → se detecta a las 18:04, no cuando cae el sitio |

**Resultado con todo puesto:** el despliegue malo de recomendaciones habría producido… una página de producto sin recomendaciones durante un rato y una alerta al equipo correspondiente. El checkout ni se habría enterado. Esa es la diferencia entre "el fallo existe" (inevitable) y "el fallo se propaga" (evitable).

</details>

---

### Ejercicio 4.3 — ¿Romperá a los consumidores?

El equipo de `clientes` quiere hacer estos cambios en su API `GET /clientes/{id}`. Para cada uno: ¿compatible o rompedor? ¿Cómo lo harías de forma segura?

- A. Añadir el campo `segmento` a la respuesta.
- B. Renombrar `email` a `correoElectronico`.
- C. El campo `telefono` (hasta hoy siempre presente) pasará a ser `null` para clientes sin teléfono — antes se guardaba `"000000000"`.
- D. Dejar de emitir el evento `ClienteModificado` con el cliente completo y emitir solo `{id, camposCambiados}`.

**Pregunta extra:** ¿cómo sabría el equipo *antes de desplegar* cuáles de estos cambios rompen a quién?

<details>
<summary>💡 Solución</summary>

- **A — Compatible.** Añadir campos es seguro *si* los consumidores son lectores tolerantes (ignoran lo desconocido). Se despliega sin coordinación. (Si algún consumidor valida el esquema de forma estricta y rechaza campos extra, hasta esto rompe — razón por la que la tolerancia del lector debe ser norma del sistema.)

- **B — Rompedor** (equivale a eliminar `email`). Forma segura — **expand–contract**: (1) *expand:* responder con `email` **y** `correoElectronico` duplicados; (2) migrar consumidores a su ritmo, comprobando con pruebas de contrato/telemetría quién sigue usando el viejo; (3) *contract:* retirar `email` cuando nadie lo use. Si no se puede duplicar, versionado explícito (v2) con convivencia de versiones. Pregunta previa legítima: ¿aporta el renombrado lo suficiente para pagar este proceso?

- **C — Rompedor, y de los traicioneros:** no cambia el esquema sino la **semántica** (un campo nunca-nulo pasa a nullable). Los consumidores que hacen `telefono.substring(...)` explotarán con NullPointerException — y ningún validador de esquema lo avisa si el esquema ya decía "string opcional". Tratarlo como rompedor: anunciarlo, plazo de adaptación, y idealmente expresarlo en el contrato (marcar el campo como nullable y verificar con pruebas de contrato que los consumidores toleran null). El dato `"000000000"` es además un clásico aviso: los consumidores pueden tener lógica que *depende* del valor mágico.

- **D — Rompedor.** El esquema del **evento** es un contrato igual que la API. Los suscriptores que usaban el cliente completo dejarán de tener los datos. Expand–contract aplicado a eventos: emitir temporalmente ambos formatos (o el nuevo formato incluyendo aún el cliente completo como bloque "deprecated"), migrar suscriptores, contraer. Ojo además con los **eventos en vuelo y reprocesamientos**: los consumidores deben tolerar ambas versiones durante la transición (campo de versión en el evento).

**Extra:** **pruebas de contrato dirigidas por el consumidor.** Cada consumidor publica qué operaciones/campos usa (incluidos los esquemas de eventos); el CI del proveedor verifica cada cambio contra esas expectativas. Con eso, B y D fallarían el build del equipo de clientes *antes de desplegar*, con el nombre del consumidor afectado. C lo cazaría solo si los contratos expresan nulabilidad — buen ejemplo de que los contratos deben capturar semántica, no solo estructura. Complemento: telemetría de uso por campo/versión para saber cuándo se puede "contraer".

</details>

---

### Ejercicio 4.4 — Conway en acción

**BancoÁgil** planea su migración. Equipos actuales: *Frontend* (12), *Backend* (15), *DBA* (4), *QA* (6). Arquitectura objetivo: servicios de `cuentas`, `pagos`, `préstamos` y `onboarding` por dominio. El plan del CTO: "los equipos se quedan como están; Backend irá construyendo los servicios y DBA gestionará las bases de datos de todos; QA validará cada release conjunta."

**Tareas:**
1. Predice, usando la ley de Conway, qué pasará con ese plan aunque la arquitectura en papel sea por dominios.
2. Propón una estructura de equipos alternativa usando el vocabulario de Team Topologies (incluye qué pasa con DBAs y QA, y quién se ocupa del monolito durante la transición).

<details>
<summary>💡 Solución</summary>

**1. Predicción:** la estructura de comunicación (Frontend ↔ Backend ↔ DBA ↔ QA) se imprimirá sobre los servicios "por dominio":
- Cada cambio funcional necesitará un ticket a Frontend, otro a Backend, otro a DBA → los servicios tendrán fronteras de dominio en el nombre, pero **el flujo de cambio seguirá cruzando 3–4 equipos**: lead time largo, colas de coordinación — exactamente lo que la migración debía eliminar.
- DBA gestionando "las bases de datos de todos" recreará la **base de datos compartida de facto** (estándares únicos, esquemas acoplados, un cuello de botella por el que pasan todos los cambios de datos) — contra la BD-por-servicio de la Sesión 3.
- "QA valida cada release conjunta" institucionaliza el **release train**: nadie despliega solo → no hay despliegue independiente → monolito distribuido (Sesión 1). Conway no es opcional: gana siempre; la única decisión es si juega a favor o en contra.

**2. Estructura alternativa (Conway inverso + Team Topologies):**

- **4 stream-aligned teams** por dominio: *Cuentas*, *Pagos*, *Préstamos*, *Onboarding* — cada uno **multifuncional**: frontend + backend + capacidades de datos y pruebas *dentro* del equipo, dueño de su(s) servicio(s) de extremo a extremo ("you build it, you run it"), desplegando de forma independiente (pruebas de contrato en lugar de QA de release conjunta).
- **1 platform team:** CI/CD, observabilidad, plantillas de servicio, infraestructura como autoservicio. Destino natural de parte de los DBAs: convertir su saber en plataforma de datos autoservida (aprovisionamiento de BDs, backups, estándares como herramientas, no como ventanilla).
- **Enabling teams temporales:** los QA seniors → enabling de calidad/automatización que enseña a los stream-aligned a poseer sus pruebas (y se retira); igual para DDD/contratos si hace falta. El resto de QA y DBAs se integra en los equipos de dominio.
- **El monolito:** propiedad explícita — o un equipo dueño con la misión de encogerlo, o (mejor a medio plazo) propiedad por zonas: cada equipo de dominio es dueño de "su" parte del monolito además de sus servicios, con el incentivo de extraerla. Nunca tierra de nadie.
- Tamaños orientativos con las ~37 personas: 4 equipos de dominio de 6–8 + plataforma de 5–6 + enabling pequeño y temporal. (Las cifras exactas importan menos que el principio: equipos = dominios, plataforma como producto, nada de equipos-capa en el flujo de cambio.)

</details>

---

### Ejercicio 4.5 — Ejercicio de síntesis: la hoja de ruta de SaludPlus

*(Ejercicio final integrador — usa material de las 4 sesiones.)*

**SaludPlus** gestiona clínicas: monolito de 14 años (citas, historiales clínicos, facturación a aseguradoras, recetas, informes). 40 desarrolladores en equipos por capa. Dolores: (1) la facturación a aseguradoras cambia cada mes por regulación y cada cambio exige una release completa mensual; (2) el módulo de citas se satura los lunes por la mañana; (3) los historiales clínicos están bajo una regulación estricta y no han cambiado funcionalmente en 3 años; (4) quieren lanzar telemedicina, que no existe en el sistema actual.

**Tarea:** Esboza una hoja de ruta de migración (fases, no fechas) que cubra: objetivo y métricas; qué se extrae, en qué orden y con qué patrón; tratamiento de datos; organización; y qué **no** se migra. Justifica cada decisión con los conceptos del curso.

<details>
<summary>💡 Solución (ejemplo razonado — admite variantes)</summary>

**Objetivo y métricas (Sesión 1):** "Que los cambios regulatorios de facturación lleguen a producción en días, no meses; que las citas aguanten los picos; y lanzar telemedicina sin heredar el legacy." Métricas: lead time de cambios de facturación (hoy ~1 mes → < 1 semana), latencia/errores de citas el lunes 8–10 h, time-to-market de telemedicina. Línea base medida antes de empezar.

**Organización primero (Sesión 4):** Conway inverso — pasar de equipos por capa a equipos de dominio: *Facturación*, *Citas*, *Telemedicina*, *Núcleo clínico* (dueño del monolito con la misión de encogerlo), + *Plataforma* (CI/CD, observabilidad, plantillas). CI/CD también para el monolito (se redesplegará mucho).

**Fase 1 — Facturación (el dolor nº 1, motor del proyecto):**
- Es **alta volatilidad** (cambios mensuales) → donde la independencia de despliegue rinde más (Sesión 1: secuenciación por volatilidad/valor).
- Borde probablemente interceptable (procesos de facturación, APIs/lotes hacia aseguradoras) → **Strangler Fig**; **ACL** hacia los datos clínicos y de citas que necesite (el modelo regulatorio nuevo no debe contaminarse del legacy).
- Por ser dinero y regulación → **Parallel Run** en sombra durante 2–3 ciclos de facturación comparando importes con el legacy antes de conmutar (Sesión 2).
- Datos: propiedad de las tablas de facturación, esquema separado, copia local por eventos de los datos maestros (pacientes, citas) que necesite; **outbox** para sus eventos (Sesión 3). Borrar el módulo del monolito al terminar.

**Fase 2 — Telemedicina (dominio nuevo):**
- No existe en el legacy → nace fuera como **Bubble Context**: modelo limpio (videoconsulta, disponibilidad, triaje) con **ACL** de solo-lo-necesario hacia pacientes/citas. Es además, probablemente, dominio **core** (diferenciación) → mejor equipo, modelado fino (Sesión 3).
- Comunicación: eventos para integrarse (cita de telemedicina creada → el núcleo la ve), síncrono solo donde haga falta respuesta inmediata (Sesión 4).

**Fase 3 — Citas (el problema de escala):**
- El pico del lunes pide **escalado selectivo**. Primer paso barato: ¿re-platforming del monolito (autoescalado) resuelve? Si no basta o se quiere autonomía: extraer `citas` con **Strangler** (las pantallas/APIs de citas son un borde claro), con su BD y eventos `CitaCreada/Cancelada` que ya consumen facturación y telemedicina.
- Resiliencia desde el día uno: timeouts, circuit breakers y degradación elegante (si recomendación de huecos falla, la agenda básica sigue) (Sesión 4).

**Qué NO se migra (decisión deliberada, Sesiones 1 y 3):**
- **Historiales clínicos:** volatilidad casi nula (3 años sin cambios) + regulación estricta = bajo valor y alto riesgo de extracción. Se quedan en el monolito (modularizado), detrás de una API interna limpia para que los servicios los consuman vía ACL. Revisable si su volatilidad cambia.
- **Informes:** no se "migran" — se alimenta una vista/plataforma analítica desde los eventos de los nuevos servicios (Sesión 3, consultas entre servicios).

**Transversales:** observabilidad (logs centralizados, trazas con propagación también por eventos) desde la primera extracción; pruebas de contrato desde el primer contrato; tests de caracterización antes de cada extracción; criterio de parada explícito: "facturación, telemedicina y citas fuera; el resto se queda en un monolito modular con dueño" — y el paso final de cada fase es **borrar** lo extraído.

**Por qué este orden:** facturación primero porque combina máximo valor (dolor mensual) con riesgo gestionable vía parallel run; telemedicina en paralelo porque no toca el legacy casi nada (greenfield protegido); citas después porque quizá lo resuelva el re-platforming (intervención mínima primero); historiales nunca, porque extraer lo estable-y-regulado es pagar máximo riesgo por valor nulo.

</details>

---

## Resumen de la sesión y cierre de la formación

- **Comunicación:** síncrono solo cuando la respuesta es imprescindible; eventos para anunciar hechos, colas para trabajo diferido; gateway como fachada (y aliado del Strangler); contratos versionados con expand–contract y pruebas de contrato.
- **Resiliencia:** el fallo parcial es lo normal; timeouts deliberados, reintentos idempotentes con backoff+jitter, circuit breakers con fallback, bulkheads — todo para que el fallo no se propague. *Lento es peor que caído.*
- **Observabilidad:** logs estructurados centralizados, métricas con alertas sobre síntomas, trazado distribuido con propagación de contexto — construida durante la migración, no después.
- **Pruebas y entrega:** caracterización antes de extraer, contratos en CI, pipeline por servicio, separar despliegue de liberación.
- **Organización:** Conway gana siempre — equipos por dominio (stream-aligned), plataforma como producto, propiedad única de cada servicio, y el monolito nunca sin dueño.
- **Hoja de ruta:** decidir con métricas → extraer por valor×facilidad con el patrón adecuado y ACLs → industrializar (datos, resiliencia, plataforma) → converger, dejar deliberadamente lo que no compense migrar, y **parar**.

*Fin de la formación — ¡gracias y buenas migraciones!* 🌱
