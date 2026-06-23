---
marp: true
theme: default
paginate: true
header: '![h:34px](nobleprog-logo.png) Tecnología de Punta Formación SL'
footer: 'Sesión 3 — Domain-Driven Design y descomposición de datos'
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

# Sesión 3
## Domain-Driven Design y descomposición de datos

**Migración de Monolitos a Microservicios**
Carlos Georg Lübbe · 3 h

<!--
ES: Bienvenida a la tercera sesión. La Sesión 2 dio el CÓMO mover funcionalidad con
seguridad; hoy respondemos las dos preguntas que esos patrones dejaban abiertas:
DÓNDE cortar (DDD) y qué hacer con los DATOS, que es la parte más difícil y más
aplazada de toda migración. Aviso de tono: la primera mitad es modelado (negocio);
la segunda, datos y consistencia (lo técnicamente más duro del curso).

EN: Welcome to session three. Session 2 gave the HOW of moving functionality safely;
today we answer the two questions those patterns left open: WHERE to cut (DDD) and what
to do with the DATA — the hardest and most-deferred part of any migration. Heads-up on
tone: the first half is modeling (business); the second, data and consistency (the
technically hardest part of the course).
-->

---

## Objetivo de hoy

Encontrar **buenos límites** (DDD) y abordar lo más difícil: **los datos**.

1. **DDD:** lenguaje ubicuo y *bounded contexts* — ¿por dónde corto?
2. **Encontrar los límites:** context maps, core/supporting/generic, granularidad
3. **Descomposición de datos:** BD por servicio, partir la BD compartida
4. **Consistencia sin transacciones distribuidas:** eventual, sagas, outbox

<!--
ES: Recorrido de la sesión. Habrá ejercicios tras cada bloque. Idea transversal: el
límite correcto sigue al NEGOCIO, no a la tecnología; y partir los datos es lo que
convierte "código extraído" en "migración hecha de verdad".

EN: Roadmap. Exercises after each block. Cross-cutting idea: the right boundary follows
the BUSINESS, not technology; and splitting the data is what turns "extracted code" into
"a migration actually done".
-->

---

<!-- _class: lead -->

# 1. Domain-Driven Design

---

## ¿Por qué DDD y microservicios?

La pregunta más cara de la migración: **¿por dónde corto?** Un corte malo = monolito distribuido (Sesión 1).

**DDD** (Eric Evans, 2003) alinea los límites del software con los del **negocio** — los límites más estables que existen.

- **DDD estratégico** — cómo trocear un dominio: *bounded contexts*, lenguaje ubicuo, context maps. ← **esto necesita la migración**
- **DDD táctico** — diseño *dentro* de un contexto: entidades, agregados… (solo tocaremos *agregados*)

<!--
ES: El mensaje clave: los límites del negocio son los más estables, por eso cortar por
ahí da servicios que no hay que rehacer cada trimestre. DDD tiene dos planos; para
decidir LÍMITES DE SERVICIO solo necesitamos el estratégico. El táctico (agregados,
repositorios) es diseño interno — lo mencionamos solo donde toca la consistencia.

EN: Key message: business boundaries are the most stable, so cutting along them yields
services you don't have to redo every quarter. DDD has two planes; for deciding SERVICE
BOUNDARIES we only need the strategic one. The tactical one (aggregates, repositories) is
internal design — we mention it only where it touches consistency.
-->

---

## Lenguaje ubicuo

Vocabulario **compartido** entre negocio y desarrollo, usado igual en conversaciones, documentos **y código**. Si negocio dice "póliza" y el código dice `ContractRecord`, cada conversación necesita traducción.

**La observación que encuentra límites:** el lenguaje **cambia de significado según el contexto**.

> "Cliente" no es lo mismo para **ventas** (un lead), **facturación** (un titular fiscal) y **soporte** (alguien con tickets). No es mala comunicación: son **conceptos distintos** que comparten nombre.

<!--
ES: El lenguaje ubicuo no es "documentación bonita": es la herramienta de detección de
fronteras. Cuando la MISMA palabra significa cosas distintas en dos sitios, casi seguro
hay dos contextos. Cuando dos palabras distintas significan lo mismo (sinónimos entre
áreas), también. Insistir: que "cliente" signifique tres cosas NO es un error a unificar;
es información sobre la estructura del dominio.

EN: Ubiquitous language isn't "nice documentation": it's the boundary-detection tool. When
the SAME word means different things in two places, there are almost certainly two
contexts. When two different words mean the same thing (synonyms across areas), likewise.
Stress: "customer" meaning three things is NOT an error to unify; it's information about
the domain's structure.
-->

---

## Bounded contexts

Frontera dentro de la cual un **modelo y su lenguaje** son válidos y consistentes. No hace falta **un modelo único de empresa** (el "Cliente canónico" de 200 campos que no sirve a nadie):

```
┌── Ventas ─────────────┐  ┌── Facturación ────────┐  ┌── Soporte ───────────┐
│ Cliente:              │  │ Cliente:              │  │ Cliente:             │
│  · interés            │  │  · NIF, dirección     │  │  · tickets           │
│  · historial contacto │  │    fiscal             │  │  · nivel SLA         │
│  · probabilidad cierre│  │  · forma de pago      │  │  · idioma preferido  │
└───────────────────────┘  └───────────────────────┘  └──────────────────────┘
        mismo nombre, tres modelos distintos — y está bien
```

<!--
ES: La consecuencia liberadora: dejas de pelear por "el" modelo de Cliente. Cada contexto
modela SU versión, solo con los atributos que le importan, y se comunican por contratos.
Dentro de Facturación, Cliente significa exactamente una cosa; fuera, no se garantiza
nada. Esto es lo contrario del modelo canónico corporativo (la entidad-dios con 200
campos), que es justo lo que en una migración se convierte en la tabla compartida
imposible de separar.

EN: The liberating consequence: you stop fighting over "the" Customer model. Each context
models ITS version, with only the attributes it cares about, and they talk via contracts.
Inside Billing, Customer means exactly one thing; outside, nothing is guaranteed. This is
the opposite of the corporate canonical model (the 200-field god-entity), which is exactly
what becomes the un-splittable shared table in a migration.
-->

---

## Cómo detectar fronteras en la práctica

- **Cambios de significado** de una palabra según el departamento
- **Sinónimos:** dos áreas llaman distinto a lo mismo
- Fronteras **organizativas** y de proceso de negocio
- En el monolito: grupos de tablas/módulos que **cambian juntos**; "traducciones" implícitas (mappers, columnas reinterpretadas)
- Talleres tipo **EventStorming:** negocio y desarrollo mapean los eventos del dominio; los racimos y cambios de vocabulario sugieren contextos

<!--
ES: Pistas concretas para el trabajo de campo. Las dos primeras salen del lenguaje
ubicuo. La cuarta es oro en una migración: el propio monolito ya tiene las costuras
dibujadas si miras qué tablas cambian juntas y dónde el código "traduce" de un modelo a
otro. EventStorming es el taller colaborativo más usado para esto — mencionarlo como
técnica, no entrar en detalle.

EN: Concrete clues for fieldwork. The first two come from ubiquitous language. The fourth
is gold in a migration: the monolith already has the seams drawn if you look at which
tables change together and where the code "translates" between models. EventStorming is
the most-used collaborative workshop for this — mention it as a technique, don't go deep.
-->

---

## Un servicio ≈ un bounded context

> Un microservicio **no debe ser más pequeño que un bounded context**; de partida, **un contexto ≈ un servicio**.

**Por qué funciona:** dentro de un contexto la cohesión es alta (cambia junto → vive junto); entre contextos el acoplamiento es bajo (pocas interacciones, formalizables como contratos). Justo lo que pide el despliegue independiente.

**Por qué fracasa cortar por capa o entidad:** "añadir descuentos por volumen" atraviesa capas y entidades → toca N servicios → monolito distribuido.

<!--
ES: La regla de oro de la descomposición. Ejemplo: e-commerce por dominio (catalogo,
pedidos, pagos, envios, clientes) → un cambio toca UN servicio. El mismo sistema por
capas (web-api / business-logic / data-access) obliga a tocar los tres para CUALQUIER
cambio. "No más pequeño que un contexto" es el límite inferior: por debajo aparece el
monolito distribuido. Y de partida: pocos servicios, uno por contexto; subdividir luego.

EN: The golden rule of decomposition. Example: e-commerce by domain (catalog, orders,
payments, shipping, customers) → one change touches ONE service. The same system by layers
(web-api / business-logic / data-access) forces touching all three for ANY change. "No
smaller than a context" is the lower bound: below it the distributed monolith appears. And
to start: few services, one per context; subdivide later.
-->

---

<!-- _class: lead -->

# Ejercicio 3.1
## Detectar bounded contexts por el lenguaje

*(ver cuaderno de ejercicios)*

<!--
ES: Pausa para ejercicio. 3.1: VuelaYa (aerolínea) — la palabra "vuelo" significa cosas
distintas en Operaciones, Ventas, Fidelidad y Mantenimiento. Tareas: desambiguar los
significados, proponer los bounded contexts, y razonar qué pasaría con una entidad Vuelo
única (la entidad-dios → tabla compartida que bloquea la migración). El doble "pasajero"
(comprador vs. embarcado) refuerza la frontera. ~15-20 min con puesta en común.

EN: Exercise break. 3.1: VuelaYa (airline) — the word "flight" means different things in
Operations, Sales, Loyalty and Maintenance. Tasks: disambiguate the meanings, propose the
bounded contexts, and reason what a single Flight entity would cause (the god-entity →
shared table that blocks the migration). The double "passenger" (buyer vs. boarded)
reinforces the boundary. ~15-20 min with group discussion.
-->

---

<!-- _class: lead -->

# 2. Encontrar los límites

---

## Context maps

El **plano de los contextos y sus relaciones**. Saber cuáles son no basta: hay que saber cómo se influyen (eso fija contratos y poder de negociación).

| Relación | Significado |
|---|---|
| **Customer–Supplier** | El proveedor tiene en cuenta al consumidor (hay negociación) |
| **Conformist** | El consumidor acepta el modelo del proveedor tal cual |
| **Anticorruption Layer** | El consumidor se protege traduciendo (Sesión 2) |
| **Shared Kernel** | Comparten parte del modelo (acoplamiento fuerte — con moderación) |
| **Open Host / Published Language** | API/formato estable y documentado para muchos |
| **Separate Ways** | No se integran: duplicar es más barato que coordinar |

<!--
ES: El context map es el segundo entregable de DDD estratégico, después de la lista de
contextos. En una migración sirve para: ver qué fronteras ya existen de facto, decidir
dónde hacen falta ACLs (relación Anticorruption Layer), y detectar relaciones enfermas —
el caso clásico: todo el mundo "conformista" con el modelo de la BD central. ACL la vimos
en la Sesión 2: aquí se ve que es UNA de las relaciones posibles del catálogo de Evans.

EN: The context map is the second strategic-DDD deliverable, after the list of contexts.
In a migration it helps: see which boundaries already exist de facto, decide where ACLs
are needed (the Anticorruption Layer relationship), and spot unhealthy relationships — the
classic one: everyone "conformist" to the central DB model. We saw ACL in Session 2: here
you see it's ONE of the possible relationships in Evans' catalog.
-->

---

## Capacidad de negocio vs. subdominio

Dos heurísticas para proponer contextos:

- **Por capacidad de negocio:** ¿qué *hace* la empresa? (gestionar pedidos, facturar). Estable, de fuera adentro. Riesgo: copiar el organigrama actual.
- **Por subdominio:** ¿de qué *se compone* el dominio?
  - **Core** — lo que te diferencia. Mejor equipo, mejor diseño.
  - **Supporting** — necesario, no diferencial; a medida porque no hay alternativa.
  - **Generic** — lo resuelve una solución comprada (contabilidad, login, email).

> **core / supporting / generic** = cómo **repartir el esfuerzo**: modelado fino al core; lo genérico a menudo **ni se migra — se reemplaza** por SaaS.

<!--
ES: Las dos heurísticas son complementarias: una mira lo que la empresa hace, la otra de
qué se compone el problema. La clasificación core/supporting/generic es la herramienta de
priorización más útil de la sesión: evita gastar el mismo esfuerzo de diseño (y de
migración) en todo por igual. Lo genérico (email, contabilidad) no se reescribe como
microservicio propio: se compra y se borra el módulo legacy — la forma más barata de
adelgazar el monolito. Es la base del Ejercicio 3.2.

EN: The two heuristics are complementary: one looks at what the company does, the other at
what the problem is made of. The core/supporting/generic classification is the most useful
prioritization tool of the session: it avoids spending the same design (and migration)
effort on everything equally. Generic stuff (email, accounting) isn't rewritten as your
own microservice: you buy it and delete the legacy module — the cheapest way to slim the
monolith. It's the basis of Exercise 3.2.
-->

---

## Granularidad: ¿demasiado pequeño?

Señales de haber cortado **demasiado fino**:

- **Cambios en cascada:** una función media toca varios servicios
- **Charla excesiva:** dos servicios siempre juntos en las trazas
- **Transacciones partidas:** necesitas consistencia fuerte *entre* servicios
- **Datos siameses:** dos servicios leen/escriben las mismas tablas
- Más servicios que desarrolladores

> **Ante la duda, corta más grueso.** Dividir un servicio grande es barato; **fusionar** dos pequeños (unir modelos, datos y contratos publicados) es caro.

<!--
ES: La granularidad es asimétrica y por eso la heurística es clara: empezar grueso. Las
cinco señales son síntomas del monolito distribuido aplicados a "te pasaste de fino".
Subdividir un servicio que resultó grande es un refactor local; fusionar dos que
resultaron pequeños obliga a deshacer contratos ya publicados, unir datos y unir modelos
— mucho más caro. Mensaje: pocos servicios alineados con contextos, y subdividir solo
cuando el dolor lo justifique.

EN: Granularity is asymmetric, so the heuristic is clear: start coarse. The five signals
are distributed-monolith symptoms applied to "you cut too fine". Subdividing a service that
turned out big is a local refactor; merging two that turned out small forces undoing
published contracts, joining data and joining models — far costlier. Message: few services
aligned to contexts, and subdivide only when the pain justifies it.
-->

---

<!-- _class: lead -->

# Ejercicio 3.2
## Core, supporting o generic

*(ver cuaderno de ejercicios)*

<!--
ES: Pausa para ejercicio. 3.2: clasificar subdominios de VuelaYa (precios dinámicos,
tripulaciones, emails, contabilidad, fidelidad) en core/supporting/generic y extraer
consecuencias para la migración. Clave: precios = core (máxima inversión, bubble context);
emails y contabilidad = generic (no se migran, se reemplazan por SaaS/ERP + ACL). El
razonamiento ("¿esto nos diferencia?") importa más que la etiqueta. ~15 min.

EN: Exercise break. 3.2: classify VuelaYa subdomains (dynamic pricing, crew, emails,
accounting, loyalty) into core/supporting/generic and draw migration consequences. Key:
pricing = core (max investment, bubble context); emails and accounting = generic (not
migrated, replaced by SaaS/ERP + ACL). The reasoning ("does this differentiate us?")
matters more than the label. ~15 min.
-->

---

<!-- _class: lead -->

# 3. Descomposición de datos

---

## El problema: la BD compartida

Extrae todo el código que quieras: mientras **compartan base de datos, siguen acoplados**.

- El **esquema es un contrato implícito**: nadie cambia una tabla sin romper a los demás → adiós despliegue independiente
- La lógica acaba en el sitio equivocado (se reimplementa y diverge en cada consumidor)
- No puedes cambiar de tecnología de almacenamiento por servicio
- El rendimiento de uno afecta a todos (bloqueos, pools, picos)

<!--
ES: Este es el corazón de la sesión y el cabo suelto de casi todos los patrones de la
Sesión 2 (el Strangler "no estrangula los datos"). La BD compartida es el acoplamiento que
sobrevive por debajo aunque hayas separado todo el código. El esquema compartido es un
contrato que nadie firmó pero todos dependen. Hasta que no se parte la BD, la migración
NO está hecha — solo lo parece.

EN: This is the heart of the session and the loose end of almost every Session 2 pattern
(the Strangler "doesn't strangle the data"). The shared DB is the coupling that survives
underneath even after you've split all the code. The shared schema is a contract nobody
signed but everyone depends on. Until the DB is split, the migration is NOT done — it just
looks done.
-->

---

## Objetivo: base de datos por servicio

> Cada servicio es el **único** que accede a sus datos; los demás pasan por su **API**.

- Restaura el **despliegue independiente** (cada uno cambia su esquema)
- La lógica de negocio vive en **un** sitio (su dueño)
- Cada servicio elige su **tecnología** de almacenamiento
- Aísla el rendimiento

Es la parte **más difícil y más aplazada** de toda migración.

<!--
ES: "BD por servicio" no es purismo: es la condición sin la cual no hay despliegue
independiente, que era el motivo entero de migrar (Sesión 1). El precio es alto (lo vemos
en la slide del coste), por eso se aplaza — pero aplazarlo indefinidamente es quedarse en
monolito distribuido. La regla operativa: nadie toca la BD de otro; todo acceso cruzado
pasa por API o por eventos.

EN: "DB per service" isn't purism: it's the condition without which there's no independent
deployment, which was the entire reason to migrate (Session 1). The price is high (see the
cost slide), which is why it's deferred — but deferring it indefinitely means staying a
distributed monolith. Operating rule: nobody touches another's DB; every cross access goes
through an API or events.
-->

---

## Dividir los datos: propiedad primero

1. **Mapear propiedad:** por cada tabla, ¿qué contexto la *escribe*? (las que escribe todo el mundo = alarma)
2. **Separar el esquema lógicamente:** esquemas/usuarios distintos en la misma instancia, sin cruces. Barato, revela todos los acoplamientos
3. **Romper los accesos cruzados** uno a uno → API del dueño · copia local por eventos · mover el dato de dueño
4. **Separar físicamente** (instancia propia) cuando ya no hay cruces

<!--
ES: El orden importa: PROPIEDAD primero. Quién escribe una tabla es su dueño natural; los
demás son lectores que hay que reconducir. El paso 2 (separar lógico antes que físico) es
el truco barato: con esquemas distintos en la misma instancia, cualquier JOIN o escritura
cruzada falla y aflora — detectas todo el acoplamiento sin mover datos. Solo al final se
separa físicamente. Romper accesos: API si necesitas el dato fresco; copia por eventos si
toleras desfase; o reasignar dueño si el reparto inicial estaba mal.

EN: Order matters: OWNERSHIP first. Whoever writes a table is its natural owner; the rest
are readers to redirect. Step 2 (logical before physical split) is the cheap trick: with
separate schemas in the same instance, any cross JOIN or write fails and surfaces — you
find all the coupling without moving data. Only at the end do you split physically.
Breaking accesses: API if you need fresh data; event-fed copy if you tolerate lag; or
reassign owner if the initial split was wrong.
-->

---

## Casos típicos difíciles

- **Tabla de referencia compartida** (`paises`): duplicar en cada servicio (cambia poco) o un servicio de referencia
- **Una tabla, dos dueños** (`pedidos` con columnas de pedido + de envío): **partirla**; la clave (`pedido_id`) pasa a ser referencia entre servicios
- **Joins entre dominios** (informe clientes × pedidos): ya no hay JOIN entre BDs → composición en el llamante, o **vista materializada** por eventos (§4.5)

<!--
ES: Los tres casos que siempre aparecen. El de referencia (paises, monedas) casi siempre
se resuelve duplicando — montar un servicio para algo que cambia una vez al año es
sobreingeniería. "Una tabla, dos dueños" delata que el reparto de columnas escondía dos
conceptos: se parte por columnas y la clave queda como referencia. Los joins entre dominios
son el dolor recurrente y enlazan con la última slide de la sesión (consultas por eventos,
no por BD compartida).

EN: The three cases that always appear. Reference data (countries, currencies) is almost
always solved by duplicating — building a service for something that changes once a year is
over-engineering. "One table, two owners" reveals the column split hid two concepts: split
by columns, key stays as a reference. Cross-domain joins are the recurring pain and link to
the session's last slide (queries via events, not a shared DB).
-->

---

## El coste de los datos distribuidos

Sin anestesia: al partir los datos **se pierde** lo que la BD única regalaba.

| Se pierde | Lo sustituye (peor, más trabajo) |
|---|---|
| Transacciones ACID entre dominios | Sagas + consistencia eventual |
| JOIN entre cualquier par de tablas | Composición por API / vistas materializadas |
| Integridad referencial (FK) global | Validación en app; tolerar referencias rotas |
| Un único lugar donde mirar | Datos repartidos; catálogo/linaje |

> Por eso los **límites importan tanto**: buen corte → transacciones y joins entre servicios raros. Los invariantes con consistencia fuerte van **dentro de un servicio** (un *agregado*).

<!--
ES: Hay que ser honesto: BD por servicio no es gratis, se pierden cosas valiosas y lo que
las sustituye es más trabajo y peor. Esta tabla es el argumento de por qué el CORTE importa
tanto (cierra el círculo con DDD): si los límites son buenos, las transacciones y joins
ENTRE servicios son raros y el coste es asumible; si son malos, son constantes y el sistema
es inmanejable. La regla de oro de consistencia: un invariante que necesita ACID debe caber
dentro de UN servicio — en DDD táctico, dentro de un agregado (la unidad de consistencia).

EN: Be honest: DB per service isn't free, you lose valuable things and what replaces them
is more work and worse. This table is the argument for why the CUT matters so much (closes
the loop with DDD): with good boundaries, transactions and joins BETWEEN services are rare
and the cost is bearable; with bad ones, they're constant and the system is unmanageable.
The golden consistency rule: an invariant needing ACID must fit inside ONE service — in
tactical DDD, inside an aggregate (the unit of consistency).
-->

---

<!-- _class: lead -->

# Ejercicio 3.3
## Partir la base de datos compartida

*(ver cuaderno de ejercicios)*

<!--
ES: Pausa para ejercicio. 3.3: ComidaFlash — pedidos y restaurantes comparten BD. Asignar
propiedad por quién escribe; eliminar cada acceso cruzado (API vs. copia por eventos según
frescura/frecuencia); decidir qué hacer con codigos_postales (datos de referencia →
duplicar); y el punto clave: al separar, el menu_item puede agotarse entre el clic y la
confirmación → aparece un problema de consistencia que se resuelve con semántica de negocio
(el restaurante confirma/rechaza = compensación de una saga). ~25 min.

EN: Exercise break. 3.3: ComidaFlash — orders and restaurants share a DB. Assign ownership
by who writes; remove each cross access (API vs. event copy by freshness/frequency); decide
on postal_codes (reference data → duplicate); and the key point: after splitting, the
menu_item can sell out between click and confirmation → a consistency problem solved with
business semantics (the restaurant confirms/rejects = a saga compensation). ~25 min.
-->

---

<!-- _class: lead -->

# 4. Consistencia sin
# transacciones distribuidas

---

## Por qué no usamos 2PC

La transacción distribuida clásica (*two-phase commit*): un coordinador pide votar a todos y luego confirma. En microservicios **se evita**:

- **Bloquea recursos** en todos los participantes mientras dura → mata el rendimiento
- El **coordinador** es un punto único de fallo, con estados "en duda" horribles
- **Acopla la disponibilidad** de todos (avanza al ritmo del más lento/caído)
- Muchas tecnologías (colas, NoSQL, HTTP) **no lo soportan**

<!--
ES: 2PC es la respuesta "obvia" a perder las transacciones ACID, y es una trampa en
microservicios. El problema de fondo: reintroduce el acoplamiento fuerte (de disponibilidad
y de rendimiento) que migrar pretendía eliminar. Si A, B y C deben commitear juntos, han
vuelto a estar acoplados aunque sean tres servicios. Por eso la industria eligió el otro
camino: consistencia eventual + sagas. No es que 2PC sea imposible; es que reconstruye el
monolito a nivel de transacción.

EN: 2PC is the "obvious" answer to losing ACID transactions, and it's a trap in
microservices. The underlying problem: it reintroduces the strong coupling (availability
and performance) that migrating was meant to remove. If A, B and C must commit together,
they're coupled again even as three services. That's why the industry chose the other path:
eventual consistency + sagas. It's not that 2PC is impossible; it's that it rebuilds the
monolith at the transaction level.
-->

---

## Consistencia eventual

Aceptar estados intermedios visibles y garantizar que el sistema **converge**: *"el pedido está creado pero el stock aún no descontado… durante 200 ms."*

**No es (solo) un problema técnico, es una conversación de negocio.** Casi todos los procesos reales *ya son* eventualmente consistentes (la tienda física vende y el stock central se entera por la noche).

> Preguntas correctas — **las responde negocio:** ¿qué desfase es aceptable aquí? ¿qué hacemos si al final no se puede completar?

<!--
ES: El salto mental más importante de la sesión: la consistencia fuerte instantánea era una
comodidad de la BD única, no una ley del universo. El mundo real funciona por consistencia
eventual desde siempre (bancos liquidando por lotes, almacenes reconciliando de noche). Por
eso las preguntas clave son de NEGOCIO, no técnicas: cuánto desfase tolera ESTE proceso, y
qué se hace si no se completa. El rol del arquitecto es hacer esas preguntas, no inventarse
las respuestas.

EN: The most important mental shift of the session: instantaneous strong consistency was a
convenience of the single DB, not a law of the universe. The real world has run on eventual
consistency forever (banks settling in batches, warehouses reconciling overnight). So the
key questions are BUSINESS ones, not technical: how much lag THIS process tolerates, and
what to do if it can't complete. The architect's job is to ask those questions, not invent
the answers.
-->

---

## El patrón Saga

Una **saga** = secuencia de transacciones **locales** (cada una ACID en su servicio) que juntas implementan un proceso. Si un paso falla, no hay rollback global: se ejecutan **compensaciones** (acciones de negocio que deshacen).

**"Realizar pedido":** 1) Pedidos crea → 2) Stock reserva → 3) Pagos cobra → 4) Pedidos confirma.
Si el cobro (3) falla → compensar 2 (liberar reserva) y 1 (marcar pedido **cancelado**, no borrado).

> La compensación es una **acción de negocio** con su semántica, no un "deshacer" mágico.

<!--
ES: La saga es la respuesta a "¿cómo coordino algo que antes era una transacción que cruzaba
tablas?". Clave: NO hay rollback automático; tú escribes las compensaciones, y son acciones
de negocio. "Cancelado, no borrado" es el ejemplo perfecto: queda el rastro y el motivo; la
compensación tiene significado de negocio (un pedido cancelado no es un pedido que nunca
existió). Esto enlaza con el Ejercicio 3.4 (diseñar una saga) donde el ORDEN de los pasos
es una decisión de dinero.

EN: The saga answers "how do I coordinate what used to be a transaction crossing tables?".
Key: there's NO automatic rollback; you write the compensations, and they're business
actions. "Cancelled, not deleted" is the perfect example: the trace and reason remain; the
compensation has business meaning (a cancelled order isn't an order that never existed).
This links to Exercise 3.4 (design a saga) where step ORDER is a money decision.
-->

---

## Orquestación vs. coreografía

**Orquestación** — un coordinador central llama a cada paso y decide compensar.
✅ proceso explícito en un sitio; fácil de razonar, depurar, modificar
⚠️ concentra acoplamiento (lo conoce todo); puede volverse un "dios"

**Coreografía** — sin coordinador; cada servicio reacciona a eventos y emite los suyos.
✅ acoplamiento mínimo, servicios autónomos, encaja con eventos
⚠️ el proceso global **no está escrito** en ningún sitio: emerge. "¿Dónde está el pedido 4711?"

> Procesos cortos y estables → coreografía. Largos, con ramas/compensaciones o que exigen visibilidad → orquestación. Mezclar vale.

<!--
ES: Los dos estilos de coordinar una saga. El eje de decisión es VISIBILIDAD vs.
ACOPLAMIENTO. Orquestación: el proceso vive en un sitio (sabes dónde está cada pedido y por
qué se atascó), a cambio de un orquestador que conoce a todos. Coreografía: máxima autonomía
y desacoplamiento, a cambio de que el flujo global no esté escrito en ninguna parte — bueno
para cascadas simples, peligroso para procesos largos. Heurística práctica en la slide; y
mezclar es legítimo (orquestar el núcleo, coreografiar las reacciones periféricas).

EN: The two styles of coordinating a saga. The decision axis is VISIBILITY vs. COUPLING.
Orchestration: the process lives in one place (you know where each order is and why it
stalled), at the cost of an orchestrator that knows everyone. Choreography: maximum autonomy
and decoupling, at the cost of the global flow being written nowhere — good for simple
cascades, dangerous for long processes. Practical heuristic on the slide; and mixing is
legitimate (orchestrate the core, choreograph the peripheral reactions).
-->

---

## La idea del Outbox

Un servicio debe **guardar en su BD y publicar un evento**. No hay transacción que cubra BD *y* broker: si guarda y se cae antes de publicar → saga colgada; si publica y falla el guardado → el mundo cree algo falso.

```
┌───────────── Servicio Pedidos ─────────────┐
│  TX local: INSERT pedido + INSERT outbox   │ ← atómico ✔
│                 ┌────────┐                 │
│                 │ outbox │ ──► relay ──────┼──► broker ──► suscriptores
│                 └────────┘  (reintenta)    │
└─────────────────────────────────────────────┘
```

<!--
ES: El outbox resuelve un problema sutil pero OMNIPRESENTE: "guardar y publicar" no es
atómico porque BD y broker son dos sistemas. El truco: meter el evento en una tabla outbox
DENTRO de la misma transacción local que el cambio de negocio (eso sí es atómico). Un relay
aparte lee la outbox y publica, reintentando. Así nunca hay "pedido sin evento". Es el
patrón de fiabilidad más importante de las arquitecturas de eventos. Enlaza con Ejercicio
3.5.

EN: The outbox solves a subtle but OMNIPRESENT problem: "save and publish" isn't atomic
because the DB and the broker are two systems. The trick: insert the event into an outbox
table INSIDE the same local transaction as the business change (that IS atomic). A separate
relay reads the outbox and publishes, retrying. So there's never an "order without event".
It's the most important reliability pattern of event-driven architectures. Links to Exercise
3.5.
-->

---

## Outbox → al-menos-una-vez → idempotencia

El relay puede **reenviar** tras un fallo → la entrega es **al-menos-una-vez**.

→ Los consumidores deben ser **idempotentes:** procesar el mismo evento dos veces sin daño.

- **Deduplicar por `event_id`** (marca de procesado en la misma TX que el efecto), o
- **Idempotencia natural** por clave de negocio (`INSERT … ON CONFLICT DO NOTHING`)

> **Outbox (productor) + idempotencia (consumidor)** = el fundamento de fiabilidad de casi toda arquitectura de eventos.

<!--
ES: La consecuencia inevitable del outbox (y de los brokers en general): entrega
al-menos-una-vez, nunca exactamente-una. Por eso el consumidor TIENE que ser idempotente, o
duplicará facturas/cobros. Dos técnicas: deduplicar por id de evento (¡la marca de procesado
debe ir en la MISMA transacción que el efecto, o reaparece el problema del productor!), o
apoyarse en una restricción de unicidad de negocio. El dúo outbox+idempotencia es el estándar
de facto. Idempotencia se trata más a fondo en la Sesión 4.

EN: The inevitable consequence of the outbox (and brokers in general): at-least-once
delivery, never exactly-once. So the consumer MUST be idempotent, or it'll duplicate
invoices/charges. Two techniques: dedupe by event id (the processed-marker must go in the
SAME transaction as the effect, or the producer's problem reappears!), or lean on a business
uniqueness constraint. The outbox+idempotency duo is the de-facto standard. Idempotency is
covered more deeply in Session 4.
-->

---

## Consultas e informes entre servicios

Sin JOINs globales, ¿"los 10 clientes con más pedidos este mes"?

1. **Composición de APIs:** el llamante consulta varios servicios y combina. Bien para pocas entidades; mal para agregaciones masivas
2. **Vista materializada / CQRS:** un componente se **suscribe a los eventos** y mantiene una BD de lectura ya cruzada. Eventualmente consistente (suele bastar para informes)
3. **Plataforma analítica:** informes pesados → data warehouse alimentado por eventos, no contra las BDs operacionales

> Las consultas que cruzan dominios son **otro consumidor de eventos**, no una excusa para volver a la BD compartida.

<!--
ES: La objeción número uno a "BD por servicio" es "¿y mis informes que cruzan todo?". La
respuesta: no vuelvas a la BD compartida — trata el reporting como otro consumidor de
eventos. Para pocas entidades, componer por API; para consultas cruzadas, una vista
materializada que se alimenta de eventos (CQRS a nivel de sistema); para analítica pesada, un
warehouse. La regla cierra la sesión: los eventos que ya publicas para las sagas sirven
también para construir las vistas de lectura.

EN: The number-one objection to "DB per service" is "what about my reports that cross
everything?". The answer: don't go back to the shared DB — treat reporting as another event
consumer. For few entities, compose via API; for cross queries, a materialized view fed by
events (system-level CQRS); for heavy analytics, a warehouse. The rule closes the session:
the events you already publish for sagas also serve to build the read views.
-->

---

<!-- _class: lead -->

# Ejercicio 3.4 + 3.5
## Diseñar una saga
## ¿Dónde falla esto? (outbox e idempotencia)

*(ver cuaderno de ejercicios)*

<!--
ES: Pausa para ejercicios finales. 3.4: ViajesTotal (vuelo+hotel+cobro) — diseñar la saga
orquestada con compensaciones; clave: el ORDEN de los pasos minimiza el coste esperado de
compensar (hotel gratis antes que vuelo de 10 €; cobro lo más tarde posible), y los 10 € los
decide negocio (las compensaciones no son rollbacks gratis). 3.5: encontrar los dos fallos de
fiabilidad (commit sin publicar; evento duplicado) y corregirlos con outbox + idempotencia.
~30 min con puesta en común.

EN: Final exercise break. 3.4: ViajesTotal (flight+hotel+charge) — design the orchestrated
saga with compensations; key: step ORDER minimizes the expected cost of compensating (free
hotel before the 10 € flight; charge as late as possible), and the 10 € is a business
decision (compensations aren't free rollbacks). 3.5: find the two reliability bugs (commit
without publishing; duplicate event) and fix them with outbox + idempotency. ~30 min with
group discussion.
-->

---

## Resumen de la sesión

- El corte sigue al **negocio:** bounded contexts con lenguaje ubicuo; un término con dos significados = una frontera
- **Context maps** para gobernar relaciones; **core/supporting/generic** para repartir esfuerzo (lo genérico se compra)
- Granularidad: ante la duda, **cortar grueso** (dividir es barato, fusionar es caro)
- Sin **BD por servicio** no hay despliegue independiente: propiedad → esquemas → eliminar cruces → separar
- Sin transacciones distribuidas: **consistencia eventual** con semántica de negocio, **sagas** con compensaciones, **outbox** + **idempotencia**

<!--
ES: El destilado. Si solo se llevan dos ideas: (1) los límites correctos siguen al negocio
(DDD), y un corte malo hace inmanejable todo lo demás; (2) partir los datos es el trabajo de
verdad, y se paga con consistencia eventual gestionada por semántica de negocio. La Sesión 4
cierra el curso: comunicación, resiliencia, observabilidad y el lado organizativo (Conway),
más la hoja de ruta final.

EN: The distillate. If they take away two ideas: (1) the right boundaries follow the business
(DDD), and a bad cut makes everything else unmanageable; (2) splitting the data is the real
work, paid for with eventual consistency managed by business semantics. Session 4 closes the
course: communication, resilience, observability and the organizational side (Conway), plus
the final roadmap.
-->

---

<!-- _class: lead -->

# ¿Preguntas?

**Próxima sesión:** Comunicación, resiliencia y organización
*Sync/async · Circuit breaker · Observabilidad · Conway · Team Topologies*

<!--
ES: Abrir turno de preguntas. Recordar dónde está el material y el cuaderno. Puente: hoy
hemos decidido DÓNDE cortar y cómo partir los datos; la Sesión 4 hace que el resultado sea
robusto en producción (resiliencia, observabilidad) y sostenible en la organización (Conway,
Team Topologies), y lo junta todo en una hoja de ruta.

EN: Open the floor. Remind them where the material and workbook are. Bridge: today we decided
WHERE to cut and how to split the data; Session 4 makes the result robust in production
(resilience, observability) and sustainable in the organization (Conway, Team Topologies),
and ties it all into a roadmap.
-->
