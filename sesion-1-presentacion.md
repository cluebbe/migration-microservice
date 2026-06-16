---
marp: true
theme: default
paginate: true
header: 'Migración de Monolitos a Microservicios'
footer: 'Sesión 1 — Fundamentos y la justificación de migrar'
style: |
  section {
    font-size: 26px;
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

EN: Problems appear with the scale of the SYSTEM and of the ORGANIZATION.
Big ball of mud (Foote & Yoder, 1997): a system with no discernible architecture,
spaghetti code at system scale. Note: spaghetti = disorder WITHIN a module;
ball of mud = lack of structure BETWEEN modules.
Stress the organizational bottleneck: it's the real #1 reason to migrate.
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
| Escalado selectivo | Operación mucho más compleja |
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
ES: Los microservicios resuelven sobre todo un problema ORGANIZATIVO. Con 1-2 equipos
ese problema no existe. La madurez operativa define hasta dónde puedes llegar sin
ahogarte. Insistir en el híbrido deliberado: no es vergonzoso, suele ser óptimo.

EN: Microservices mostly solve an ORGANIZATIONAL problem. With 1-2 teams that problem
doesn't exist. Operational maturity defines how far you can go without drowning.
Insist on the deliberate hybrid: it's not shameful, it's often optimal.
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

EN: Migrating is a MEANS, never an end. If the goal is vague ("modernize"), there
will be no stopping criterion and no way to prioritize.
Lead time = time from commit to production (a DORA metric).
MTTR = mean time to recovery after an incident.
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
ES: La plataforma (CI/CD, observabilidad, plantillas) es lo que hace que la migración
escale. Si cada servicio cuesta como el primero, nunca terminarás.

EN: The platform (CI/CD, observability, templates) is what makes the migration scale.
If every service costs as much as the first one, you'll never finish.
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

EN: Decomposition says WHERE to cut; migration says HOW to execute the cut safely;
operation makes the result livable. A real migration combines all three.
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
