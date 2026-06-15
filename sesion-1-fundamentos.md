# Sesión 1 — Fundamentos y la justificación de migrar

**Duración:** 3 h · **Objetivo:** establecer un vocabulario común y un encuadre claro y honesto de cuándo (y cuándo no) migrar de un monolito a microservicios.

---

## 1. Introducción: monolitos vs. microservicios

### 1.1 ¿Qué es un monolito?

El término **monolito** se usa para dos ideas distintas que a menudo se confunden:

- **Monolito como unidad de despliegue:** toda la aplicación se compila, empaqueta y despliega como un solo artefacto (un WAR, un binario, una imagen).
- **Monolito como unidad de organización del código:** todo el código vive en un mismo proyecto, con (o sin) estructura interna.

Ambas cosas suelen ir juntas, pero no son lo mismo: puede haber un código bien modularizado que se despliega como un único artefacto (lo veremos como *monolito modular*), y un repositorio único del que salen varios artefactos. En esta formación, cuando decimos "monolito" sin más, nos referimos a la **unidad de despliegue única** — porque es el despliegue, no la organización del código, lo que los microservicios cambian.

Un monolito **no es intrínsecamente malo**. Tiene ventajas reales:

| Ventaja | Por qué |
|---|---|
| Simplicidad operativa | Un solo proceso que desplegar, monitorizar y escalar |
| Transacciones ACID | Una sola base de datos: consistencia "gratis" |
| Refactorización fácil | El IDE ve todo el código; renombrar es seguro |
| Latencia mínima | Las llamadas internas son invocaciones de método, no red |
| Depuración sencilla | Un stack trace cuenta toda la historia |

Sus problemas aparecen con la **escala** (del sistema y de la organización):

- **Acoplamiento creciente:** sin disciplina, todo acaba dependiendo de todo, hasta degenerar en una *big ball of mud* — la "gran bola de barro" (Foote & Yoder, 1997): un sistema sin arquitectura discernible, código espagueti a escala de sistema.
- **Despliegues acoplados:** un cambio pequeño exige redesplegar todo; el riesgo de cada despliegue crece y la frecuencia baja.
- **Escalado grueso:** si solo un módulo necesita más recursos, hay que escalar la aplicación entera.
- **Cuellos de botella organizativos:** muchos equipos tocando el mismo código → conflictos, colas de integración, "trenes de release".
- **Atadura tecnológica:** una sola pila tecnológica para todos los problemas.

### 1.2 ¿Qué es un microservicio?

Un **microservicio** es un servicio desplegable de forma independiente, modelado alrededor de un **dominio de negocio**, que se comunica con otros servicios a través de la red y **oculta sus datos** tras una interfaz.

Las propiedades clave (más importantes que el tamaño):

1. **Despliegue independiente** — puedo cambiar y desplegar el servicio A sin coordinarme con el servicio B. Esta es *la* propiedad definitoria.
2. **Modelado en torno al negocio** — los límites siguen el dominio (pedidos, facturación), no las capas técnicas (UI, lógica, datos).
3. **Propiedad de sus datos** — ningún otro servicio accede directamente a su base de datos.
4. **Autonomía del equipo** — un equipo puede ser dueño del ciclo de vida completo del servicio.

> **Idea fuerza:** "micro" no significa "pequeño en líneas de código", significa "pequeño en superficie de acoplamiento". La pregunta correcta no es "¿cuántas líneas?" sino "¿puedo desplegarlo sin pedir permiso?".

### 1.3 Lo que se gana y lo que se paga

Los microservicios **intercambian complejidad local por complejidad distribuida**:

| Se gana | Se paga |
|---|---|
| Despliegue independiente, releases frecuentes | Sistema distribuido: latencia, fallos parciales |
| Escalado selectivo por servicio | Operación mucho más compleja (orquestación, observabilidad) |
| Autonomía de equipos | Necesidad de contratos y versionado entre servicios |
| Aislamiento de fallos (potencial) | Consistencia de datos sin transacciones globales |
| Libertad tecnológica por servicio | Coste de plataforma (CI/CD, descubrimiento, gateway…) |

Las famosas **falacias de la computación distribuida** (Peter Deutsch, Sun Microsystems) resumen lo que deja de ser cierto al pasar por la red: la red *no* es fiable, la latencia *no* es cero, el ancho de banda *no* es infinito, la red *no* es segura, la topología *cambia*…

---

## 2. Más allá de la dicotomía: el espectro arquitectónico

No hay que elegir entre "monolito caótico" y "200 microservicios". Existe un **espectro**:

```
Big ball of mud → Monolito por capas → Monolito modular → SOA / servicios gruesos → Microservicios
```

### 2.1 Los peldaños del espectro

**Big ball of mud** — Un monolito *sin* arquitectura discernible: todo depende de todo, sin límites internos (§1.1). Importa la distinción de escala frente al código espagueti: el espagueti es desorden *dentro* de un módulo (feo, pero localizado y arreglable); la bola de barro es la ausencia de estructura *entre* módulos. Y es esta última la que estorba a una migración: sin límites internos, no hay costuras por donde cortar.

**Monolito por capas** — Un monolito con orden y disciplina, pero cuya estructura es *técnica*: capas horizontales (presentación, lógica de negocio, acceso a datos). Es el clásico monolito empresarial "bien hecho". Su límite: un cambio de negocio típico atraviesa todas las capas, y las capas no ofrecen fronteras por donde extraer servicios — el orden existe, pero no sigue al negocio.

**Monolito modular** — El corte cambia de dirección: *vertical*, por dominios de negocio (pedidos, facturación…); cada módulo contiene sus capas dentro y expone una interfaz pequeña. Esos cortes verticales sí son costuras extraíbles — por eso es el peldaño previo natural a los microservicios. Lo desarrollamos en §2.2.

**SOA / servicios gruesos** — La arquitectura orientada a servicios clásica de los 2000: el sistema ya está repartido en varios servicios desplegables que se comunican por red, pero en servicios **gruesos** (*coarse-grained*): pocos y grandes, a menudo del tamaño de una aplicación entera ("el servicio de clientes" de toda la empresa). Sus señas: integración centralizada en un **ESB** (un bus intermediario inteligente donde acababa viviendo mucha lógica), estándares pesados (SOAP, WS-\*), un "modelo de datos canónico" corporativo, gobernanza y releases coordinadas centralmente, y datos a menudo todavía compartidos. Su lección histórica: distribuir *sin* descentralizar reproduce los cuellos de botella del monolito con la complejidad de la red añadida.

**Microservicios** — El grano se afina y las decisiones de la SOA clásica se invierten: muchos servicios pequeños modelados por dominio de negocio, *smart endpoints, dumb pipes* (la lógica vive en los servicios; la red solo transporta), un modelo propio por contexto en lugar del canónico corporativo, base de datos por servicio, y autonomía y despliegue independiente por equipo (§1.2). En este sentido, los microservicios son "SOA de grano fino y descentralizada".

### 2.2 El monolito modular

Un **monolito modular** mantiene una sola unidad de despliegue, pero el código está organizado en **módulos con límites explícitos**:

- Cada módulo expone una **interfaz pública pequeña** y oculta su interior.
- Los módulos se comunican solo a través de esas interfaces (idealmente verificado por herramientas).
- Cada módulo puede tener **su propio esquema de datos** lógico (aunque compartan instancia de BD).

El monolito modular es:

- Un **destino válido** en sí mismo: muchas organizaciones obtienen el 80 % del beneficio (límites claros, equipos con propiedad) sin pagar el coste de la distribución.
- El **mejor punto de partida** para una futura migración: si los límites de módulo son buenos, extraer un módulo a servicio es mucho más barato.

> **Regla práctica:** si no eres capaz de construir un monolito modular bien estructurado, los microservicios no van a arreglarlo — van a distribuir el desorden y añadirle red.

### 2.3 ¿Dónde situarse en el espectro?

No hay un punto "correcto" del espectro: hay un punto adecuado para cada organización en cada momento. Los factores que más pesan en la decisión:

- **Tamaño y número de equipos** — el factor más determinante. Los microservicios resuelven sobre todo un problema *organizativo*: muchos equipos pisándose en el mismo código y en la misma release. Con uno o dos equipos, ese problema no existe — y la autonomía de despliegue que pagarías tan cara apenas tendría beneficiarios.
- **Necesidades de escalado diferencial** — si una parte del sistema necesita 10 veces más recursos que el resto (la búsqueda, la generación de informes), separarla permite escalarla sola. Si todo el sistema escala de forma homogénea, el monolito replicado escala perfectamente bien.
- **Madurez operativa** — operar N servicios exige CI/CD, observabilidad y automatización de infraestructura. Cada paso a la derecha del espectro multiplica las piezas que hay que desplegar, monitorizar y depurar; sin esa base, el día a día se vuelve inviable. La madurez operativa marca el límite derecho *alcanzable*, no el deseable.
- **Velocidad de cambio requerida, y dónde se concentra** — la independencia de despliegue rinde donde el código cambia a menudo. Si el cambio se concentra en dos o tres zonas, quizá solo esas zonas merezcan ser servicios y el resto pueda quedarse en el monolito.

Tres consecuencias prácticas de mirar el espectro así:

1. **La posición no es permanente.** Se puede (y se suele) avanzar hacia la derecha a medida que crecen el equipo y la madurez — eso es exactamente lo que llamamos migración. También es legítimo retroceder: fusionar servicios que resultaron demasiado finos es una corrección, no un fracaso.
2. **La posición no tiene que ser uniforme.** Un mismo sistema puede tener su núcleo volátil como microservicios y sus zonas estables en un monolito modular. Los híbridos deliberados no son estados de transición vergonzantes: son a menudo el destino final óptimo.
3. **El movimiento tiene un orden natural.** Saltar de la bola de barro directamente a microservicios significa cortar sin costuras; pasar antes por el monolito modular crea los límites que luego se extraen. De ahí la regla práctica de §2.2: quien no sabe construir un monolito modular no está listo para microservicios.

---

## 3. Justificar la división

### 3.1 Empezar por el porqué: objetivos y criterios de éxito

Migrar a microservicios es un **medio**, nunca un fin. Antes de empezar hay que poder completar esta frase:

> "Migramos porque queremos conseguir **X**, lo mediremos con **Y**, y sabremos que hemos terminado/avanzado cuando **Z**."

Motivaciones legítimas habituales y su métrica natural:

| Objetivo | Métrica de éxito posible |
|---|---|
| Acelerar la entrega | Frecuencia de despliegue, lead time de cambios |
| Escalar la organización | Nº de equipos que entregan de forma autónoma |
| Escalar técnicamente una parte | Coste de infraestructura, latencia bajo carga |
| Aislar fallos | Radio de impacto de incidentes, MTTR |
| Modernizar tecnología | % de tráfico servido por la nueva pila |

> **Dos términos de la tabla:**
> - **Lead time (de cambios):** el tiempo que transcurre desde que un cambio se confirma en el código (*commit*) hasta que está funcionando en producción. Mide cuánto tarda el trabajo terminado en llegar al usuario; un lead time alto delata cuellos de botella en integración, pruebas o despliegue. (Es una de las cuatro métricas DORA, junto con la frecuencia de despliegue, la tasa de fallos en cambios y el MTTR.)
> - **MTTR (*Mean Time To Recovery*):** el tiempo medio que se tarda en restablecer el servicio tras un incidente en producción. No mide cuántos fallos hay, sino con qué rapidez se recupera el sistema de ellos.

Si el objetivo es difuso ("modernizar", "es lo que hace todo el mundo"), la migración no tendrá criterio de parada ni forma de priorizar.

### 3.2 Cuándo *no* merece la pena migrar

Señales de que los microservicios son probablemente una mala idea ahora:

- **Equipo pequeño** (p. ej. < 10–15 personas): el coste operativo se come la ganancia de autonomía.
- **Dominio aún poco entendido o producto en búsqueda de mercado:** los límites cambiarán; mover límites entre servicios es carísimo, dentro de un monolito es barato.
- **El problema real es otro:** falta de tests, despliegues manuales, deuda de calidad. La distribución no arregla nada de eso; lo amplifica.
- **Madurez operativa insuficiente:** sin CI/CD, sin observabilidad, sin automatización de infraestructura, operar N servicios es inviable.
- **Producto que va a ser reemplazado o tiene vida corta.**

Alternativas que conviene considerar **antes** de dividir:

- **Re-platforming:** mover el monolito tal cual a mejor infraestructura (contenedores, nube, autoescalado). A menudo resuelve los problemas de escalado.
- **Modularización interna:** convertirlo en monolito modular. Resuelve los problemas de acoplamiento y de equipos.
- **Extraer solo 1–2 servicios** donde haya un dolor concreto (p. ej. el módulo que necesita escalar), y parar.

### 3.3 El anti-patrón del monolito distribuido

El **monolito distribuido** es el peor resultado posible: muchos servicios que **deben desplegarse juntos** porque siguen acoplados.

Síntomas:

- Un cambio funcional típico toca 3–4 servicios a la vez.
- Los despliegues se coordinan en "trenes": "el viernes desplegamos todos".
- Servicios que comparten base de datos.
- Cadenas largas de llamadas síncronas: la caída de uno tumba a todos.
- Contratos que cambian sin versionado: hay que actualizar consumidores y productor al unísono.

Causas raíz típicas: dividir por capas técnicas en lugar de por dominio, dividir antes de entender el dominio, no romper la base de datos compartida.

> **Test rápido:** si para entregar una funcionalidad media necesitas coordinar despliegues de varios servicios, no tienes microservicios; tienes un monolito con latencia de red.

---

## 4. Fundamentos de estrategia de migración

### 4.1 Incremental vs. big-bang

- **Big-bang:** reescribir el sistema completo y cambiar de golpe. Casi siempre fracasa en sistemas no triviales: el monolito sigue evolucionando mientras reescribes (objetivo móvil), no entregas valor en meses/años, y el riesgo se concentra en un único evento de corte. (Lo analizaremos como patrón en la Sesión 2.)
- **Incremental:** extraer funcionalidad pieza a pieza, manteniendo el sistema funcionando en todo momento. El monolito y los servicios **conviven durante años**, y eso no es un fallo del plan: *es* el plan.

Principios del enfoque incremental:

1. **Pasos pequeños y reversibles.** Cada paso debe poder deshacerse (rollback) sin drama.
2. **Entregar valor por el camino.** Cada extracción debe justificar su coste por sí misma.
3. **El sistema funciona siempre.** Nunca hay una fase "ahora estamos rotos durante 3 meses".
4. **Aprender y ajustar.** Las primeras extracciones enseñan; el plan se revisa.

### 4.2 Gestión del riesgo

- **Empieza por algo que enseñe mucho y arriesgue poco:** un módulo razonablemente desacoplado, con valor visible, pero no el corazón crítico del negocio.
- **Separa el despliegue de la activación:** poder desplegar el servicio nuevo sin enviarle tráfico real (feature flags, enrutado gradual) reduce el riesgo de cada paso.
- **Mide antes y después:** sin línea base (rendimiento, errores, lead time) no podrás demostrar mejora ni detectar regresión.
- **Plan de retirada del legacy:** la migración no termina cuando el servicio nuevo funciona, sino cuando el código antiguo **se borra**. El "modo híbrido eterno" es un coste permanente.

### 4.3 Cómo secuenciar una migración

Criterios para decidir **qué extraer primero**:

| Criterio | Favorece extraer pronto |
|---|---|
| Acoplamiento | Módulos con pocas dependencias entrantes/salientes |
| Valor | Donde la autonomía/escalado aporta beneficio medible |
| Volatilidad | Código que cambia mucho (la independencia de despliegue rinde más) |
| Riesgo | Funcionalidad no crítica para los primeros pasos |
| Datos | Módulos con datos poco compartidos con el resto |

Herramienta útil: una matriz **valor de extraer × facilidad de extraer**:

```
                            Facilidad de extraer
                        baja                  alta
                 ┌──────────────────────┬──────────────────────┐
            alto │ (2) INVERTIR         │ (1) EMPEZAR AQUÍ     │
                 │ merece el esfuerzo:  │ victorias rápidas    │
   Valor de      │ planificar, abordar  │ que entregan valor   │
   extraer       │ cuando haya oficio   │ y enseñan            │
                 ├──────────────────────┼──────────────────────┤
            bajo │ (4) ¿EXTRAER JAMÁS?  │ (3) RELLENO          │
                 │ candidato a quedarse │ barato pero aporta   │
                 │ en el monolito       │ poco: no confundir   │
                 │ (deliberadamente)    │ con progreso         │
                 └──────────────────────┴──────────────────────┘
```

- **(1) Alto valor / alta facilidad — empezar aquí:** cada extracción se justifica sola y, de paso, el equipo aprende y la plataforma madura.
- **(2) Alto valor / baja facilidad — invertir, pero después:** son las piezas que justifican la migración a largo plazo (a menudo el corazón del negocio); se abordan cuando las extracciones del cuadrante (1) ya han dado oficio, herramientas y APIs estables alrededor.
- **(3) Bajo valor / alta facilidad — relleno:** extraer por extraer engorda el número de servicios sin mover las métricas; útil como mucho para practicar, pero no confundir actividad con progreso.
- **(4) Bajo valor / baja facilidad — cuestionar si jamás:** el candidato natural a quedarse en el monolito (modularizado) de forma deliberada y documentada. Que algo *pueda* extraerse no significa que *deba*.

### 4.4 Buenas prácticas generales

- Trata la migración como **producto**, no como proyecto big-bang: backlog, prioridades, métricas, revisión continua.
- **No congeles el monolito** salvo en las zonas en extracción activa; el negocio no puede parar.
- Invierte pronto en la **plataforma** (CI/CD, observabilidad, plantillas de servicio): el segundo y tercer servicio deben ser mucho más baratos que el primero.
- Comunica el **estado de la convivencia**: qué vive ya en servicios, qué sigue en el monolito, qué está a medias.

---

## 5. El panorama de patrones (mapa de la formación)

Los patrones que veremos se agrupan así:

```
┌─ Patrones de MIGRACIÓN (Sesión 2) ────────────────────────────┐
│  ¿Cómo muevo funcionalidad del monolito a servicios?          │
│  · Strangler Fig        · Branch by Abstraction               │
│  · Anticorruption Layer · Parallel Run                        │
│  · Bubble Context       · (Big-Bang, como contraste)          │
└───────────────────────────────────────────────────────────────┘
┌─ Patrones de DESCOMPOSICIÓN (Sesión 3) ───────────────────────┐
│  ¿Dónde corto? ¿Qué hago con los datos?                       │
│  · DDD: bounded contexts, lenguaje ubicuo, context maps       │
│  · BD por servicio · Saga · Outbox · Consistencia eventual    │
└───────────────────────────────────────────────────────────────┘
┌─ Patrones de OPERACIÓN del resultado (Sesión 4) ──────────────┐
│  ¿Cómo hago robusto el sistema distribuido resultante?        │
│  · Sync/async, eventos · API Gateway · Descubrimiento         │
│  · Circuit breaker, retries, timeouts · Observabilidad        │
│  · Conway, Team Topologies                                    │
└───────────────────────────────────────────────────────────────┘
```

Relación entre grupos: los patrones de **descomposición** dicen *dónde* cortar; los de **migración** dicen *cómo* ejecutar el corte de forma segura; los de **operación** hacen vivible el resultado. Una migración real los combina todos.

---

## Ejercicios

### Ejercicio 1.1 — ¿Monolito culpable o inocente?

La empresa **LibroExpress** (e-commerce de libros, 8 desarrolladores en 1 equipo) tiene un monolito en Java. Se quejan de:

1. Los despliegues son manuales y ocurren una vez al mes, en sábado.
2. No hay tests automatizados; cada release rompe algo.
3. En campaña de Navidad, la web entera va lenta porque el módulo de búsqueda consume toda la CPU.

Ante esta situación, el CTO propone: *"Migremos a microservicios, así desplegaremos a diario y escalará mejor."*

**Pregunta:** ¿Cuáles de los problemas (1–3) resolverían realmente los microservicios? ¿Qué propondrías tú en su lugar? Justifica con los conceptos de la sesión.

<details>
<summary>💡 Solución</summary>

**Análisis problema a problema:**

1. **Despliegues manuales y mensuales** — No es un problema del monolito, es un problema de **madurez operativa** (falta de CI/CD). Los microservicios lo *empeorarían*: pasarían a tener N despliegues manuales en vez de 1. Se puede desplegar un monolito varias veces al día (muchas empresas lo hacen). Solución: pipeline de CI/CD.

2. **Sin tests, cada release rompe algo** — Problema de **calidad/deuda técnica**. Los microservicios lo amplifican: a los bugs locales se suman los fallos de integración entre servicios, mucho más difíciles de depurar. Solución: invertir en tests automatizados primero. Además, sin tests, cualquier migración (que es una refactorización gigante) sería extremadamente arriesgada.

3. **La búsqueda consume toda la CPU en picos** — Este es el único problema donde la división aporta algo: **escalado selectivo**. Pero hay alternativas más baratas que una migración completa: re-platforming (autoescalado horizontal del monolito entero, que suele ser suficiente), o extraer **solo el servicio de búsqueda** (a menudo además con tecnología especializada tipo motor de búsqueda) y dejar el resto como está.

**Propuesta razonable:**
- Con **8 personas en 1 equipo**, no existe el problema organizativo que los microservicios resuelven (no hay equipos pisándose). El coste operativo de N servicios para un equipo tan pequeño es desproporcionado.
- Orden sugerido: (1) CI/CD y tests → despliegues frecuentes y seguros; (2) modularizar internamente el monolito; (3) si el escalado de búsqueda sigue doliendo, extraer únicamente ese servicio.
- Al CTO: los objetivos "desplegar a diario" y "escalar mejor" son legítimos, pero el medio propuesto no es el más barato para conseguirlos. Definir métricas (frecuencia de despliegue, coste/latencia en picos) y atacar con la intervención mínima.

**Conceptos aplicados:** migrar es un medio y no un fin; cuándo no migrar (equipo pequeño, problema real es otro); re-platforming/modularización como alternativas; extraer 1 servicio y parar.

</details>

---

### Ejercicio 1.2 — Detectar el monolito distribuido

El equipo de **PagoSeguro** migró hace dos años a 12 microservicios. Observas lo siguiente:

- A. Para añadir un campo al formulario de alta de cliente hay que modificar y desplegar `customer-api`, `customer-validation`, `customer-persistence` y `notification-service`.
- B. `customer-persistence` y `billing-service` leen y escriben en la misma base de datos Oracle.
- C. Los despliegues se hacen todos juntos el segundo martes de cada mes ("release train").
- D. `payment-service` se despliega solo, varias veces por semana, sin coordinar con nadie.
- E. Cuando `customer-validation` se cae, el alta de clientes falla por completo con timeout de 30 s.

**Preguntas:**
1. ¿Qué síntomas de monolito distribuido reconoces y por qué?
2. ¿Hay alguna parte sana del sistema?
3. Para los síntomas A y B, ¿cuál es probablemente la causa raíz?

<details>
<summary>💡 Solución</summary>

**1. Síntomas de monolito distribuido:**

- **A — Cambio funcional que atraviesa 4 servicios.** El test rápido falla: un cambio medio exige tocar y desplegar varios servicios coordinadamente. Además, los nombres (`customer-api`, `customer-validation`, `customer-persistence`) delatan que se dividió por **capas técnicas** (API/lógica/persistencia) en lugar de por dominio: cualquier cambio del dominio "cliente" atraviesa todas las capas.
- **B — Base de datos compartida.** Acoplamiento por el esquema: ninguno de los dos servicios puede cambiar sus tablas sin romper al otro. Viola la propiedad de "cada servicio es dueño de sus datos".
- **C — Release train mensual.** La consecuencia directa de A y B: como los servicios están acoplados, nadie se atreve a desplegarlos por separado. Se ha perdido *la* propiedad definitoria de los microservicios (despliegue independiente) pero se paga todo el coste de la distribución.
- **E — Fallo en cascada con timeouts largos.** Acoplamiento síncrono fuerte sin mecanismos de resiliencia (sin circuit breaker, timeout excesivo). Es señal de que la división creó una cadena síncrona obligatoria. (Lo trataremos a fondo en la Sesión 4.)

**2. Parte sana:** **D — `payment-service`**. Se despliega de forma independiente y frecuente: exactamente el comportamiento que define a un microservicio de verdad. Probablemente su límite coincide con un buen límite de dominio (pagos) y posee sus propios datos. Sirve de referencia interna: "queremos que el resto se parezca a este".

**3. Causa raíz de A y B:**

- **A:** descomposición por **capas técnicas** en vez de por **dominio de negocio**. La unidad que cambia junta (el dominio "cliente") quedó repartida en varios servicios. La corrección de fondo sería fusionar esas capas en un único servicio `customer` modelado por dominio — fusionar servicios también es una decisión válida en una migración.
- **B:** la migración dividió el **código** pero nunca dividió los **datos**. La descomposición de datos (BD por servicio) es la parte más difícil y a menudo se aplaza indefinidamente; mientras no se haga, el acoplamiento persiste por debajo. (Sesión 3.)

</details>

---

### Ejercicio 1.3 — Objetivos y criterios de éxito

Tu directora dice: *"Quiero que migremos a microservicios para ser más ágiles."*

**Tarea:** Reformula este encargo en (a) un objetivo concreto, (b) 2–3 métricas medibles con su valor actual hipotético y su valor objetivo, y (c) un criterio explícito de *parada* (cómo sabríais que ya no merece la pena seguir extrayendo servicios). Inventa cifras razonables.

<details>
<summary>💡 Solución (ejemplo de respuesta)</summary>

No hay una única respuesta correcta; lo que se evalúa es la estructura. Un ejemplo:

**(a) Objetivo concreto:**
"Queremos que los 4 equipos de producto puedan entregar cambios en producción de forma autónoma, sin coordinar releases entre sí, reduciendo el tiempo desde commit hasta producción."

**(b) Métricas (línea base → objetivo):**

| Métrica | Hoy | Objetivo (18 meses) |
|---|---|---|
| Frecuencia de despliegue | 1/mes (release train) | ≥ 2/semana por equipo |
| Lead time (commit → producción) | 3 semanas | < 2 semanas |
| Despliegues que requieren coordinación entre equipos | 100 % | < 10 % |

**(c) Criterio de parada:**
"Dejamos de extraer servicios cuando cada equipo posea los servicios que necesita para entregar de forma autónoma. Si tras extraer un candidato la matriz valor×facilidad solo deja módulos de bajo valor o alta dificultad, esos módulos se quedan en el monolito (modularizado) de forma deliberada y documentada."

**Puntos clave que debería tener cualquier solución:**
- El objetivo habla de un **resultado de negocio/entrega**, no de tecnología ("tener microservicios" no es un objetivo).
- Las métricas tienen **línea base**: sin medir el hoy, no se puede demostrar mejora.
- Existe un **criterio de parada**: una buena migración no implica extraerlo todo; "monolito modular + algunos servicios" es un final perfectamente válido.

</details>

---

### Ejercicio 1.4 — Secuenciar la migración

El monolito de **AseguraTodo** (seguros) tiene estos módulos. Las flechas indican "depende de":

```
Tarificación  ──→  Catálogo de productos
Contratación  ──→  Tarificación, Clientes, Documentos
Siniestros    ──→  Clientes, Documentos, Pagos
Clientes      ──→  (nada)
Documentos    ──→  (nada)        [genera PDFs; consume mucha CPU]
Pagos         ──→  Clientes      [cambia poco; crítico y regulado]
Catálogo      ──→  (nada)        [cambia cada semana por negocio]
```

Datos adicionales: *Documentos* provoca picos de CPU que degradan todo el sistema. *Catálogo* es donde negocio pide cambios constantemente. *Pagos* no ha cambiado en un año. *Contratación* es el corazón del negocio.

**Tarea:** Propón los **dos primeros módulos** a extraer y **uno que dejarías para el final (o nunca)**. Justifica con los criterios de secuenciación (acoplamiento, valor, volatilidad, riesgo, datos).

<details>
<summary>💡 Solución</summary>

**Primeras extracciones recomendadas:**

1. **Documentos** — Candidato casi perfecto:
   - *Acoplamiento:* sin dependencias salientes; lo consumen otros, así que basta exponer una API estable.
   - *Valor:* altísimo y medible — sus picos de CPU degradan todo el monolito; extraído, se escala de forma independiente y el beneficio se nota de inmediato (caso clásico de escalado selectivo).
   - *Datos:* un generador de PDFs suele tener pocos datos propios compartidos.
   - *Riesgo:* no es el corazón del negocio; un fallo es molesto, no catastrófico.

2. **Catálogo** — Segundo candidato fuerte:
   - *Acoplamiento:* sin dependencias salientes; solo Tarificación lo consume.
   - *Volatilidad:* es el módulo que más cambia (cambios semanales de negocio) — es donde la independencia de despliegue rinde más: cada cambio de catálogo dejaría de requerir redesplegar el monolito.
   - *Riesgo:* moderado y acotable.

   (*Clientes* también es defendible por su bajo acoplamiento saliente, pero tiene muchas dependencias entrantes —Contratación, Siniestros, Pagos— y probablemente datos muy compartidos, lo que encarece la extracción. Y su volatilidad parece baja: el beneficio es menor.)

**Dejar para el final (o nunca):**

- **Pagos** — *Volatilidad casi nula* (un año sin cambios): la independencia de despliegue no aporta apenas valor. *Riesgo altísimo* (crítico y regulado): cualquier error en la extracción es un incidente grave. En la matriz valor×facilidad cae en bajo-valor/alto-riesgo: candidato a quedarse en el monolito de forma deliberada.
- **Contratación** también se deja para tarde, pero por la razón opuesta: es el corazón del negocio y el más acoplado (3 dependencias). Conviene extraerlo cuando el equipo ya haya aprendido con extracciones menores y cuando sus dependencias (Clientes, Documentos, Tarificación) ya sean servicios con APIs estables.

**Lección general:** se empieza por módulos *hoja* (sin dependencias salientes), con dolor concreto (Documentos) o alta volatilidad (Catálogo); se aplaza lo crítico-y-estable (Pagos) y lo central-y-acoplado (Contratación).

</details>

---

## Resumen de la sesión

- Monolito y microservicios son **puntos de un espectro**; el monolito modular es un destino válido y el mejor punto de partida.
- Los microservicios compran **despliegue independiente y autonomía de equipos** pagando con **complejidad distribuida**.
- Antes de migrar: objetivo concreto, métricas con línea base, criterio de parada — y considerar re-platforming o modularización como alternativas más baratas.
- El **monolito distribuido** (servicios que deben desplegarse juntos) es el peor resultado posible; se detecta con el test "¿cuántos servicios toca un cambio medio?".
- La migración sana es **incremental**: pasos pequeños y reversibles, valor entregado por el camino, convivencia prolongada de lo viejo y lo nuevo, y borrado final del legacy.

**Próxima sesión:** la caja de herramientas de patrones de migración — Strangler Fig, Anticorruption Layer, Branch by Abstraction, Parallel Run, Bubble Context y la comparativa entre ellos.
