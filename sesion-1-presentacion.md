---
marp: true
theme: default
paginate: true
header: '![h:34px](nobleprog-logo.png) Tecnología de Punta Formación SL'
footer: 'Sesión 1 — Fundamentos y la justificación de migrar'
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

![w:360](logo.png)

# Sesión 1
## Fundamentos y la justificación de migrar

**Migración de Monolitos a Microservicios**
Carlos Georg Lübbe · 3 h

<!--
ES: Bienvenida. Esta primera sesión es la base conceptual de todo el curso.
El objetivo no es vender microservicios, sino dar un vocabulario común y un
encuadre honesto de cuándo (y cuándo no) migrar. Mucho de lo que veremos hoy
es "cómo evitar meterse en un lío".

EN: Welcome. This first session is the conceptual foundation of the whole course.
The goal is not to sell microservices, but to give a shared vocabulary and an
honest framing of when (and when not) to migrate. A lot of what we cover today
is "how to avoid getting into a mess".
-->

---

## Objetivo de hoy

Establecer un **vocabulario común** y un encuadre **honesto** de cuándo —y cuándo no— migrar.

1. Monolitos vs. microservicios: qué son de verdad
2. Más allá de la dicotomía: el **espectro** arquitectónico
3. Justificar la división (y el anti-patrón a evitar)
4. Fundamentos de estrategia de migración
5. El panorama de patrones del curso

<!--
ES: Recorrido de la sesión. Avisar de que habrá ejercicios intercalados.
Recordar que el enfoque es conceptual: las tecnologías solo como ejemplos.

EN: Roadmap of the session. Warn them there will be exercises along the way.
Remind them the focus is conceptual: technologies only as examples.
-->

---

<!-- _class: lead -->

# 1. Monolitos vs. microservicios

---

## ¿Qué es un monolito?

El término se usa para **dos ideas distintas**:

- **Unidad de despliegue:** toda la app se compila y despliega como un solo artefacto (WAR, binario, imagen)
- **Unidad de organización del código:** todo el código en un mismo proyecto

> Suelen ir juntas, pero **no son lo mismo**.
> En el curso, "monolito" = **unidad de despliegue única** — porque es el despliegue lo que los microservicios cambian.

<!--
ES: Punto clave: separar despliegue de organización. Contraejemplos: un monolito
modular (bien organizado, un solo artefacto) y un monorepo (un repo, varios
artefactos). La confusión entre ambas acepciones está detrás de muchos malentendidos.

EN: Key point: separate deployment from organization. Counter-examples: a modular
monolith (well organized, single artifact) and a monorepo (one repo, several
artifacts). Confusing these two meanings is behind many misunderstandings.
-->

---

## El monolito NO es intrínsecamente malo

| Ventaja | Por qué |
|---|---|
| Simplicidad operativa | Un solo proceso que desplegar y monitorizar |
| Transacciones ACID | Una sola BD: consistencia "gratis" |
| Refactorización fácil | El IDE ve todo el código |
| Latencia mínima | Llamadas internas = invocación de método, no red |
| Depuración sencilla | Un stack trace cuenta toda la historia |

<!--
ES: Importante empezar por aquí para no caer en el hype. El monolito tiene ventajas
REALES. Muchas se pierden al distribuir. Preguntar a la sala: ¿quién tiene un
monolito hoy? ¿qué de esto valoran?

EN: Important to start here so we don't fall for the hype. The monolith has REAL
advantages. Many are lost when you distribute. Ask the room: who has a monolith
today? Which of these do they value?
-->

---

## ...pero sufre con la escala

- **Acoplamiento creciente** → *big ball of mud* (gran bola de barro)
- **Despliegues acoplados:** un cambio pequeño obliga a redesplegar todo
- **Escalado grueso:** escalar todo aunque solo un módulo lo necesite
- **Cuellos de botella organizativos:** muchos equipos, un código, "trenes de release"
- **Atadura tecnológica:** una sola pila para todos los problemas

<!--
ES: Los problemas aparecen con la escala del SISTEMA y de la ORGANIZACIÓN.
Big ball of mud (Foote & Yoder, 1997): sistema sin arquitectura discernible,
código espagueti a escala de sistema. Ojo: espagueti = desorden DENTRO de un
módulo; bola de barro = ausencia de estructura ENTRE módulos.
Subrayar el cuello de botella organizativo: es el motivo nº1 real para migrar.

Tren de release: como el monolito es una única unidad de despliegue, nadie
despliega su cambio por separado; todos los cambios listos "viajan" en el mismo
tren que sale en fecha fija (p.ej. cada 2 semanas). El que termina antes espera
al tren; un cambio roto puede retrasar o tumbar el tren entero (vuelve atrás
TODO). Es un SÍNTOMA del despliegue acoplado, no una buena práctica elegida.
Con microservicios cada equipo tiene su propio tren —o ninguno: despliega cuando
quiere.

Atadura tecnológica: un solo artefacto = una sola pila (lenguaje, runtime,
framework, BD) para TODOS los problemas, aunque no encaje. No puedes reescribir
un módulo intensivo en CPU en Go, ni usar una BD de grafos para una carga que lo
pediría, sin migrar la app entera. Más adelante esto se convierte en "libertad
tecnológica" en la slide del trade-off.

EN: Problems appear with the scale of the SYSTEM and of the ORGANIZATION.
Big ball of mud (Foote & Yoder, 1997): a system with no discernible architecture,
spaghetti code at system scale. Note: spaghetti = disorder WITHIN a module;
ball of mud = lack of structure BETWEEN modules.
Stress the organizational bottleneck: it's the real #1 reason to migrate.

Release train: because the monolith is a single deployment unit, nobody ships
their change on its own; every ready change "rides" the same train that leaves on
a fixed schedule (e.g. every 2 weeks). Whoever finishes early waits for the train;
one broken change can delay or roll back the WHOLE train (everything goes back).
It's a SYMPTOM of coupled deployment, not a best practice you chose. With
microservices each team gets its own train —or none: deploy on demand.

Technology lock-in: one artifact = one stack (language, runtime, framework, DB)
for EVERY problem, even when it's a poor fit. You can't rewrite a CPU-heavy module
in Go, or use a graph DB for a workload that wants one, without migrating the
whole app. Later this flips into "technology freedom" on the trade-off slide.
-->

---

## ¿Qué es un microservicio?

Servicio **desplegable de forma independiente**, modelado en torno a un **dominio de negocio**, que oculta sus datos tras una interfaz.

1. **Despliegue independiente** ← *la* propiedad definitoria
2. **Modelado por negocio** (pedidos, facturación), no por capas
3. **Propiedad de sus datos** (nadie toca su BD)
4. **Autonomía del equipo**

<!--
ES: La propiedad nº1 es la que importa: si no puedo desplegar A sin coordinar con B,
no tengo microservicios. Las otras tres existen para hacer posible la primera.
"Modelado por negocio" lo retomaremos en la Sesión 3 con DDD.

EN: Property #1 is the one that matters: if I can't deploy A without coordinating
with B, I don't have microservices. The other three exist to make the first one
possible. "Modeled by business" we'll revisit in Session 3 with DDD.
-->

---

<!-- _class: lead -->

## "Micro" no es tamaño en líneas de código
## Es **superficie de acoplamiento**

La pregunta correcta no es *"¿cuántas líneas?"*
sino *"¿puedo desplegarlo sin pedir permiso?"*

<!--
ES: Idea fuerza de la sesión. Repetirla. Mucha gente cree que microservicio = pocas
líneas y acaba con 200 servicios diminutos acoplados (monolito distribuido, que
veremos luego). El tamaño es secundario.

EN: Core idea of the session. Repeat it. Many people think microservice = few lines
and end up with 200 tiny coupled services (the distributed monolith, which we'll
see later). Size is secondary.
-->

---

## Lo que se gana y lo que se paga

| Se gana | Se paga |
|---|---|
| Despliegue independiente, releases frecuentes | Sistema distribuido: latencia, fallos parciales |
| Escalado selectivo (pagas solo lo que escala) | Operación mucho más compleja |
| Autonomía de equipos | Contratos y versionado entre servicios |
| Aislamiento de fallos (potencial) | Consistencia sin transacciones globales |
| Libertad tecnológica | Coste de plataforma (CI/CD, gateway...) |

**Se cambia complejidad local por complejidad distribuida.**

<!--
ES: Este es el trade-off central. No hay almuerzo gratis. Cada fila de la izquierda
tiene su precio a la derecha. La decisión de migrar = decidir si las ganancias
justifican los costes EN TU CONTEXTO.

EN: This is the central trade-off. No free lunch. Each row on the left has its price
on the right. The decision to migrate = deciding whether the gains justify the
costs IN YOUR CONTEXT.

Matiz sobre coste (ES): hay que separar DOS costes distintos. El "Coste de plataforma"
(derecha) es una inversión FIJA y de entrada (CI/CD, gateway, observabilidad): la pagas
para poder operar, no depende de la carga. El coste de ESCALAR bajo carga es otra cosa:
ahí el microservicio gana, porque escalas solo la parte caliente en vez de replicar todo
el monolito. Por eso "Escalado selectivo" lleva ahora "pagas solo lo que escala".

Cost nuance (EN): keep TWO costs separate. "Platform cost" (right) is a FIXED, upfront
investment (CI/CD, gateway, observability) — you pay it to operate at all, independent of
load. Scaling cost under load is different: microservices win there because you scale only
the hot part instead of replicating the whole monolith. Hence "Selective scaling (you pay
only for what you scale)".
-->

---

## Las falacias de la computación distribuida

*Peter Deutsch, Sun Microsystems*

Lo que deja de ser cierto al pasar por la red:

- La red **no** es fiable
- La latencia **no** es cero
- El ancho de banda **no** es infinito
- La red **no** es segura
- La topología **cambia**...

<!--
ES: Al distribuir, todas estas suposiciones que el monolito daba por buenas se rompen.
Cada una se traduce en trabajo extra (reintentos, timeouts, seguridad, descubrimiento)
que veremos en la Sesión 4. Mensaje: la red es un nuevo enemigo.

EN: When you distribute, all these assumptions the monolith took for granted break.
Each one translates into extra work (retries, timeouts, security, discovery) that
we'll see in Session 4. Message: the network is a new enemy.
-->

---

<!-- _class: lead -->

# 2. Más allá de la dicotomía
## El espectro arquitectónico

---

## No es binario: es un espectro

```
Big ball of mud
   → Monolito por capas
      → Monolito modular
         → SOA / servicios gruesos
            → Microservicios
```

No hay que elegir entre "monolito caótico" y "200 microservicios".

<!--
ES: Mensaje central de la sección. La narrativa de izquierda a derecha:
primero aparece el ORDEN (capas), luego el orden sigue al NEGOCIO (modular),
luego se DISTRIBUYE (SOA), y finalmente se DESCENTRALIZA (microservicios).

EN: Core message of the section. The left-to-right narrative: first comes ORDER
(layers), then order follows the BUSINESS (modular), then it gets DISTRIBUTED (SOA),
and finally DECENTRALIZED (microservices).
-->

---

## Los peldaños (1/2)

**Big ball of mud** — sin arquitectura discernible; no hay costuras por donde cortar.

**Monolito por capas** — hay orden, pero *técnico* (presentación / lógica / datos). Un cambio de negocio cruza todas las capas; las capas no dan fronteras extraíbles.

**Monolito modular** — corte *vertical* por dominios; cada módulo con su interfaz. Costuras sí extraíbles → peldaño previo natural a microservicios.

<!--
ES: La diferencia capas vs modular es la DIRECCIÓN del corte, no el grado de orden.
Horizontal (técnico) vs vertical (negocio). Esta distinción reaparece en el
ejercicio 1.2 (monolito distribuido por capas) y en la Sesión 3 (DDD).

EN: The difference layers vs modular is the DIRECTION of the cut, not the degree of
order. Horizontal (technical) vs vertical (business). This distinction reappears in
exercise 1.2 (distributed monolith split by layers) and in Session 3 (DDD).
-->

---

## Los peldaños (2/2)

**SOA / servicios gruesos** — pocos servicios grandes, ESB central, modelo canónico corporativo, releases coordinadas. *Lección: distribuir sin descentralizar = cuellos de botella del monolito + red.*

**Microservicios** — grano fino por dominio, *smart endpoints / dumb pipes*, modelo por contexto, BD por servicio, autonomía de equipo. "SOA de grano fino y descentralizada".

<!--
ES: SOA: merece mencionarse porque muchos en la sala lo vivieron (SOAP, ESB, WS-*).
Es historia, no destino. La lección de la SOA es la misma del monolito distribuido.
No profundizar: 30 segundos si vamos justos de tiempo.

EN: SOA: worth mentioning because many in the room lived through it (SOAP, ESB, WS-*).
It's history, not a destination. The lesson of SOA is the same as the distributed
monolith. Don't go deep: 30 seconds if we're short on time.

SOA vs microservicios (ES): ambos parten el sistema en servicios sobre la red; la
diferencia es DÓNDE vive la inteligencia y cuánto descentralizas. SOA = pocos servicios
gruesos, ESB central inteligente (smart pipes), modelo de datos canónico, releases
coordinadas, gobierno centralizado. Microservicios = grano fino por dominio, smart
endpoints / dumb pipes, BD por servicio, despliegue independiente, autonomía de equipo.
"Microservicios = SOA de grano fino y descentralizada." SOA distribuye pero no
descentraliza (el ESB es el nuevo punto central de acoplamiento). Frase de aula: "SOA
partió el sistema pero dejó el cerebro en el centro; los microservicios reparten el
cerebro entre los servicios."

SOA vs microservices (EN): both split the system into services over the network; the
difference is WHERE the smarts live and how decentralized you are. SOA = few coarse
services, central smart ESB (smart pipes), canonical data model, coordinated releases,
centralized governance. Microservices = fine-grained per domain, smart endpoints / dumb
pipes, database per service, independent deployment, team autonomy. "Microservices =
fine-grained, decentralized SOA." SOA distributes but doesn't decentralize (the ESB
becomes the new central coupling point).
-->

---

## El monolito modular

Una sola unidad de despliegue, pero módulos con **límites explícitos**:

- Interfaz pública pequeña, interior oculto
- Comunicación solo vía interfaces (idealmente verificado por herramientas)
- Esquema de datos propio por módulo (aunque compartan instancia)

**Es a la vez:** un destino válido (80% del beneficio sin el coste de distribuir) **y** el mejor punto de partida para migrar.

<!--
ES: Mensaje doble muy importante. Para muchas empresas el monolito modular ES el
destino correcto. Y para las que sí van a microservicios, es el trampolín.

EN: Very important double message. For many companies the modular monolith IS the
right destination. And for those who do go to microservices, it's the springboard.
-->

---

<!-- _class: lead -->

> Si no eres capaz de construir un **monolito modular** bien estructurado, los microservicios **no** lo van a arreglar:
> van a **distribuir el desorden** y añadirle **red**.

<!--
ES: Regla práctica para grabar a fuego. Conecta con un error clásico: huir hacia
microservicios para escapar de un monolito caótico, y acabar peor.

EN: A rule of thumb to burn into memory. Connects to a classic mistake: fleeing to
microservices to escape a chaotic monolith, and ending up worse.
-->

---

## ¿Dónde situarse en el espectro?

- **Tamaño y nº de equipos** ← el factor más determinante
- **Escalado diferencial** entre partes
- **Madurez operativa** (CI/CD, observabilidad) ← marca el límite *alcanzable*
- **Velocidad de cambio** y dónde se concentra

Y recordar:
1. La posición **no es permanente** (avanzar = migrar; retroceder = corregir)
2. **No tiene que ser uniforme** (híbridos deliberados son destino válido)
3. El movimiento tiene **orden natural** (pasar por el modular antes)

<!--
ES: Slide interactiva. Convertir cada factor en una pregunta al público y dejar que
SE SITÚEN ellos mismos. No dar la respuesta: que el grupo razone.

Preguntas por factor:
- Tamaño y equipos: "¿Cuántos equipos tocan y despliegan hoy el mismo código? ¿Se
  pisan al hacer release?" (Clave: con 1-2 equipos el problema organizativo NO existe.)
- Escalado diferencial: "¿Hay alguna parte que se satura mientras el resto está ocioso?
  ¿O escaláis todo el bloque a la vez?"
- Madurez operativa: "¿Podéis desplegar un servicio sin coordinar a media empresa?
  ¿Tenéis CI/CD real? ¿Si algo se rompe de noche entre varios componentes, sabéis DÓNDE
  mirar?" (Clave: la madurez marca el límite ALCANZABLE — sin ella te ahogas.)
- Velocidad de cambio: "¿Dónde se concentran los cambios? ¿El 80% toca siempre el mismo
  módulo, o está repartido?"

Cierre, otra ronda de preguntas:
- "¿Dónde diríais que estáis HOY en el espectro?"
- "¿Y dónde necesitáis estar de verdad?" (Recordar: la posición no es permanente, no
  tiene que ser uniforme, y el híbrido deliberado no es vergonzoso: suele ser óptimo.)

EN: Interactive slide. Turn each factor into a question for the audience and let them
PLACE THEMSELVES. Don't hand them the answer — make the group reason it out.

Questions per factor:
- Size & teams: "How many teams touch and deploy the same code today? Do they collide on
  release?" (Key: with 1-2 teams the organizational problem doesn't exist.)
- Differential scaling: "Is there a part that saturates while the rest sits idle? Or do
  you scale the whole block at once?"
- Operational maturity: "Can you deploy one service without coordinating half the
  company? Do you have real CI/CD? If something breaks at night across components, do you
  know WHERE to look?" (Key: maturity sets the ACHIEVABLE limit — without it you drown.)
- Rate of change: "Where does change concentrate? Does 80% always hit the same module, or
  is it spread out?"

Wrap-up, another round of questions:
- "Where would you say you are TODAY on the spectrum?"
- "And where do you actually need to be?" (Remind: position isn't permanent, needn't be
  uniform, and the deliberate hybrid isn't shameful — it's often optimal.)
-->

---

<!-- _class: lead -->

# Ejercicio 1.1 + 1.2
## ¿Monolito culpable o inocente?
## Detectar el monolito distribuido

*(ver cuaderno de ejercicios)*

<!--
ES: Pausa para ejercicios. 1.1: LibroExpress — distinguir problemas de madurez/calidad
de problemas que los microservicios sí resuelven. 1.2: PagoSeguro — reconocer
síntomas de monolito distribuido. ~20-25 min con puesta en común.

EN: Break for exercises. 1.1: LibroExpress — distinguish maturity/quality problems
from problems microservices actually solve. 1.2: PagoSeguro — recognize symptoms of
a distributed monolith. ~20-25 min including group discussion.
-->

---

<!-- _class: lead -->

# 3. Justificar la división

---

## Empezar por el porqué

> "Migramos porque queremos **X**, lo mediremos con **Y**, y sabremos que avanzamos cuando **Z**."

| Objetivo | Métrica posible |
|---|---|
| Acelerar la entrega | Frecuencia de despliegue, lead time |
| Escalar la organización | Nº equipos autónomos |
| Escalar una parte | Coste infra, latencia bajo carga |
| Aislar fallos | Radio de impacto, MTTR |

<!--
ES: Migrar es un MEDIO, nunca un fin. Si el objetivo es difuso ("modernizar"), no
habrá criterio de parada ni de prioridad.
Lead time = tiempo de commit a producción (métrica DORA).
MTTR = tiempo medio de recuperación tras un incidente.
Radio de impacto (blast radius) = cuánto del sistema se cae cuando falla una parte.
En el monolito es máximo (un fallo tumba todo el proceso: es el síntoma #3 de
LibroExpress, la búsqueda satura la CPU y cae la web entera); con servicios bien
aislados el fallo queda contenido. Se mide como el % de funcionalidad/usuarios/
transacciones afectados ante el fallo de un componente. Ojo: no baja solo —si la
división crea una cadena síncrona obligatoria, el radio no se reduce, se reparte;
reducirlo de verdad exige resiliencia (timeouts, circuit breakers), Sesión 4.

EN: Migrating is a MEANS, never an end. If the goal is vague ("modernize"), there
will be no stopping criterion and no way to prioritize.
Lead time = time from commit to production (a DORA metric).
MTTR = mean time to recovery after an incident.
Blast radius = how much of the system goes down when one part fails. In the
monolith it's maximal (one fault takes down the whole process: that's LibroExpress
symptom #3, search saturates the CPU and the entire site goes down); with
well-isolated services the failure stays contained. Measured as the % of
functionality/users/transactions affected when a component fails. Note: it doesn't
drop by itself —if the split creates a mandatory synchronous chain, the radius
isn't reduced, just spread; truly reducing it requires resilience (timeouts,
circuit breakers), Session 4.
-->

---

## Cuándo NO merece la pena migrar

- **Equipo pequeño** (< 10–15): el coste operativo se come la ganancia
- **Dominio poco entendido:** los límites cambiarán (caro moverlos entre servicios)
- **El problema real es otro:** falta de tests, despliegues manuales → distribuir lo amplifica
- **Madurez operativa insuficiente**
- **Producto de vida corta**

**Alternativas antes de dividir:** re-platforming · modularización · extraer 1–2 servicios y parar

<!--
ES: Esta slide es quizá la más honesta del curso. La mayoría de "fracasos de
microservicios" son organizaciones que no debían haber migrado.
Re-platforming = mover a mejor infra sin trocear. A menudo resuelve el escalado.

EN: This slide is perhaps the most honest of the course. Most "microservice failures"
are organizations that should never have migrated.
Re-platforming = move to better infra without breaking apart. Often solves scaling.
-->

---

## El anti-patrón: el monolito distribuido

Lo **peor** posible: muchos servicios que deben **desplegarse juntos**.

Síntomas:
- Un cambio típico toca 3–4 servicios
- Despliegues coordinados ("el viernes desplegamos todos")
- Servicios que comparten BD
- Cadenas largas de llamadas síncronas

<!--
ES: Causa raíz: dividir por capas técnicas, dividir antes de entender el dominio,
no romper la BD compartida. Se paga TODO el coste de la distribución sin ganar
la independencia de despliegue. Lo peor de los dos mundos.

EN: Root cause: splitting by technical layers, splitting before understanding the
domain, not breaking the shared DB. You pay ALL the cost of distribution without
gaining deployment independence. The worst of both worlds.
-->

---

<!-- _class: lead -->

## Test rápido

Si para entregar una funcionalidad media necesitas **coordinar despliegues** de varios servicios...

### no tienes microservicios:
### tienes un monolito con latencia de red

<!--
ES: Test de bolsillo para llevarse a casa. Sirve para auditar cualquier arquitectura
"de microservicios" existente. Conecta directo con el ejercicio 1.2.

EN: A pocket test to take home. Useful to audit any existing "microservices"
architecture. Connects directly to exercise 1.2.
-->

---

<!-- _class: lead -->

# 4. Estrategia de migración

---

## Incremental vs. big-bang

**Big-bang** (reescribir todo y cambiar de golpe): casi siempre fracasa
- Objetivo móvil (el monolito sigue vivo)
- Cero valor durante meses/años
- Riesgo concentrado en un único corte

**Incremental** (pieza a pieza, sistema siempre vivo):
- Pasos pequeños y **reversibles**
- Valor entregado por el camino
- Lo viejo y lo nuevo **conviven durante años** ← es el plan, no un fallo

<!--
ES: La convivencia prolongada asusta a los gestores pero es lo correcto. Big-bang
lo veremos como patrón (y sus raras excepciones) en la Sesión 2.

EN: The prolonged coexistence scares managers but it's the right thing. Big-bang
we'll cover as a pattern (and its rare exceptions) in Session 2.
-->

---

## Gestión del riesgo

*El "cómo" del enfoque incremental: prácticas para que cada paso sea seguro y reversible.*

- **Empieza por algo que enseñe mucho y arriesgue poco** (no el corazón crítico)
- **Separa despliegue de activación** (feature flags, enrutado gradual)
- **Mide antes y después** (sin línea base no hay prueba de mejora)
- **Plan de retirada del legacy:** la migración termina cuando el código viejo **se borra**

<!--
ES: "Modo híbrido eterno" = coste permanente. El paso de BORRAR el legacy es el que
todo el mundo olvida. Sin línea base medida ANTES, no podrás demostrar que mejoró
nada (ni detectar regresiones).

EN: "Eternal hybrid mode" = permanent cost. The step of DELETING the legacy is the
one everyone forgets. Without a baseline measured BEFORE, you won't be able to prove
anything improved (nor detect regressions).
-->

---

## Secuenciar: la matriz valor × facilidad

*Si migras de forma incremental, el siguiente paso es decidir el orden: qué pieza extraer primero, cuál después y cuál no tocar.*

```
                     Facilidad de extraer
                   baja               alta
            ┌──────────────────┬──────────────────┐
       alto │ (2) INVERTIR     │ (1) EMPEZAR AQUÍ │
  Valor     │ después          │ victorias rápidas│
            ├──────────────────┼──────────────────┤
       bajo │ (4) ¿JAMÁS?      │ (3) RELLENO      │
            │ dejar en monolito│ no es progreso   │
            └──────────────────┴──────────────────┘
```

Criterios: acoplamiento · valor · volatilidad · riesgo · datos

<!--
ES: (1) empezar: alto valor + fácil → victorias que enseñan.
(2) invertir cuando haya oficio (suele ser el core).
(3) relleno: actividad sin progreso, no confundir.
(4) candidato a quedarse en el monolito DELIBERADAMENTE.
Que algo PUEDA extraerse no significa que DEBA. Conecta con ejercicio 1.4.

EN: (1) start here: high value + easy → wins that teach.
(2) invest once you have the craft (usually the core).
(3) filler: activity without progress, don't confuse the two.
(4) candidate to stay in the monolith DELIBERATELY.
That something CAN be extracted doesn't mean it SHOULD. Connects to exercise 1.4.
-->

---

## Buenas prácticas generales

- Trata la migración como **producto**, no como proyecto big-bang
- **No congeles el monolito** salvo en zonas de extracción activa
- Invierte pronto en la **plataforma** (el 2º servicio debe ser mucho más barato que el 1º)
- Comunica el **estado de la convivencia** (qué está en servicios, qué en el monolito, qué a medias)

<!--
ES: (1) Producto, no proyecto: backlog y métricas, no fecha de corte; encaja con lo incremental.
(2) No congelar el monolito: sigue evolucionando; solo se congela la pieza en extracción activa.
(3) Invertir PRONTO en plataforma (CI/CD, observabilidad, plantillas): si el 2º servicio cuesta como el 1º, no terminas.
(4) Comunicar la convivencia: qué está en servicios, en el monolito o a medias; sin mapa, bugs y trabajo duplicado.

EN: (1) Product, not project: backlog and metrics, not a cutover date; fits the incremental approach.
(2) Don't freeze the monolith: it keeps evolving; only the piece under active extraction is frozen.
(3) Invest EARLY in the platform (CI/CD, observability, templates): if the 2nd service costs like the 1st, you never finish.
(4) Communicate the coexistence: what's in services, in the monolith, or half-done; no map → bugs and duplicated work.
-->

---

<!-- _class: lead -->

# Ejercicio 1.3 + 1.4
## Objetivos y criterios de éxito
## Secuenciar la migración

*(ver cuaderno de ejercicios)*

<!--
ES: 1.3: reformular "ser más ágiles" en objetivo + métricas con línea base + criterio
de parada. 1.4: AseguraTodo — secuenciar qué extraer primero y qué dejar (o nunca),
usando la matriz valor × facilidad. ~25-30 min.

EN: 1.3: reframe "be more agile" into goal + metrics with a baseline + stopping
criterion. 1.4: AseguraTodo — sequence what to extract first and what to leave (or
never), using the value × ease matrix. ~25-30 min.
-->

---

<!-- _class: lead -->

# 5. El panorama de patrones

---

## El mapa del curso

**Migración** (Sesión 2) — *¿cómo muevo funcionalidad?*
Strangler · ACL · Branch by Abstraction · Parallel Run · Bubble Context

**Descomposición** (Sesión 3) — *¿dónde corto? ¿y los datos?*
DDD · bounded contexts · BD por servicio · Saga · Outbox

**Operación** (Sesión 4) — *¿cómo hago robusto el resultado?*
Sync/async · API Gateway · Circuit breaker · Observabilidad · Conway

<!--
ES: Descomposición dice DÓNDE cortar; migración dice CÓMO ejecutar el corte con
seguridad; operación hace vivible el resultado. Una migración real combina los tres.
Glosario rápido:
- Strangler: envolver el monolito y reemplazar funciones poco a poco hasta "estrangularlo".
- ACL (anti-corruption layer): capa traductora que aísla el modelo nuevo del legacy.
- Branch by Abstraction: meter una abstracción para cambiar la implementación sin big-bang.
- Parallel Run: correr viejo y nuevo a la vez y comparar antes de confiar en el nuevo.
- Bubble Context: crear un contexto nuevo y limpio junto al legacy, sin contaminarlo.
- DDD: diseñar el software según el dominio de negocio.
- Bounded context: límite donde un modelo es coherente; base para trazar servicios.
- BD por servicio: cada servicio dueño de sus datos; nadie toca la BD de otro.
- Saga: transacción distribuida por pasos con compensaciones (sin commit global).
- Outbox: publicar eventos fiablemente escribiéndolos en la misma transacción que los datos.
- Sync/async: llamada bloqueante (REST) vs. mensajería/eventos no bloqueante.
- API Gateway: puerta de entrada única que enruta y centraliza (auth, rate limit).
- Circuit breaker: corta llamadas a un servicio caído para evitar cascadas.
- Observabilidad: logs, métricas y trazas para entender el sistema en producción.
- Conway: la arquitectura acaba reflejando la estructura de comunicación de los equipos.

EN: Decomposition says WHERE to cut; migration says HOW to execute the cut safely;
operation makes the result livable. A real migration combines all three.
Quick glossary:
- Strangler: wrap the monolith and replace features bit by bit until it's "strangled".
- ACL (anti-corruption layer): translation layer that isolates the new model from the legacy.
- Branch by Abstraction: add an abstraction to swap the implementation without a big-bang.
- Parallel Run: run old and new together and compare before trusting the new one.
- Bubble Context: build a clean new context next to the legacy, without contaminating it.
- DDD: design the software around the business domain.
- Bounded context: boundary where one model is coherent; basis for drawing services.
- DB per service: each service owns its data; nobody touches another's DB.
- Saga: distributed transaction as steps with compensations (no global commit).
- Outbox: publish events reliably by writing them in the same transaction as the data.
- Sync/async: blocking call (REST) vs. non-blocking messaging/events.
- API Gateway: single entry point that routes and centralizes (auth, rate limit).
- Circuit breaker: cuts off calls to a downed service to prevent cascades.
- Observability: logs, metrics and traces to understand the system in production.
- Conway: the architecture ends up mirroring the teams' communication structure.
-->

---

## Resumen de la sesión

- Monolito y microservicios son **puntos de un espectro**; el modular es destino válido
- Los microservicios compran **autonomía** pagando **complejidad distribuida**
- Antes de migrar: **objetivo, métricas con línea base, criterio de parada**
- Evita el **monolito distribuido** (test: ¿cuántos servicios toca un cambio?)
- Migración sana = **incremental**, reversible, con valor por el camino y **borrado** del legacy

<!--
ES: Cierre. Estas cinco ideas son el destilado de la sesión. La siguiente sesión:
la caja de herramientas de patrones de migración.

EN: Closing. These five ideas are the distillate of the session. Next session:
the toolbox of migration patterns.
-->

---

<!-- _class: lead -->

# ¿Preguntas?

**Próxima sesión:** Patrones de migración
*Strangler Fig · ACL · Branch by Abstraction · Parallel Run · Bubble Context*

<!--
ES: Abrir turno de preguntas. Recordar dónde está el material y el cuaderno de ejercicios.

EN: Open the floor for questions. Remind them where the material and the exercise
notebook are.
-->
