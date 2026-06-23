---
marp: true
theme: default
paginate: true
header: '![h:34px](nobleprog-logo.png) Tecnología de Punta Formación SL'
footer: 'Sesión 2 — Patrones de migración'
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

# Sesión 2
## Patrones de migración

**Migración de Monolitos a Microservicios**
Carlos Georg Lübbe · 3 h

<!--
ES: Bienvenida a la segunda sesión. En la primera vimos el PORQUÉ y el CUÁNDO;
hoy vemos el CÓMO. El objetivo es construir una caja de herramientas de patrones
de migración probados, entendiendo de cada uno sus ventajas, sus límites y dónde
encaja. No hay un patrón "ganador": el oficio está en elegir y combinar.

EN: Welcome to session two. The first session covered WHY and WHEN; today is the
HOW. The goal is to build a toolbox of proven migration patterns, understanding
each one's strengths, limits and where it fits. There's no single "winning"
pattern: the craft is in choosing and combining them.
-->

---

## Objetivo de hoy

Construir una **caja de herramientas** completa de patrones de migración.

1. **Strangler Fig** — interceptar y reemplazar desde el borde
2. **ACL** — traducir sin que el legacy contamine lo nuevo
3. **Big-Bang** — por qué casi nunca, y el "casi"
4. **Branch by Abstraction** — reemplazar lo enterrado
5. **Parallel Run** — ganar confianza con tráfico real
6. **Bubble Context** — remodelar el dominio dentro del legacy
7. **Comparativa** — elegir y combinar

<!--
ES: Recorrido de la sesión. Avisar de que habrá ejercicios intercalados tras cada
bloque de patrones. Idea transversal: estos patrones existen para TROCEAR la
migración en pasos pequeños, seguros y reversibles. Vistazo rápido a cada uno:
- Strangler Fig: meter una fachada delante del monolito y mover funcionalidad al
  servicio nuevo ruta a ruta, hasta que el viejo queda vacío.
- ACL (capa anticorrupción): capa de traducción en la frontera para que el modelo
  legacy no se filtre y contamine el modelo nuevo. Es transversal: aparece dentro de
  casi todos los demás.
- Big-Bang: reescribirlo todo y conmutar de golpe. Casi siempre mala idea; el "casi"
  son piezas pequeñas, congeladas y bien especificadas.
- Branch by Abstraction: el "strangler de interior" — una abstracción con dos
  implementaciones y un flag, para reemplazar algo enterrado sin borde interceptable.
- Parallel Run: ejecutar viejo y nuevo a la vez con tráfico real y comparar; valida
  lo crítico antes de confiar en lo nuevo.
- Bubble Context: arrancar un contexto nuevo y limpio (protegido por una ACL) para
  remodelar un dominio dentro del entorno legacy.
- Comparativa: cómo elegir entre ellos y, sobre todo, cómo combinarlos (no compiten).

EN: Roadmap of the session. Warn them there will be exercises after each block of
patterns. Cross-cutting idea: these patterns exist to BREAK the migration into
small, safe, reversible steps. Quick gloss of each one:
- Strangler Fig: put a facade in front of the monolith and move functionality to the
  new service route by route, until the old one is empty.
- ACL (anticorruption layer): a translation layer at the boundary so the legacy model
  doesn't leak in and pollute the new model. It's cross-cutting: it shows up inside
  almost all the others.
- Big-Bang: rewrite everything and switch over at once. Almost always a bad idea; the
  "almost" is small, frozen, well-specified pieces.
- Branch by Abstraction: the "strangler of the interior" — an abstraction with two
  implementations and a flag, to replace something buried with no interceptable edge.
- Parallel Run: run old and new at once with real traffic and compare; validates the
  critical stuff before trusting the new one.
- Bubble Context: start a clean new context (protected by an ACL) to reshape a domain
  inside the legacy environment.
- Comparison: how to choose among them and, above all, how to combine them (they
  don't compete).
-->

---

<!-- _class: lead -->

# 1. Patrón Estrangulador
## Strangler Fig

---

## Concepto y origen

La **higuera estranguladora**: crece alrededor del árbol hasta reemplazarlo, sin un corte abrupto. *(Martin Fowler, 2004.)*

> El sistema nuevo **crece alrededor** del viejo hasta sustituirlo.

La idea esencial: **interceptar las peticiones** al monolito y, funcionalidad a funcionalidad, redirigirlas al sistema nuevo.

Tres pasos en bucle:
1. **Identificar** una funcionalidad
2. **Implementarla** en un servicio nuevo
3. **Redirigir** sus llamadas al servicio nuevo

<!--
ES: La metáfora es potente: en ningún momento talas el árbol viejo de un hachazo.
El sistema viejo sigue sosteniendo todo lo que aún no has reemplazado. Fowler la
popularizó en 2004; sigue siendo el patrón de migración por defecto.
El bucle de 3 pasos se repite una y otra vez, una funcionalidad cada vez.

EN: The metaphor is powerful: you never chop down the old tree in one blow. The old
system keeps holding up everything you haven't replaced yet. Fowler popularized it
in 2004; it's still the default migration pattern. The 3-step loop repeats over and
over, one feature at a time.
-->

---

## Arquitectura: la fachada de enrutado

```
                      ┌──────────────┐
   Clientes ────────► │   Fachada    │
                      │  (proxy /    │
                      │   gateway)   │
                      └──────┬───────┘
              /pedidos/*     │      todo lo demás
            ┌────────────────┴──────────────┐
            ▼                               ▼
   ┌─────────────────┐             ┌────────────────┐
   │ Servicio nuevo  │             │    Monolito    │
   │   "Pedidos"     │ ──(a veces)─►   (legacy)     │
   └─────────────────┘             └────────────────┘
```

La pieza clave es la **fachada** (proxy/gateway) delante del monolito.

<!--
ES: La fachada es el corazón del patrón. Por una ruta (/pedidos/*) va al servicio
nuevo; el resto sigue yendo al monolito. La flecha "a veces" indica que el servicio
nuevo todavía puede necesitar leer datos del monolito mientras los datos no se han
separado (eso se resuelve en la Sesión 3).

EN: The facade is the heart of the pattern. One route (/pedidos/*) goes to the new
service; everything else still goes to the monolith. The "sometimes" arrow shows the
new service may still need to read data from the monolith while the data isn't yet
split (resolved in Session 3).
-->

---

## Propiedades clave

- **Paso 0:** la fachada se introduce **antes** de extraer nada (enruta 100 % al monolito). Bajo riesgo, infraestructura lista.
- **Redirección gradual** (*canary*): 1 % → 10 % → 100 % del tráfico.
- **Rollback trivial:** volver a apuntar la ruta al monolito.
- El corte natural es por **funcionalidad vertical** visible (URLs, operaciones de API), no por capas internas.

<!--
ES: Estas cuatro propiedades son las que hacen al Strangler tan seguro. El paso 0 es
clave y a menudo se olvida: meter el proxy primero, sin cambiar comportamiento. La
gradualidad y el rollback trivial son lo que distingue migrar de jugársela. "Corte
vertical" = una funcionalidad completa de punta a punta, no la capa de datos sola.
Explicar "canary" (primera vez que aparece): viene del "canario en la mina" — los
mineros bajaban un canario, más sensible a los gases tóxicos; si caía, les avisaba del
peligro ANTES de que les afectara. Aquí igual: liberas a una porción pequeña (la
"canary") y, si algo va mal, solo afecta a ese grupo, que actúa de alarma temprana
antes de exponer a todos. Eso es el 1%→10%→100%; "solo empleados internos" es elegir
QUIÉN forma esa primera cohorte.

EN: These four properties are what make the Strangler so safe. Step 0 is key and
often forgotten: put the proxy in first, with no behavior change. Graduality and
trivial rollback are what separate migrating from gambling. "Vertical cut" = a whole
feature end-to-end, not the data layer alone. Explain "canary" (first time it appears):
it comes from the "canary in a coal mine" — miners took a canary down, more sensitive
to toxic gas; if it collapsed it warned them of danger BEFORE it harmed them. Same here:
you release to a small slice (the "canary") and, if something goes wrong, only that group
is affected — they act as an early warning before you expose everyone. That's the
1%→10%→100%; "internal employees only" is choosing WHO that first cohort is.
-->

---

## Enrutado gradual: ¿cómo se elige la cohorte?

La ruta dice *qué* funcionalidad; un **predicado extra** sobre la petición dice *a quién* se le da lo nuevo:

1. **Identity / claims** — gateway tras el login: lee `role=employee` del JWT/sesión.
2. **Red / IP** — personal interno por VPN/IP de oficina → allowlist, sin identidad.
3. **Cabecera upstream** — auth en el borde sella `X-User-Type: internal`.
4. **Cookie / feature-flag** — cookie de cohorte para usuarios opt-in.
5. **Porcentaje** — hash del user/sesión → bucket del 1 %.

> (1) y (3) exigen que el gateway **vea la identidad**; si no, solo IP o %.

💡 Si el monolito gestiona su propia auth, su token es opaco al gateway → **externalizar el auth suele ser de las primeras extracciones**: desbloquea el enrutado por identidad.

<!--
ES: Esta slide resuelve la duda natural de "si la fachada enruta por ruta, ¿cómo
redirige 'solo a empleados internos'?". Respuesta: la ruta es el selector grueso (¿es
una petición de devoluciones?); encima se evalúa un predicado de cohorte. Cuatro
mecanismos habituales (claims, IP, cabecera, cookie) + el porcentaje del canary. El
matiz importante es el requisito: enrutar por identidad (1 y 3) necesita que el gateway
esté DESPUÉS del login o pueda leer el token; si está delante del auth, te quedan IP (2)
o porcentaje. Es el primo a nivel de gateway del feature flag de Branch by Abstraction:
misma pregunta ("¿quién recibe lo nuevo?"), una decidida en el borde por atributos de la
petición, la otra en proceso por configuración.
Sobre el punto 💡: si el monolito es el dueño del auth, su token/cookie es opaco para el
gateway, así que (1) y (3) no funcionan de fábrica — solo IP o porcentaje. Para
desbloquearlas: o le das al gateway capacidad de validar el token (secreto compartido /
introspección), o —mejor— externalizas el auth al borde. Por eso la autenticación es a
menudo de los primeros candidatos a extraer: no por ser la función de más valor, sino
porque DESBLOQUEA pasos posteriores (enrutado por identidad, que otros servicios confíen
en un principal verificado). Buen ejemplo del tema de la Sesión 1: el orden de extracción
importa, y a veces se extrae algo pronto para habilitar lo demás. (Enlaza con API Gateway
y concerns transversales de la Sesión 4.)

EN: This slide resolves the natural doubt: "if the facade routes by path, how does it
redirect 'only to internal employees'?". Answer: the path is the coarse selector (is this
a returns request?); on top of it you evaluate a cohort predicate. Four common mechanisms
(claims, IP, header, cookie) + the canary percentage. The key nuance is the prerequisite:
identity-based routing (1 and 3) needs the gateway to sit AFTER login or be able to read
the token; if it sits in front of auth, you're left with IP (2) or percentage. It's the
gateway-level cousin of Branch by Abstraction's feature flag: same question ("who gets the
new thing?"), one decided at the edge by request attributes, the other in-process by config.
On the 💡 point: if the monolith owns auth, its token/cookie is opaque to the gateway, so
(1) and (3) don't work out of the box — only IP or percentage. To unlock them: either give
the gateway token-validation ability (shared secret / introspection), or —better—
externalize auth to the edge. That's why authentication is often one of the first
extraction candidates: not because it's the highest-value feature, but because it UNBLOCKS
later steps (identity-based routing, other services trusting a verified principal). A good
example of the Session 1 theme: extraction order matters, and sometimes you extract
something early to enable everything else. (Links to API Gateway and cross-cutting concerns
in Session 4.)
-->

---

## Ventajas y limitaciones

**Ventajas**
- Incremental; el sistema funciona siempre
- Reversible en cada paso
- Progreso visible y medible (% de tráfico en lo nuevo)
- No exige tocar el interior del monolito

**Limitaciones**
- Necesita un **borde interceptable** (HTTP ideal); si está enterrado → *Branch by Abstraction*
- **Los datos no se estrangulan solos** (Sesión 3)
- La fachada es un punto único: fiable y observable
- Riesgo de **estrangulamiento eterno** si nadie prioriza terminar

<!--
ES: La limitación más importante de cara al resto de la sesión: el Strangler necesita
un borde donde interceptar. Cuando no lo hay, el patrón hermano es Branch by
Abstraction (sección 4). Y recordar siempre: estrangular el código NO estrangula los
datos. El "estrangulamiento eterno" es el equivalente aquí del "modo híbrido eterno".

EN: The most important limitation for the rest of the session: the Strangler needs an
edge to intercept at. When there's none, the sibling pattern is Branch by Abstraction
(section 4). And always remember: strangling the code does NOT strangle the data. The
"eternal strangling" is the equivalent here of the "eternal hybrid mode".
-->

---

## Ejemplo: extraer "devoluciones"

E-commerce monolítico → extraer gestión de devoluciones:

1. Gateway delante; todo el tráfico pasa por él (sin cambios)
2. `returns-service` con su propia BD **para lo que posee**; **todavía** lee datos maestros (clientes, pedidos) del monolito vía API interna *— los datos aún no están separados (Sesión 3)*
3. Redirigir `/devoluciones/*` solo a empleados internos (1 %) → observar → 100 %
   *(aquí se verifica; si el riesgo lo pide, comparando con el monolito vía Parallel Run → §5)*
4. **Borrar** el código de devoluciones del monolito ✂️ *← el paso que nadie debe olvidar*

<!--
ES: Un ejemplo concreto del bucle completo. Pregunta típica: "¿no debería conservar el
código viejo para verificar que el nuevo hace lo mismo?". Matiz importante: SÍ se
verifica contra el viejo, pero eso ocurre en el paso 3 (mientras conviven), no
conservándolo para siempre. La verificación tiene dos niveles: (a) la propia
redirección gradual 1%→100% observando métricas; (b) si el riesgo es alto y el
resultado es comparable, un Parallel Run en sombra que compara la salida del nuevo con
la del monolito (lo veremos en §5). Además, el rollback NO depende de tener el código
viejo: se hace re-apuntando la ruta en la fachada. El orden correcto es verificar →
estabilizar (p. ej. semanas al 100%) → borrar. El paso 4 es el FINAL: si no se borra,
caes en el "estrangulamiento eterno" — dos implementaciones, doble mantenimiento, un
sitio más donde corregir cada bug, y una migración que nunca termina. "Por si acaso"
no tiene fecha de fin. Ojo a la posible confusión con la slide anterior: "su propia BD"
NO significa que los datos estén resueltos. El servicio tiene almacén para lo que posee,
pero todavía depende de datos maestros del monolito (los lee por API) — exactamente la
limitación "los datos no se estrangulan solos". Estrangular el código mueve las
PETICIONES; separar/migrar los datos es un trabajo aparte (Sesión 3).

EN: A concrete example of the full loop. Typical question: "shouldn't I keep the old
code to verify the new one does the same?". Important nuance: YES you verify against the
old one, but that happens in step 3 (while they coexist), not by keeping it forever.
Verification has two levels: (a) the gradual redirect itself, 1%→100% watching metrics;
(b) if risk is high and the output is comparable, a shadow-mode Parallel Run comparing
the new output against the monolith's (covered in §5). Also, rollback does NOT depend on
keeping the old code: you re-point the route in the facade. The correct order is verify →
stabilize (e.g. weeks at 100%) → delete. Step 4 is the LAST one: if you don't delete, you
fall into "eternal strangling" — two implementations, double maintenance, one more place
to fix every bug, and a migration that never ends. "Just in case" has no end date.
Watch for the possible confusion with the previous slide: "its own DB" does NOT mean the
data is solved. The service has a store for what it owns, but it still depends on master
data from the monolith (reads it via API) — exactly the "data isn't strangled by itself"
limitation. Strangling the code moves the REQUESTS; splitting/migrating the data is a
separate job (Session 3).
-->

---

<!-- _class: lead -->

# Ejercicio 2.1
## El bucle del estrangulador

*(ver cuaderno de ejercicios)*

<!--
ES: Pausa para ejercicio. 2.1: el bucle completo del Strangler sobre SaludPlus (gestión
de citas, rutas /citas/*). Recorrer paso 0 (fachada enrutando el 100 % al monolito) →
construir el servicio (lee pacientes vía API interna + ACL) → redirección canary 1 %→100 %
→ borrar. Insistir en: verificar ANTES de borrar (canary + opcional Parallel Run), rollback
por re-enrutado en la fachada, y que "estrangular el código no estrangula los datos".
~15-20 min con puesta en común.

EN: Exercise break. 2.1: the full Strangler loop on SaludPlus (appointment management,
routes /citas/*). Walk through step 0 (facade routing 100 % to the monolith) → build the
service (reads patients via internal API + ACL) → canary redirect 1 %→100 % → delete.
Stress: verify BEFORE deleting (canary + optional Parallel Run), rollback by re-routing at
the facade, and that "strangling the code doesn't strangle the data". ~15-20 min with
group discussion.
-->

---

<!-- _class: lead -->

# 2. Capa Anticorrupción
## Anticorruption Layer (ACL)

---

## Concepto y relación con DDD

*(Término de Domain-Driven Design, Eric Evans.)*

**Problema:** cuando lo nuevo habla con el legacy, el modelo del legacy (nombres confusos, campos sobrecargados, reglas implícitas) **se filtra** y corrompe el modelo nuevo.

> Si el servicio nuevo acaba lleno de `flagEstado2` y `tipo_cliente_aux`, la reescritura no ha limpiado nada.

**Solución:** una capa de **traducción explícita** entre los dos modelos.

<!--
ES: La ACL responde a un problema muy real: el modelo legacy es "pegajoso". Si dejas
que sus conceptos crucen al servicio nuevo, en seis meses tu servicio "limpio" está
tan sucio como el monolito. La ACL es la frontera que lo impide. Es el patrón más
transversal de la sesión: aparece dentro de casi todos los demás.

EN: The ACL answers a very real problem: the legacy model is "sticky". If you let its
concepts cross into the new service, in six months your "clean" service is as dirty
as the monolith. The ACL is the boundary that prevents it. It's the most cross-cutting
pattern of the session: it shows up inside almost all the others.
-->

---

## La capa de traducción

```
┌────────────────────┐     ┌─────────────────────────┐     ┌──────────────┐
│  Servicio nuevo    │     │   ACL                   │     │   Monolito   │
│  (modelo limpio)   │◄──► │  · adaptadores          │◄──► │   (modelo    │
│                    │     │  · traductores de modelo│     │    legacy)   │
│  Customer          │     │  · fachadas             │     │  KUNDE_STAMM │
│  PolicyStatus      │     │  KUNDE_STAMM→Customer   │     │  FLAG_ST_2   │
└────────────────────┘     └─────────────────────────┘     └──────────────┘
```

- **Adaptador:** convierte protocolo/tecnología (SOAP→REST, tablas→objetos)
- **Traductor de modelo:** mapea conceptos (`FLAG_ST_2='X'` → `PolicyStatus.SUSPENDED`)
- **Fachada:** simplifica una interfaz legacy enrevesada

<!--
ES: Tres componentes con nombres de patrones clásicos. El adaptador es traducción
TÉCNICA (cómo hablo con el legacy); el traductor de modelo es traducción de
SIGNIFICADO (qué significan sus datos en mi mundo). La fachada agrupa operaciones
legacy en algo con sentido de negocio. No siempre hacen falta los tres.

EN: Three components named after classic patterns. The adapter is TECHNICAL
translation (how I talk to the legacy); the model translator is MEANING translation
(what its data means in my world). The facade groups legacy operations into something
with business sense. You don't always need all three.
-->

---

## Reglas de oro de la ACL

- La ACL **pertenece al sistema nuevo** (la escribe y mantiene su equipo, en sus términos)
- El modelo legacy **no cruza** la ACL: ningún tipo, nombre ni convención del legacy aparece más allá
- Es **código de usar y tirar a largo plazo**: cuando el legacy muera, la ACL se borra con él → puede ser pragmática, no perfecta

<!--
ES: Estas tres reglas son las que se evalúan en el ejercicio 2.2. La propiedad de la
ACL es lo más importante: si la pone el equipo legacy, el acoplamiento vuelve por la
puerta de atrás. Y recordar que la ACL es mortal: nace para morir con el legacy, así
que no hay que sobreingenierizarla.

EN: These three rules are what's assessed in exercise 2.2. ACL ownership is the most
important: if the legacy team builds it, coupling comes back through the back door.
And remember the ACL is mortal: it's born to die with the legacy, so don't
over-engineer it.
-->

---

## Beneficios y limitaciones

**Beneficios**
- Protege la integridad del modelo nuevo (la razón de la reescritura)
- Aísla el cambio: si el legacy cambia, solo se toca la ACL
- Documenta de forma ejecutable el mapeo viejo↔nuevo

**Limitaciones / costes**
- Código adicional que escribir y mantener (semánticas que no mapean 1:1, datos sucios)
- Un salto más (latencia, posible punto de fallo)
- Riesgo de **mini-monolito** si una sola ACL gigante lo centraliza todo → mejor **una ACL por contexto**

<!--
ES: La ACL no es gratis, pero el coste es pequeño comparado con la alternativa (un
servicio nuevo contaminado). La trampa a evitar: la ACL única y central que todos
usan. Eso recrea el acoplamiento que querías eliminar. Una ACL por frontera/servicio.

EN: The ACL isn't free, but the cost is small compared to the alternative (a polluted
new service). The trap to avoid: the single, central ACL everyone shares. That
recreates the coupling you wanted to remove. One ACL per boundary/service.
-->

---

## Ejemplo: la aseguradora

`policy-service` lee pólizas de un **mainframe**. *Activa* = cobertura **en vigor** (concepto de negocio, no un código).

**Lo que el dominio ve** (modelo nuevo): `Policy { id, status: ACTIVE | LAPSED | CANCELLED }`

**El mapeo vive dentro de la ACL:**

```
toPolicy(raw):                          # raw = registro del mainframe
  if   raw.MIGR_FLAG == 'S': st = migrStatus(raw.id)   # ← 2ª tabla
  elif raw.ST == '02':       st = ACTIVE
  elif raw.ST == '07':       st = LAPSED
  else:                      st = CANCELLED
  return Policy(raw.id, st)
```

El dominio llama `policy.isActive()` (≡ `status == ACTIVE`); nunca ve `ST`, `MIGR_FLAG` ni la 2ª tabla. Apagar el mainframe → reescribir `toPolicy()`, borrar la ACL; el dominio no cambia.

<!--
ES: Aquí se ve el mapeo concreto, que responde a las cuatro dudas habituales:
(1) "Activa" es un concepto de NEGOCIO (cobertura en vigor), no el código ST=02; el
mainframe lo CODIFICA como ST=02, la ACL lo recupera como concepto.
(2) El Policy se expone como tipo limpio del servicio nuevo (id + status enum); el
dominio solo conoce ese tipo.
(3) El mapeo es UNA función (toPolicy) que vive en la ACL, no en el dominio — es el
"traductor de modelo" de la slide de componentes.
(4) La 2ª tabla (caso MIGR_FLAG='S', pólizas migradas de un sistema anterior) se
resuelve con un lookup extra DENTRO de la ACL; el dominio nunca sabe que existió.
En términos de puertos/adaptadores: isActive()/el repositorio es el PUERTO, toPolicy()
es el ADAPTADOR legacy. Apagar el mainframe = reescribir el adaptador y borrar la ACL;
el dominio no cambia. Esa es la meta.

EN: Here the concrete mapping answers the four usual doubts:
(1) "Active" is a BUSINESS concept (coverage in force), not the code ST=02; the
mainframe ENCODES it as ST=02, the ACL recovers it as a concept.
(2) Policy is exposed as the new service's clean type (id + status enum); the domain
only knows that type.
(3) The mapping is ONE function (toPolicy) living in the ACL, not the domain — it's the
"model translator" from the components slide.
(4) The 2nd table (the MIGR_FLAG='S' case, policies migrated from an older system) is
resolved with an extra lookup INSIDE the ACL; the domain never knows it existed.
In ports/adapters terms: isActive()/the repository is the PORT, toPolicy() is the legacy
ADAPTER. Decommissioning the mainframe = rewrite the adapter and delete the ACL; the
domain doesn't change. That's the goal.
-->

---

<!-- _class: lead -->

# Ejercicio 2.2
## Diseñar una ACL

*(ver cuaderno de ejercicios)*

<!--
ES: Pausa para ejercicio. 2.2: traducir clientes del legacy (KD_TYP, STATUS_KZ…) al
modelo limpio Customer. Puntos clave: el caso 'X' arqueológico, el STATUS_KZ=9 que se
descompone en DOS conceptos (active + creditHold), y la pregunta trampa: ¿dejar que el
legacy traduzca en su vista? No — la traducción de modelo se queda en el lado nuevo.
~15-20 min con puesta en común.

EN: Exercise break. 2.2: translate legacy customers (KD_TYP, STATUS_KZ…) into the clean
Customer model. Key points: the archaeological 'X' case, STATUS_KZ=9 splitting into TWO
concepts (active + creditHold), and the trick question: let the legacy translate in its
view? No — model translation stays on the new side. ~15-20 min with group discussion.
-->

---

<!-- _class: lead -->

# 3. Reescritura Big-Bang

---

## Por qué es tentadora

- *"El código viejo no tiene arreglo; de cero será más rápido."*
- Psicológicamente atractiva: **campo verde**, tecnología nueva, sin deuda
- Evita la complejidad de la **convivencia** (proxies, sincronización, dos sistemas)

<!--
ES: Importante reconocer la tentación con honestidad: hay razones emocionales reales
para querer el big-bang. Empezar de cero es liberador. Por eso es peligroso: la
decisión se toma con el corazón, no con los datos. La siguiente slide es el
contrapeso.

EN: It's important to honestly acknowledge the temptation: there are real emotional
reasons to want the big-bang. Starting clean is liberating. That's exactly why it's
dangerous: the decision is made with the heart, not the data. The next slide is the
counterweight.
-->

---

## Por qué es arriesgada

- **Objetivo móvil:** el monolito sigue evolucionando durante la reescritura (el negocio no para)
- **El conocimiento está en el código:** miles de reglas no documentadas que se redescubren… en producción
- **Cero valor hasta el final:** todo se entrega de golpe en un corte de altísimo riesgo
- **Segundo sistema (Brooks):** la tentación de añadir "todo lo que faltaba" infla el alcance
- La evidencia es contundente: las reescrituras big-bang grandes **fracasan o se desbordan** con muchísima frecuencia

<!--
ES: Los cuatro riesgos clásicos. El "objetivo móvil" y "el conocimiento está en el
código" son los que matan proyectos. Un monolito de 10 años es una especificación
ejecutable de miles de reglas que nadie ha escrito en ningún sitio. Reescribir desde
cero es prometer reimplementarlas todas… sin tener la lista.

EN: The four classic risks. The "moving target" and "knowledge is in the code" are the
project-killers. A 10-year-old monolith is an executable spec of thousands of rules no
one has written down anywhere. Rewriting from scratch is promising to reimplement them
all… without having the list.
-->

---

## Cuándo *puede* justificarse

Casos acotados donde es defendible:

- Sistema **pequeño** y **bien especificado** (suite de tests exhaustiva, comportamiento trivial)
- Sistema **congelado de verdad** (sin cambios de negocio durante la reescritura)
- Imposibilidad real de convivencia (plataforma que desaparece) **y** tamaño manejable
- Reescrituras **parciales**: big-bang de *un módulo* dentro de una estrategia incremental

> El problema no es reescribir; es reescribirlo **todo** y entregarlo **de golpe**.

<!--
ES: El matiz que evita el dogmatismo: big-bang no está prohibido, está prohibido para
sistemas grandes, vivos y mal especificados. De hecho, cada extracción del Strangler
ES una reescritura big-bang en miniatura de esa pieza. El ejercicio 2.5 caso E es
justo el caso defendible. Lo veremos al final.

EN: The nuance that avoids dogmatism: big-bang isn't forbidden, it's forbidden for
large, live, poorly-specified systems. In fact, every Strangler extraction IS a
miniature big-bang rewrite of that piece. Exercise 2.5 case E is exactly the
defensible case. We'll see it at the end.
-->

---

<!-- _class: lead -->

# 4. Branch by Abstraction

---

## El problema que resuelve

Strangler intercepta **desde fuera**. ¿Y si lo que quieres reemplazar está **enterrado dentro**, llamado desde decenas de sitios? → no hay borde donde poner un proxy.

**Enfoque ingenuo — *rama de larga vida*:** ramificar en git, arrancar lo viejo y construir lo nuevo aparte durante semanas.
- el resto del equipo sigue en *trunk* (la línea principal, `main`) → la rama **diverge** → infierno de *merges*
- lo que está a medias **no se puede desplegar** → cero valor hasta el final

**Idea de BbA:** sustituir la rama de *git* por una **rama lógica en el código** — una abstracción con dos implementaciones. Ambas viven en `main`, compilan y se despliegan; **nada diverge**.

<!--
ES: El término "rama de larga vida" (long-lived branch) hay que explicarlo: es una rama
de control de versiones que vive semanas o meses separada de la principal. Tiene dos
males: (a) DIVERGE de main mientras los demás siguen commiteando → al final el merge es
enorme y doloroso; (b) el código a medias NO es desplegable, así que no entregas nada
hasta el final — un mini big-bang. "Trunk" = la línea principal (main/master);
"trunk-based development" = todos integran cambios pequeños en main a menudo,
manteniéndola siempre desplegable.
El truco de BbA: la "rama" se hace en el CÓDIGO (la abstracción con dos implementaciones
detrás), no en git. Las dos compilan y se despliegan en main; un flag elige cuál corre.
Así obtienes el aislamiento de una rama SIN el coste de una rama. El CÓMO concreto son
los cinco pasos de la siguiente slide. De ahí el nombre: "ramificar por abstracción".

EN: The term "long-lived branch" needs explaining: it's a version-control branch that
lives for weeks or months apart from the main line. Two evils: (a) it DIVERGES from main
while everyone else keeps committing → the eventual merge is huge and painful; (b) the
half-done code is NOT deployable, so you ship nothing until the end — a mini big-bang.
"Trunk" = the main line (main/master); "trunk-based development" = everyone integrates
small changes into main often, keeping it always releasable.
The BbA trick: the "branch" is made in the CODE (the abstraction with two implementations
behind it), not in git. Both compile and deploy on main; a flag picks which one runs. You
get a branch's isolation WITHOUT a branch's cost. The concrete HOW is the five steps on
the next slide. Hence the name: "branch by abstraction".
-->

---

## Los cinco pasos

1. **Crear una abstracción** (interfaz) sobre la funcionalidad a reemplazar
2. **Migrar los clientes** internos a la abstracción *(pasos 1–2 = pura refactorización)*
3. **Crear la implementación nueva** detrás de la misma abstracción (desplegada pero inactiva)
4. **Conmutar** con un *feature flag* (configurable, rollback al instante)
5. **Limpiar:** borrar la implementación vieja y, si ya no aporta, la abstracción/flag

<!--
ES: Los pasos 1-2 no cambian comportamiento: son refactorización segura. El paso 3
despliega código nuevo pero apagado. El paso 4 es el único que cambia comportamiento,
y el feature flag lo hace reversible en segundos SIN redesplegar. El paso 5 es el que
todos olvidan (igual que el "borrar" del Strangler): sin él, flags zombis para siempre.

EN: Steps 1-2 don't change behavior: safe refactoring. Step 3 deploys new code but
switched off. Step 4 is the only one that changes behavior, and the feature flag makes
it reversible in seconds WITHOUT redeploying. Step 5 is the one everyone forgets (like
the Strangler's "delete"): without it, zombie flags forever.
-->

---

## Visualmente

```
Paso 1-2:                       Paso 3-4:
  clientes                        clientes
     │                               │
     ▼                               ▼
 ┌──────────────┐               ┌──────────────┐
 │ INotificador │               │ INotificador │  ← flag decide
 └──────┬───────┘               └───┬──────┬───┘
        ▼                           ▼      ▼
 ┌──────────────┐            ┌────────┐ ┌─────────────────┐
 │ impl. legacy │            │ legacy │ │ nueva (llama al │
 └──────────────┘            └────────┘ │ servicio nuevo) │
                                        └─────────────────┘
```

Ambas implementaciones **conviven** detrás de la interfaz; el flag elige.

<!--
ES: La clave visual: en el paso 3-4 las dos implementaciones existen a la vez detrás
de la MISMA interfaz, y el flag decide cuál corre. Esa convivencia no es solo un
estado de transición: es una red de seguridad (puedes hacer fallback a la vieja).

EN: The visual key: in step 3-4 both implementations exist at once behind the SAME
interface, and the flag decides which runs. That coexistence isn't just a transitional
state: it's a safety net (you can fall back to the old one).
-->

---

## La abstracción no es solo una interfaz

Es **cualquier costura** (*seam*) donde insertar el conmutador:

- **Interfaz / clase abstracta** — lo típico en OO
- **Función de orden superior / Strategy / delegate** — la implementación como dato
- **Fachada, adaptador o inyección de dependencias** — costura a nivel de módulo / *wiring*
- **Proxy / decorador** — un envoltorio que delega a la vieja o la nueva
- **Publicar un evento** en vez de llamar directo — el contrato del evento es la abstracción
- **Costura de datos/infra** — repositorio, vista de BD, SPI/plugin, ruta del gateway

> Cuanto más **esparcida** esté la funcionalidad, más arriba conviene la costura (módulo/infra) que una interfaz fina.

<!--
ES: El objetivo de esta slide es romper la idea "abstracción = interfaz". La abstracción
de BbA es cualquier PUNTO DE INDIRECCIÓN (costura/seam, término de Michael Feathers)
donde puedas elegir entre dos implementaciones sin editar a los llamantes. Va por
niveles: lenguaje (interfaz, función/Strategy, subclase), módulo (fachada, adaptador,
inyección de dependencias, proxy/decorador), mensajería (evento en vez de llamada) e
infra/datos (repositorio, vista de BD, SPI/plugin, ruta del gateway). Regla práctica: si
la funcionalidad está muy esparcida, una interfaz fina cuesta mucho de introducir →
elige una costura más alta (un módulo-fachada, un evento, un proxy). Esto matiza la
limitación "encontrar una buena abstracción es difícil": a veces la costura correcta no
es una interfaz.

EN: The point of this slide is to break the "abstraction = interface" idea. BbA's
abstraction is any POINT OF INDIRECTION (a seam, Michael Feathers' term) where you can
choose between two implementations without editing the callers. It spans levels:
language (interface, function/Strategy, subclass), module (facade, adapter, dependency
injection, proxy/decorator), messaging (an event instead of a call) and infra/data
(repository, DB view, SPI/plugin, gateway route). Rule of thumb: if the functionality is
very scattered, a fine-grained interface is costly to introduce → pick a higher seam (a
facade module, an event, a proxy). This nuances the "finding a good abstraction is hard"
limitation: sometimes the right seam isn't an interface.
-->

---

## Ventajas y limitaciones

**Ventajas**
- Todo en trunk, despliegues continuos; sin ramas de larga vida
- Dos implementaciones conmutables al instante; cada paso pequeño y seguro
- Combina con Strangler: **Strangler para los bordes, BbA para las entrañas**

**Limitaciones**
- Exige **poder modificar** el código del monolito (no sirve para legacy intocable/de terceros)
- Una buena abstracción puede ser difícil si la funcionalidad está esparcida
- Coste doble de mantenimiento mientras conviven; requiere disciplina para el paso 5

<!--
ES: La frase a recordar: "Strangler para los bordes, BbA para las entrañas". Son el
mismo espíritu (reemplazo incremental y reversible) aplicado a niveles distintos. La
limitación dura: necesitas las fuentes y poder tocarlas. Para legacy de terceros solo
te quedan los bordes (Strangler + ACL).

EN: The phrase to remember: "Strangler for the edges, BbA for the guts". Same spirit
(incremental, reversible replacement) at different levels. The hard limitation: you
need the sources and the ability to touch them. For third-party legacy you're left
with only the edges (Strangler + ACL).
-->

---

<!-- _class: lead -->

# Ejercicio 2.3
## Branch by Abstraction, paso a paso

*(ver cuaderno de ejercicios)*

<!--
ES: Pausa para ejercicio. 2.3: el monolito llama SmtpMailer.send() desde 23 sitios;
moverlo a notification-service sin romper nada. Para cada paso: qué se despliega, si
cambia el comportamiento, cómo se hace rollback. Insistir en el papel del feature flag
(rollback por config) y en la política de fallos del envío nuevo. ~20 min.

EN: Exercise break. 2.3: the monolith calls SmtpMailer.send() from 23 places; move it
to notification-service without breaking anything. For each step: what's deployed,
whether behavior changes, how to roll back. Stress the feature flag's role (config
rollback) and the failure policy for the new sender. ~20 min.
-->

---

<!-- _class: lead -->

# 5. Parallel Run

---

## Concepto y variantes

A veces conmutar —aunque sea gradual— **da miedo**: cálculos financieros, tarificación, decisiones regulatorias.

**Parallel Run:** ejecutar **las dos implementaciones a la vez** con las mismas peticiones reales y **comparar resultados**.

- **Lo viejo es la verdad** (*shadow mode*): el cliente recibe la respuesta del legacy; la del nuevo se registra y compara. Las discrepancias no afectan a nadie.
- **Lo nuevo es la verdad:** el cliente recibe la respuesta nueva; el legacy corre detrás como verificación, antes de apagarse.

<!--
ES: Parallel Run no es un patrón de "movimiento" como Strangler o BbA: es un patrón de
VALIDACIÓN. Suele montarse encima de uno de los otros. El modo sombra es el habitual al
principio: el usuario nunca ve el resultado nuevo, así que cero riesgo, pero acumulas
evidencia con tráfico real.

EN: Parallel Run isn't a "movement" pattern like Strangler or BbA: it's a VALIDATION
pattern. It's usually layered on top of one of the others. Shadow mode is the usual
starting point: the user never sees the new result, so zero risk, but you accumulate
evidence with real traffic.
-->

---

## Qué se compara y cómo

- Resultados funcionales (¿la prima es la misma?), efectos esperados, y lo **no funcional** (¿latencia aceptable?)
- Hace falta un **comparador** (desde un job nocturno hasta comparación en línea)
- ⚠️ **Efectos secundarios:** en sombra el nuevo **no** debe enviar emails, cobrar tarjetas ni escribir fuera → stubs/sandbox
- ⚠️ **No determinismo:** timestamps, redondeos, IDs… las reglas deben tolerarlo o las falsas discrepancias ahogan a las reales

<!--
ES: Los dos "cuidados" son donde está el trabajo real de ingeniería. Desconectar los
efectos secundarios del sistema en sombra es DISEÑO, no es gratis: un parallel run que
cobra dos veces es el incidente, no su prevención. Y el no determinismo genera ruido:
si no lo controlas, te ahogas en discrepancias falsas y dejas de mirar el informe.

EN: The two "cautions" are where the real engineering work lives. Disconnecting the
shadow system's side effects is DESIGN, not free: a parallel run that charges twice is
the incident, not its prevention. And non-determinism generates noise: if you don't
control it, you drown in false discrepancies and stop reading the report.
-->

---

## Ventajas, limitaciones y encaje

**Ventajas:** la validación más fuerte posible — datos reales, sin riesgo; confianza cuantificable (*"99,98 % de coincidencia en 3M de peticiones"*); detecta lo que ningún test sintético encuentra.

**Limitaciones:** coste doble de ejecución; la infra de comparación es trabajo real; difícil con escrituras; solo verifica lo que el tráfico real ejercita.

**Encaje:** alto riesgo + **resultado comparable** (cálculos, scoring). Va *dentro* de un Strangler o un BbA, no aislado.

<!--
ES: El encaje es la idea a llevarse: Parallel Run no se usa solo. BbA pone las dos
implementaciones detrás de la abstracción; Parallel Run las ejecuta ambas y compara.
Y solo tiene sentido cuando hay una "respuesta correcta" comparable. Para algo cuyo
resultado QUIERE ser distinto (recomendaciones), no aplica — ahí va un A/B.

EN: The fit is the takeaway: Parallel Run isn't used alone. BbA puts both implementations
behind the abstraction; Parallel Run runs both and compares. And it only makes sense when
there's a comparable "correct answer". For something whose result is MEANT to differ
(recommendations), it doesn't apply — there you use an A/B test.
-->

---

<!-- _class: lead -->

# Ejercicio 2.4
## Parallel Run: diseñar la comparación

*(ver cuaderno de ejercicios)*

<!--
ES: Pausa para ejercicio. 2.4: sustituir el cálculo de comisiones (COBOL) que paga por
transferencia a fin de mes; "cero errores en nóminas". Discutir: variante (sombra, legacy
como verdad) y duración (varios cierres mensuales); el efecto secundario crítico (¡no
pagar de verdad!); 3 discrepancias falsas (redondeos, snapshot, formato); y qué hacer
cuando el equivocado es el COBOL (escalar a negocio, no replicar el bug). ~20-25 min.

EN: Exercise break. 2.4: replace the commission calculation (COBOL) that pays by transfer
at month-end; "zero payroll errors". Discuss: variant (shadow, legacy as truth) and
duration (several monthly closes); the critical side effect (don't actually pay!); 3 false
discrepancies (rounding, snapshot, format); and what to do when COBOL is the wrong one
(escalate to business, don't replicate the bug). ~20-25 min.
-->

---

<!-- _class: lead -->

# 6. Bubble Context

---

## Concepto

*(Patrón DDD, Eric Evans.)* Cuando quieres modelar **bien** un dominio nuevo pero todo el entorno es legacy: creas una **burbuja** — un contexto nuevo, pequeño y limpio, **protegido por una ACL**.

```
┌────────────────────────── legacy ─────────────────────────┐
│            ┌─ ACL ─┐   ┌──────────────────────┐           │
│   datos ◄──┤ trad. ├──►│   BURBUJA            │           │
│   legacy   └───────┘   │  modelo limpio,      │           │
│                        │  lenguaje ubicuo,    │           │
│                        │  reglas nuevas       │           │
│                        └──────────────────────┘           │
└───────────────────────────────────────────────────────────┘
```

La burbuja se centra en **el modelo**, no en la infraestructura: puede vivir *dentro* del despliegue del monolito.

<!--
ES: El matiz que distingue Bubble de "extraer un servicio": aquí lo primero es la
FRONTERA DE MODELO, no la de proceso. Puedes tener una burbuja con su modelo limpio
viviendo todavía dentro del mismo despliegue del monolito. Lo que la protege es la ACL.
El motor de este patrón no es escalar ni desplegar aparte: es remodelar el dominio.

EN: The nuance that distinguishes Bubble from "extract a service": here the MODEL
BOUNDARY comes first, not the process boundary. You can have a bubble with its clean
model still living inside the monolith's deployment. What protects it is the ACL. This
pattern's driver isn't scaling or separate deployment: it's reshaping the domain.
-->

---

## Evolución típica

1. **Burbuja simple:** sin persistencia propia; la ACL traduce desde los datos del legacy en cada acceso
2. **Burbuja con BD propia** (*autonomous bubble*): la ACL pasa a ser una capa de **sincronización** (a menudo asíncrona)
3. La burbuja crece y se le da despliegue propio → **microservicio nacido sano**; o revienta la frontera y absorbe al legacy

<!--
ES: La burbuja es un punto de partida que evoluciona. Empieza barata (sin BD propia) y
va ganando autonomía. El destino feliz: se convierte en un microservicio que nació
limpio, con buen modelo desde el día uno, en vez de un trozo de monolito mal cortado.
El riesgo: que se quede estancada en el paso 1 como isla rara.

EN: The bubble is a starting point that evolves. It starts cheap (no own DB) and gains
autonomy over time. The happy ending: it becomes a microservice that was born clean,
with a good model from day one, instead of a badly-cut chunk of monolith. The risk:
getting stuck at step 1 as a weird island.
-->

---

## Ventajas y limitaciones

**Ventajas**
- Aplicar **DDD de verdad** desde el día uno en la parte nueva
- Inversión inicial pequeña (no exige plataforma de microservicios)
- Ideal cuando el motor es un **dominio nuevo** o que necesita remodelarse

**Limitaciones**
- La ACL de **sincronización** puede volverse compleja (sobre todo con escrituras en ambos lados)
- Si la burbuja no crece, queda como **isla rara** dentro del legacy
- Requiere **vigilancia de la frontera** ("saltarse la ACL solo esta vez" es una presión real)

<!--
ES: La limitación técnica dura es la sincronización bidireccional de datos: si ambos
lados escriben los mismos datos, la ACL de sync se complica mucho (es un anticipo del
problema de datos de la Sesión 3). La limitación humana: la disciplina de no perforar
la frontera. Basta una excepción "solo por hoy" para empezar a recontaminar.

EN: The hard technical limitation is bidirectional data sync: if both sides write the
same data, the sync ACL gets very complex (a preview of Session 3's data problem). The
human limitation: the discipline not to pierce the boundary. One "just for today"
exception is enough to start re-contaminating.
-->

---

<!-- _class: lead -->

# 7. Comparativa de patrones

---

## Tabla de decisión

| Patrón | Pregunta que responde | Mejor cuando… | Riesgo principal |
|---|---|---|---|
| **Strangler Fig** | ¿Reemplazo funcionalidad del borde? | Hay un punto de entrada interceptable | Quedarse a medias; ignorar datos |
| **ACL** | ¿Evito que el legacy contamine lo nuevo? | Siempre que lo nuevo hable con lo viejo | Mini-monolito; mantenimiento |
| **Big-Bang** | ¿Lo reescribo todo de golpe? | Sistema pequeño, congelado, especificado | Objetivo móvil; cero valor |
| **Branch by Abstraction** | ¿Reemplazo algo enterrado? | Puedes tocar el código; muchos llamantes | No ejecutar la limpieza |
| **Parallel Run** | ¿Gano confianza en lo nuevo? | Alto riesgo + salida comparable | Coste doble; efectos 2arios |
| **Bubble Context** | ¿Modelo limpio dentro del legacy? | El motor es remodelar el dominio | Sincronización; burbuja estancada |

<!--
ES: Esta tabla es la chuleta de la sesión. La columna clave para decidir es la segunda
("¿qué pregunta responde?"): cada patrón responde a una pregunta distinta, así que no
compiten — se eligen según la situación. Sugerir hacerle foto. Es la base del ejercicio
2.5.

EN: This table is the session's cheat sheet. The key column for deciding is the second
one ("what question does it answer?"): each pattern answers a different question, so
they don't compete — you pick by situation. Suggest taking a photo. It's the basis for
exercise 2.5.
-->

---

## No son excluyentes: se combinan

Una migración real los **apila**:

- **Strangler Fig** como estrategia global de redirección desde el borde…
- …usando **Branch by Abstraction** para las piezas internas sin borde…
- …con una **ACL** en cada frontera entre lo nuevo y lo viejo…
- …validando lo crítico con **Parallel Run** antes de conmutar…
- …y arrancando los dominios a remodelar como **Bubble Contexts**.

<!--
ES: El mensaje central de la sesión: no se elige UN patrón, se compone una estrategia
con varios operando a niveles distintos. El error de novato es preguntarse "¿uso
Strangler O BbA?" cuando la respuesta casi siempre es "ambos, en sitios distintos, con
ACLs por todas las fronteras". El ejercicio 2.6 practica exactamente esto.

EN: The session's central message: you don't pick ONE pattern, you compose a strategy
with several operating at different levels. The rookie mistake is asking "Strangler OR
BbA?" when the answer is almost always "both, in different places, with ACLs at every
boundary". Exercise 2.6 practices exactly this.
-->

---

## Criterios transversales de elección

1. **¿Dónde está el corte?** Borde visible → Strangler · Entrañas → Branch by Abstraction
2. **¿Cuál es el motor?** Escalado/entrega → Strangler/BbA · Remodelar dominio → Bubble (+ACL)
3. **¿Cuánto miedo da equivocarse?** Mucho y comparable → añade Parallel Run
4. **¿Puedo tocar el legacy?** No → solo bordes: Strangler + ACL
5. **Siempre:** ACL en las fronteras y **plan explícito de borrado** del legacy

<!--
ES: Estas cinco preguntas son un árbol de decisión práctico. Las preguntas 1 y 4 son
las más discriminantes (dónde cortas y si puedes tocar el código). La 5 es el recordatorio
permanente: ACL en cada frontera y, como en la Sesión 1, el paso de BORRAR el legacy que
todo el mundo olvida.

EN: These five questions are a practical decision tree. Questions 1 and 4 are the most
discriminating (where you cut and whether you can touch the code). Number 5 is the
permanent reminder: ACL at every boundary and, as in Session 1, the DELETE-the-legacy
step everyone forgets.
-->

---

<!-- _class: lead -->

# Ejercicio 2.5 + 2.6
## Elegir patrón para cada situación
## Combinar patrones: plan integral

*(ver cuaderno de ejercicios)*

<!--
ES: Pausa para ejercicios finales. 2.5: cinco escenarios (A-E), elegir patrón y
justificar — incluye el caso E donde Big-Bang SÍ es correcto. 2.6: MediaStream — plan
integral combinando patrones para dos extracciones (recomendaciones con BbA+Bubble;
facturación con Strangler+Parallel Run), señalando ACLs y validación. ~30 min con puesta
en común. Son la síntesis de toda la sesión.

EN: Final exercise break. 2.5: five scenarios (A-E), pick a pattern and justify —
including case E where Big-Bang IS correct. 2.6: MediaStream — a full plan combining
patterns for two extractions (recommendations with BbA+Bubble; billing with Strangler+
Parallel Run), marking ACLs and validation. ~30 min with group discussion. They're the
synthesis of the whole session.
-->

---

## Resumen de la sesión

- **Strangler Fig:** interceptar en el borde y redirigir gradualmente; reversible y visible, pero no resuelve los datos
- **ACL:** traducción que impide que el legacy contamine lo nuevo; pertenece al lado nuevo; se borra con el legacy
- **Big-Bang:** mala idea para sistemas grandes y vivos; justificable solo en piezas pequeñas, congeladas y especificadas
- **Branch by Abstraction:** el "strangler de interior" — abstracción, dos implementaciones, flag, limpieza; en trunk
- **Parallel Run:** ambos sistemas con tráfico real y comparación; la validación más fuerte para lo crítico y comparable
- **Bubble Context:** un contexto limpio protegido por ACL para remodelar el dominio dentro del legacy
- Los patrones **se combinan**; y siempre: ACL en toda frontera y **borrado** del legacy

<!--
ES: El destilado de la sesión. Si solo se llevan una idea: los patrones no compiten, se
combinan, y cada uno responde a una pregunta distinta. La siguiente sesión ataca el
problema que todos estos patrones dejan abierto: DÓNDE cortar (DDD) y cómo partir los
DATOS, que es lo más difícil de todo.

EN: The session's distillate. If they take away one idea: patterns don't compete, they
combine, and each answers a different question. The next session tackles the problem all
these patterns leave open: WHERE to cut (DDD) and how to split the DATA, which is the
hardest part of all.
-->

---

<!-- _class: lead -->

# ¿Preguntas?

**Próxima sesión:** Domain-Driven Design y descomposición de datos
*Bounded contexts · BD por servicio · Saga · Outbox*

<!--
ES: Abrir turno de preguntas. Recordar dónde está el material y el cuaderno de ejercicios.
Hacer de puente: hoy hemos visto CÓMO mover funcionalidad con seguridad; la Sesión 3
responde DÓNDE cortar y resuelve el cabo suelto que ha aparecido en casi todos los
patrones de hoy: los datos.

EN: Open the floor for questions. Remind them where the material and exercise workbook
are. Bridge forward: today we saw HOW to move functionality safely; Session 3 answers
WHERE to cut and resolves the loose end that showed up in almost every pattern today:
the data.
-->
