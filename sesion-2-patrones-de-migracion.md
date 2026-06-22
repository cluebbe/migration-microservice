<img src="logo.png" alt="NobleProg" width="280">

# Sesión 2 — Patrones de migración

**Duración:** 3 h · **Objetivo:** construir una caja de herramientas completa de patrones de migración, entendiendo las ventajas, los límites y el encaje típico de cada uno.

---

## 1. Patrón Estrangulador (Strangler Fig)

### 1.1 Concepto y origen

El nombre viene de la **higuera estranguladora**: una planta que germina en la copa de un árbol, crece a su alrededor enviando raíces hacia el suelo y, con el tiempo, el árbol original muere y la higuera queda en pie con su forma. Martin Fowler acuñó la analogía en 2004: el sistema nuevo **crece alrededor** del viejo hasta reemplazarlo, sin que en ningún momento haya un corte abrupto.

La idea esencial: **interceptar las peticiones** que van al monolito y, funcionalidad a funcionalidad, redirigirlas al sistema nuevo.

### 1.2 Arquitectura y funcionamiento

El patrón tiene tres pasos que se repiten en bucle:

1. **Identificar** una funcionalidad a extraer.
2. **Implementarla** en un servicio nuevo (o moverla, si el código es recuperable).
3. **Redirigir** las llamadas de esa funcionalidad al servicio nuevo.

La pieza clave es la **fachada de enrutado** (proxy/gateway) delante del monolito:

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

Propiedades importantes:

- La fachada se introduce **antes** de extraer nada (paso 0): primero enruta el 100 % al monolito. Esto es de bajo riesgo y deja lista la infraestructura de redirección.
- La redirección puede ser **gradual** (canary): primero el 1 % del tráfico al servicio nuevo, luego el 10 %, luego todo.
- El **rollback es trivial**: volver a apuntar la ruta al monolito.
- El corte natural es por **funcionalidad vertical** visible desde fuera (URLs, operaciones de API), no por capas internas.

### 1.3 Ventajas y limitaciones

**Ventajas:**

- Incremental por naturaleza; el sistema funciona en todo momento.
- Reversible en cada paso (cambiar el enrutado).
- El progreso es visible y medible (% de tráfico/rutas en lo nuevo).
- No exige tocar el interior del monolito (si la funcionalidad se reimplementa).

**Limitaciones:**

- Funciona mejor cuando las peticiones **entran por un borde interceptable** (HTTP es ideal). Si la funcionalidad está enterrada en lo profundo del monolito y no se distingue desde fuera, no hay dónde cortar — ahí encaja mejor *Branch by Abstraction*.
- **Los datos no se estrangulan solos:** si el servicio nuevo y el monolito comparten datos, hay que resolver la propiedad y sincronización de datos aparte (Sesión 3).
- La fachada es un punto único de paso: necesita ser fiable y observable.
- Riesgo de **estrangulamiento eterno**: si nadie prioriza terminar, se queda en híbrido para siempre.

### 1.4 Ejemplo

Un e-commerce monolítico quiere extraer "gestión de devoluciones":

1. Se coloca un gateway delante; todo el tráfico pasa por él (sin cambios de comportamiento).
2. Se construye `returns-service` con su propia BD; durante la construcción lee los datos maestros del monolito vía una API interna.
3. Se redirige `/devoluciones/*` al servicio nuevo solo para empleados internos (1 % del tráfico) → se observa → se abre al 100 %.
4. Se borra el código de devoluciones del monolito. ✂️ ← *paso que nadie debe olvidar*

---

## 2. Capa Anticorrupción (Anticorruption Layer, ACL)

### 2.1 Concepto y relación con DDD

El término viene de **Domain-Driven Design** (Eric Evans). Problema: cuando el sistema nuevo habla con el legacy, el modelo del legacy (sus nombres confusos, sus campos sobrecargados, sus reglas implícitas) tiende a **filtrarse** hacia el modelo nuevo y corromperlo. Si el servicio nuevo acaba lleno de `flagEstado2` y `tipo_cliente_aux`, la reescritura no habrá servido para limpiar nada.

La **Capa Anticorrupción** es una capa de **traducción explícita** entre los dos modelos: del lado del sistema nuevo, todo lo que llega del legacy pasa por traductores que lo convierten al modelo limpio del dominio nuevo (y a la inversa al escribir).

### 2.2 La capa de traducción

```
┌────────────────────┐        ┌─────────────────────────┐       ┌──────────────┐
│  Servicio nuevo    │        │   ACL                   │       │   Monolito   │
│  (modelo limpio)   │ ◄────► │  · adaptadores          │ ◄───► │   (modelo    │
│                    │        │  · traductores de modelo│       │    legacy)   │
│  Customer          │        │  · fachadas             │       │  KUNDE_STAMM │
│  PolicyStatus      │        │  KUNDE_STAMM→Customer   │       │  FLAG_ST_2   │
└────────────────────┘        └─────────────────────────┘       └──────────────┘
```

Componentes típicos dentro de la ACL (vocabulario de patrones clásicos):

- **Adaptador:** convierte el protocolo/tecnología (SOAP→REST, lectura directa de tablas→objetos).
- **Traductor de modelo:** mapea conceptos (el `FLAG_ST_2 = 'X'` del legacy se convierte en `PolicyStatus.SUSPENDED`).
- **Fachada:** simplifica una interfaz legacy enrevesada en operaciones con sentido de negocio.

Reglas de oro:

- La ACL **pertenece al sistema nuevo** (la escribe y mantiene el equipo del servicio nuevo, en sus términos).
- El modelo legacy **no cruza** la ACL: ningún tipo, nombre o convención del legacy aparece más allá de ella.
- Es **código de usar y tirar a largo plazo**: cuando el legacy muera, la ACL se borra con él. Eso justifica que sea pragmática, no perfecta.

### 2.3 Beneficios y limitaciones

**Beneficios:**

- Protege la integridad del modelo nuevo (la razón de ser de la reescritura).
- Aísla el cambio: si el legacy cambia, solo se toca la ACL.
- Documenta de forma ejecutable el mapeo entre el mundo viejo y el nuevo.
- Permite que el equipo nuevo trabaje con su lenguaje ubicuo sin "contaminarse".

**Limitaciones / costes:**

- Es código adicional que escribir y mantener (y a veces no es trivial: semánticas que no mapean 1:1, datos sucios).
- Añade un salto más (latencia, posible punto de fallo).
- Riesgo de convertirse en un "mini-monolito" si una sola ACL gigante centraliza todas las traducciones: mejor ACLs por contexto/servicio.

### 2.4 Ejemplo

El nuevo `policy-service` de una aseguradora necesita datos de pólizas que viven en un mainframe. En el mainframe, una póliza "activa" es `ST=02` salvo que `MIGR_FLAG='S'`, en cuyo caso hay que mirar otra tabla. La ACL encapsula esa lógica arqueológica en un método `isActive(policyId)` y expone `Policy` con el modelo limpio del servicio nuevo. Cuando el mainframe se apague, se borra la ACL y `policy-service` no nota nada más que el cambio de implementación del puerto.

---

## 3. Reescritura Big-Bang

### 3.1 Por qué es tentadora

- "El código viejo no tiene arreglo; empezar de cero será más rápido."
- Psicológicamente atractiva: campo verde, tecnología nueva, sin deuda.
- Evita la complejidad de la convivencia (proxies, sincronización de datos, dos sistemas).

### 3.2 Por qué es arriesgada

- **Objetivo móvil:** el monolito sigue evolucionando durante la reescritura (el negocio no para). El sistema nuevo persigue una especificación que cambia cada semana, o el negocio se congela durante meses — ambas cosas son carísimas.
- **El conocimiento está en el código:** un monolito de 10 años contiene miles de reglas de negocio no documentadas (casos límite, correcciones de bugs, requisitos regulatorios). La reescritura los redescubre… en producción.
- **Cero valor hasta el final:** todo el esfuerzo se entrega de golpe en un único evento de corte de altísimo riesgo, sin posibilidad de aprender por el camino.
- **Segundo sistema (Brooks):** la tentación de añadir "todo lo que faltaba" infla el alcance.
- La evidencia empírica es contundente: las reescrituras grandes big-bang fracasan o se desbordan con muchísima frecuencia.

### 3.3 Cuándo puede justificarse

Casos acotados donde es defendible:

- El sistema es **pequeño** y está **bien especificado** (p. ej. por una suite de tests exhaustiva o un comportamiento trivial de describir).
- El sistema está **congelado de verdad** (sin cambios de negocio durante la reescritura) — raro pero ocurre.
- Imposibilidad técnica real de convivencia (plataforma que desaparece, hardware sin soporte) **y** tamaño manejable.
- Reescrituras *parciales*: big-bang de **un módulo** dentro de una estrategia incremental global es a menudo razonable (cada extracción del Strangler es, en miniatura, una reescritura de esa pieza).

> **Mensaje:** el problema no es reescribir; es reescribirlo *todo* y entregarlo *de golpe*. Los patrones de esta sesión existen precisamente para trocear la reescritura.

---

## 4. Branch by Abstraction

### 4.1 El problema que resuelve

Strangler Fig intercepta llamadas **desde fuera**. Pero ¿qué pasa cuando lo que quieres reemplazar está **enterrado dentro** del monolito y lo llaman decenas de sitios del propio código? No hay borde donde poner un proxy.

La alternativa clásica era una **rama de larga vida** en el control de versiones ("la rama de la migración"), con su infierno de merges. Branch by Abstraction logra el mismo fin **en trunk**, desplegando continuamente.

### 4.2 Los cinco pasos

1. **Crear una abstracción** (interfaz) sobre la funcionalidad a reemplazar.
2. **Migrar los clientes** internos para que usen la abstracción en lugar de la implementación directa. *(El sistema sigue funcionando igual; pasos 1–2 son pura refactorización.)*
3. **Crear la implementación nueva** detrás de la misma abstracción — puede ser una llamada a un servicio externo nuevo. Se desarrolla con calma, desplegada pero inactiva.
4. **Conmutar** la abstracción a la implementación nueva (idealmente con un *feature flag*, que permite conmutar por configuración y volver atrás al instante).
5. **Limpiar:** borrar la implementación vieja y, si ya no aporta, la propia abstracción/flag.

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

### 4.3 Ventajas y limitaciones

**Ventajas:** todo ocurre en trunk con despliegues continuos; ambas implementaciones conviven y son conmutables al instante; cada paso es pequeño y seguro; combina perfectamente con Strangler (Strangler para los bordes, BbA para las entrañas).

**Limitaciones:** exige poder modificar el código del monolito (no sirve para legacy intocable o de terceros); encontrar una buena abstracción puede ser difícil si la funcionalidad está esparcida; mientras conviven dos implementaciones hay coste doble de mantenimiento; requiere disciplina para ejecutar el paso 5 (limpiar).

---

## 5. Parallel Run

### 5.1 Concepto

A veces conmutar al sistema nuevo —aunque sea gradualmente— da miedo: cálculos financieros, tarificación, decisiones regulatorias. **Parallel Run** consiste en ejecutar **las dos implementaciones a la vez** con las mismas peticiones reales y **comparar resultados**.

Variantes según quién manda:

- **Lo viejo es la fuente de verdad** (lo habitual al principio): el cliente recibe la respuesta del legacy; la del sistema nuevo se registra y compara en *shadow mode* (modo sombra). Las discrepancias no afectan a nadie: son material de depuración.
- **Lo nuevo es la fuente de verdad**: el cliente recibe la respuesta nueva; el legacy corre detrás como red de seguridad/verificación, antes de apagarse.

### 5.2 Qué se compara y cómo

- Resultados funcionales (¿la prima calculada es la misma?), efectos secundarios esperados, y también características no funcionales (¿la latencia del nuevo es aceptable?).
- Se necesita un **comparador**: desde un job nocturno que cruza registros hasta una comparación en línea por petición.
- Cuidado con los **efectos secundarios**: en modo sombra, el sistema nuevo no debe enviar emails reales, cobrar tarjetas ni escribir en sistemas externos. Hay que "desconectar" sus efectos (stubs/sandbox) — y eso es trabajo de diseño.
- Cuidado con el **no determinismo**: timestamps, redondeos, IDs generados, datos que cambian entre las dos ejecuciones. Las reglas de comparación deben tolerarlo o las discrepancias falsas ahogarán a las reales.

### 5.3 Ventajas y limitaciones

**Ventajas:** la validación más fuerte posible — datos reales, tráfico real, sin riesgo para el usuario; genera confianza cuantificable ("99,98 % de coincidencia en 3 millones de peticiones"); detecta diferencias que ningún test sintético encontraría.

**Limitaciones:** coste doble de ejecución; la infraestructura de comparación es trabajo real; difícil con efectos secundarios u operaciones de escritura (¿cuál de las dos escrituras vale?); solo verifica los casos que el tráfico real ejercita durante la ventana de comparación.

**Encaje típico:** funcionalidad de **alto riesgo y resultado comparable** (cálculos, motores de reglas, scoring). Suele usarse como fase de validación *dentro* de un Strangler o un Branch by Abstraction, no como patrón aislado: BbA pone las dos implementaciones detrás de la abstracción, Parallel Run las ejecuta ambas y compara.

---

## 6. Bubble Context

### 6.1 Concepto

Patrón de origen DDD (Eric Evans): cuando quieres empezar a modelar **bien** un dominio nuevo o muy cambiado pero todo el entorno es legacy, creas una **burbuja**: un contexto nuevo, pequeño y limpio, con su propio modelo de dominio, **protegido por una ACL** que lo separa del legacy.

```
┌────────────────────────── legacy ─────────────────────────┐
│                                                           │
│            ┌─ ACL ─┐   ┌──────────────────────┐           │
│   datos ◄──┤ trad. ├──►│   BURBUJA            │           │
│   legacy   └───────┘   │  modelo limpio,      │           │
│                        │  lenguaje ubicuo,    │           │
│                        │  reglas nuevas       │           │
│                        └──────────────────────┘           │
└───────────────────────────────────────────────────────────┘
```

Diferencia de matiz con "extraer un servicio": la burbuja se centra en **el modelo**, no en la infraestructura. Puede vivir inicialmente *dentro* del mismo despliegue del monolito (como módulo aislado) — lo esencial es la frontera de modelo, no la frontera de proceso.

Evolución típica:

1. **Burbuja simple:** sin persistencia propia; la ACL traduce desde los datos del legacy en cada acceso.
2. **Burbuja con BD propia** (*autonomous bubble*): la burbuja tiene su almacén; la ACL se convierte en una capa de **sincronización** (a menudo asíncrona) con el legacy.
3. La burbuja crece, se le da despliegue propio → se ha convertido en un microservicio nacido sano; o bien revienta la frontera y absorbe al legacy desde dentro.

### 6.2 Ventajas y limitaciones

**Ventajas:** permite aplicar DDD de verdad desde el día uno en la parte nueva; inversión inicial pequeña (no exige plataforma de microservicios); ideal cuando el motivo de la migración es un dominio nuevo o que necesita remodelarse.

**Limitaciones:** la ACL de sincronización puede volverse compleja (sobre todo con escrituras en ambos lados); si la burbuja no crece ni se consolida, queda como una isla rara dentro del legacy; requiere vigilancia constante de la frontera (la presión por "saltarse la ACL solo esta vez" es real).

---

## 7. Comparativa de patrones

### 7.1 Tabla de decisión

| Patrón | Pregunta que responde | Mejor cuando… | Riesgo principal |
|---|---|---|---|
| **Strangler Fig** | ¿Cómo reemplazo funcionalidad visible desde el borde? | Hay un punto de entrada interceptable (HTTP, mensajes) | Quedarse a medias; ignorar los datos |
| **ACL** | ¿Cómo evito que el modelo legacy contamine lo nuevo? | Siempre que lo nuevo hable con lo viejo | Convertirse en mini-monolito; coste de mantenimiento |
| **Big-Bang** | ¿Y si lo reescribo todo de golpe? | Sistema pequeño, congelado y bien especificado | Objetivo móvil; cero valor hasta el final |
| **Branch by Abstraction** | ¿Cómo reemplazo algo enterrado en el monolito? | Puedes tocar el código del monolito; muchos llamantes internos | No ejecutar la limpieza final |
| **Parallel Run** | ¿Cómo gano confianza en que lo nuevo es correcto? | Alto riesgo + salida comparable (cálculos) | Coste doble; efectos secundarios; falsas discrepancias |
| **Bubble Context** | ¿Cómo modelo limpio un dominio dentro de un entorno legacy? | El motor de la migración es remodelar el dominio | Sincronización compleja; burbuja estancada |

### 7.2 No son excluyentes: se combinan

Los patrones operan a niveles distintos y una migración real los apila:

- **Strangler Fig** como estrategia global de redirección desde el borde…
- …usando **Branch by Abstraction** para las piezas internas sin borde interceptable…
- …con una **ACL** en cada frontera entre lo nuevo y lo viejo…
- …validando las piezas críticas con **Parallel Run** antes de conmutar…
- …y arrancando los dominios a remodelar como **Bubble Contexts**.

### 7.3 Criterios transversales de elección

1. **¿Dónde está el corte?** Borde visible → Strangler. Entrañas → Branch by Abstraction.
2. **¿Cuál es el motor?** Escalado/entrega → Strangler/BbA. Remodelar el dominio → Bubble Context (+ACL).
3. **¿Cuánto miedo da equivocarse?** Mucho y comparable → añade Parallel Run.
4. **¿Puedo tocar el legacy?** No (vendor, sin fuentes) → solo bordes: Strangler + ACL.
5. **Siempre:** ACL en las fronteras y plan explícito de borrado del legacy.

---

## Ejercicios

### Ejercicio 2.1 — El bucle del estrangulador, paso a paso

**SaludPlus**, una clínica online, tiene un monolito Java accesible por HTTP. Quieres extraer la **gestión de citas**, que los pacientes usan bajo las rutas `/citas/*`. Las citas necesitan leer datos de pacientes (nombre, historia) que viven en el monolito. El monolito está en producción y no puede pararse.

**Tarea:** Describe la secuencia completa del Strangler. Para cada paso indica: qué se despliega, si cambia el comportamiento observable y cómo se haría rollback. Responde además:
1. ¿Qué haces en el "paso 0", antes de extraer nada?
2. El servicio nuevo necesita los datos de pacientes del monolito mientras se construye. ¿Cómo lo resuelves sin acoplarte a su modelo?
3. ¿Cómo verificas que el servicio nuevo se comporta bien **antes** de borrar el código viejo? ¿Y cuándo borras?

<details>
<summary>💡 Solución</summary>

**Paso 0 — Introducir la fachada.** Colocar un proxy/gateway delante del monolito que enruta el **100 % al monolito** (sin cambio de comportamiento).
*Despliegue:* sí (infraestructura de enrutado). *Comportamiento:* idéntico. *Rollback:* quitar el proxy. Es el paso de menor riesgo y deja lista la redirección.

**Paso 1 — Construir `appointments-service`** con su propia BD, desplegado pero **sin tráfico** (la fachada sigue enviando todo al monolito). Durante la construcción lee los datos de pacientes del monolito a través de una **API interna**, no leyendo sus tablas directamente.
*Comportamiento:* idéntico. *Rollback:* innecesario, no recibe tráfico.

**Paso 2 — Redirección gradual (canary).** Apuntar `/citas/*` al servicio nuevo para una porción pequeña (p. ej. citas de empleados internos, o el 1 % de pacientes) → observar métricas y errores → subir a 10 % → 100 %.
*Comportamiento:* cambia solo para la porción redirigida. *Rollback:* re-apuntar la ruta al monolito en la fachada (cambio de configuración, segundos).

**Paso 3 — Borrar** el código de citas del monolito una vez el servicio nuevo lleva estable un tiempo al 100 %. ✂️

**Respuestas:**

1. **Paso 0:** meter la fachada enrutando todo al monolito. No extrae nada todavía; solo prepara la infraestructura de redirección con riesgo casi nulo.

2. **Datos de pacientes:** el servicio nuevo los lee vía una **API interna** del monolito (no accediendo a sus tablas), y pasa esa entrada por una **ACL** que traduce el modelo del paciente legacy al modelo limpio del servicio de citas. Así, cuando los datos se separen de verdad (Sesión 3), solo cambia la implementación detrás de la ACL. *Estrangular el código no estrangula los datos.*

3. **Verificar antes de borrar:** la verificación ocurre **mientras conviven** (paso 2), no conservando el código viejo para siempre. Dos niveles: (a) la propia redirección gradual observando métricas/errores en cada escalón; (b) si el riesgo fuera alto y la salida comparable, un **Parallel Run** en sombra que compara la respuesta del servicio nuevo con la del monolito (ver Ejercicio 2.4). El rollback durante esta fase **no** depende del código viejo: se hace re-apuntando la ruta en la fachada. **Se borra** (paso 3) solo tras estabilizar al 100 % durante un periodo (p. ej. varias semanas). No borrar es el anti-patrón de **estrangulamiento eterno**: dos implementaciones y doble mantenimiento para siempre.

**Lección:** el Strangler es seguro porque cada paso es pequeño, reversible por configuración y verificable con tráfico real *antes* del corte definitivo; y porque tiene un final explícito (borrar), no un "híbrido para siempre".

</details>

---

### Ejercicio 2.2 — Diseñar una ACL

El nuevo `customer-service` necesita clientes del legacy, que los expone así (vía una vista de BD):

| Campo legacy | Contenido real |
|---|---|
| `KD_NR` | ID numérico del cliente |
| `KD_TYP` | `'P'`=particular, `'F'`=empresa, `'X'`=migrado de un sistema anterior (puede ser cualquiera de los dos; hay que mirar si `CIF_NIF` empieza por letra → empresa) |
| `STATUS_KZ` | `1`=activo, `2`=baja, `9`=activo pero moroso |
| `NAME_1`, `NAME_2` | Para particulares: apellidos / nombre. Para empresas: razón social / (vacío o nombre comercial) |

El modelo del servicio nuevo es:

```
Customer { id, type: PERSON|COMPANY, displayName, active: bool, creditHold: bool }
```

**Tareas:**
1. Escribe (en pseudocódigo o prosa) las reglas de traducción de la ACL.
2. ¿Dónde vive este código y quién lo mantiene?
3. El equipo legacy ofrece "añadir los campos nuevos directamente a la vista para que no tengáis que traducir". ¿Aceptas?

<details>
<summary>💡 Solución</summary>

**1. Reglas de traducción (pseudocódigo):**

```
function toCustomer(row):
    type = switch row.KD_TYP:
        'P' → PERSON
        'F' → COMPANY
        'X' → COMPANY si row.CIF_NIF empieza por letra, si no PERSON
        otro → error/log (dato inesperado: decidir política, p. ej. cuarentena)

    displayName = si type == PERSON:
                      row.NAME_2 + " " + row.NAME_1      // nombre + apellidos
                  si no:
                      row.NAME_1                          // razón social

    active     = row.STATUS_KZ in {1, 9}
    creditHold = row.STATUS_KZ == 9

    return Customer(id: row.KD_NR, type, displayName, active, creditHold)
```

Puntos que debe capturar cualquier solución: el caso `'X'` con su regla arqueológica encapsulada; que `STATUS_KZ=9` se descompone en **dos** conceptos limpios (`active=true` + `creditHold=true`) — el modelo nuevo no hereda el estado sobrecargado "9"; la semántica distinta de `NAME_1/NAME_2` según el tipo; y una política explícita para datos que no encajan (los datos legacy *siempre* traen sorpresas).

**2. ¿Dónde vive?** En el **lado del servicio nuevo** (`customer-service`), mantenida por su equipo. Razones: la ACL existe para proteger *su* modelo, está escrita en *sus* términos, y cuando el legacy muera se borra junto con la integración. Más allá de la ACL, ningún código del servicio debe conocer `KD_TYP` ni `STATUS_KZ`.

**3. ¿Aceptar que el legacy "traduzca" en su vista?** Como regla general, **no** — aunque la oferta sea bienintencionada:

- La traducción contiene **lógica del dominio nuevo** (`creditHold`, la regla del tipo `'X'`); ponerla en el legacy significa que el modelo nuevo pasa a estar definido por el equipo legacy, y cada ajuste del modelo nuevo requiere coordinar un cambio en el monolito — justo el acoplamiento que queremos evitar.
- La ACL en el lado nuevo se prueba, versiona y despliega con el servicio nuevo.

Sí es razonable pedirle al legacy una **interfaz más estable y limpia de protocolo** (p. ej. una API en lugar de leer tablas) — la traducción *técnica* puede acercarse al legacy; la traducción *de modelo* se queda en la ACL del lado nuevo.

</details>

---

### Ejercicio 2.3 — Branch by Abstraction, paso a paso

El monolito envía notificaciones llamando directamente a `SmtpMailer.send(...)` desde 23 sitios. Quieres que las notificaciones pasen a un nuevo `notification-service` (que además enviará SMS y push en el futuro). El monolito se despliega a diario y no puede romperse.

**Tarea:** Describe la secuencia completa de pasos, indicando para cada uno: qué se despliega, si cambia el comportamiento observable y cómo se haría rollback. Incluye el papel del feature flag y qué harías con un envío de email que falle en la implementación nueva.

<details>
<summary>💡 Solución</summary>

**Paso 1 — Crear la abstracción.** Definir `interface Notifier { send(notification) }` y hacer que `SmtpMailer` la implemente (o envolverlo en un adaptador).
*Despliegue:* sí, cambio interno. *Comportamiento:* idéntico. *Rollback:* revertir el commit (trivial, es refactorización pura).

**Paso 2 — Migrar los 23 llamantes** a usar `Notifier` en lugar de `SmtpMailer` directamente. Puede hacerse en varios despliegues pequeños (de 5 en 5 llamantes, p. ej.).
*Comportamiento:* idéntico. *Rollback:* revertir; ningún riesgo funcional.

**Paso 3 — Implementación nueva.** Crear `RemoteNotifier implements Notifier` que llama a `notification-service` por HTTP/cola. Desplegarla **inactiva** (el flag sigue apuntando a `SmtpMailer`). Construir y desplegar también el `notification-service`.
*Comportamiento:* idéntico (código nuevo desplegado pero sin tráfico). *Rollback:* innecesario, no está activo.

**Paso 4 — Conmutar con feature flag.** El flag decide qué implementación de `Notifier` se usa. Activación gradual: primero notificaciones internas o un % pequeño, observando métricas. El flag permite **rollback en segundos por configuración, sin redesplegar** — esa es su gran aportación al patrón.
*Comportamiento:* las notificaciones salen por el camino nuevo.

**Fallos en la implementación nueva:** decidir la política *antes* de conmutar. Opciones razonables: (a) *fallback* — si `RemoteNotifier` falla, reintentar con `SmtpMailer` (posible porque ambas implementaciones conviven; cuidado con duplicados → idempotencia); (b) encolar y reintentar. Lo importante es que la convivencia de implementaciones es una **ventaja de seguridad** del patrón, no solo un estado transitorio.

**Paso 5 — Limpiar.** Con lo nuevo estable (p. ej. 4 semanas al 100 %): borrar `SmtpMailer`, el flag y, si ya no aporta, los stubs. La interfaz `Notifier` probablemente se queda (es una buena costura permanente).
*Riesgo si se omite:* flags zombis y dos rutas de código para siempre — la deuda clásica de este patrón.

**Resumen de garantías:** nunca hay rama de larga vida; cada despliegue es pequeño y compatible; el comportamiento solo cambia en el paso 4, y ese paso es reversible por configuración.

</details>

---

### Ejercicio 2.4 — Parallel Run: diseñar la comparación

Vais a sustituir el cálculo de comisiones de los agentes comerciales (legacy COBOL) por un servicio nuevo. El cálculo corre cada noche para ~50.000 agentes y el resultado son apuntes que **se pagan por transferencia a fin de mes**. La dirección exige "cero errores en nóminas".

**Preguntas:**
1. ¿Qué variante de Parallel Run usarías y durante cuánto tiempo (cualitativamente)?
2. ¿Qué problema gordo de efectos secundarios hay que resolver y cómo?
3. Da tres ejemplos de discrepancias *falsas* (que no son bugs) que esperarías encontrar.
4. ¿Qué harías si descubres que en el 0,3 % de los casos discrepan… y el equivocado es el COBOL?

<details>
<summary>💡 Solución</summary>

**1. Variante y duración.** Modo sombra con **el legacy como fuente de verdad**: el COBOL sigue generando los apuntes que se pagan; el servicio nuevo calcula en paralelo y sus resultados van solo al comparador. Al ser un proceso **mensual de pago**, hay que cubrir al menos **varios ciclos completos de fin de mes** (los casos raros —ajustes, retrocesiones, agentes que causan baja a mitad de mes— solo aparecen en ciertos cierres). Cualitativamente: varios meses, con criterio de salida cuantitativo, p. ej. "≥ 2 cierres mensuales consecutivos con 100 % de coincidencia o discrepancias explicadas".

**2. Efecto secundario crítico: el dinero.** El sistema nuevo **no debe generar transferencias reales** mientras esté en sombra. Hay que cortar explícitamente su salida hacia el sistema de pagos (escribir en una tabla/archivo de sombra en lugar del canal real, o un stub del sistema de pagos). Esto debe diseñarse y *verificarse* (¡un parallel run que paga dos veces a 50.000 agentes es el incidente, no la prevención del incidente!).

**3. Discrepancias falsas típicas:**
- **Redondeos:** el COBOL usa aritmética decimal con unas reglas de redondeo; el nuevo, otras. Diferencias de ±0,01 € masivas que no son errores de lógica (aunque ojo: para pagos, hasta el redondeo debe acordarse y al final igualarse o aceptarse formalmente).
- **Momento de lectura de datos:** si los dos cálculos no leen exactamente el mismo snapshot (un contrato modificado entre la ejecución de uno y otro), discrepan sin que ninguno esté mal. Solución: ejecutar ambos sobre el mismo corte de datos.
- **Orden/formato:** IDs generados, timestamps, orden de los apuntes, codificación de textos. El comparador debe comparar el **contenido económico**, no el byte a byte.

**4. Discrepancias donde el equivocado es el legacy.** Es un hallazgo *habitual* en parallel runs y hay que tener la política decidida: **no** "corregir" el sistema nuevo para replicar el bug por defecto. Pasos: (a) documentar el caso y cuantificar el impacto (¿se ha estado pagando de más/de menos?); (b) escalarlo a negocio/legal — un error en pagos históricos puede tener implicaciones serias que no decide el equipo técnico; (c) decisión explícita por caso: corregir el comportamiento (lo normal) o, excepcionalmente, replicar el comportamiento legacy de forma documentada y temporal si hay razones contractuales. El comparador debe entonces marcar esos casos como "discrepancia conocida y aceptada" para no enmascarar discrepancias nuevas.

</details>

---

### Ejercicio 2.5 — Elegir patrón para cada situación

Para cada escenario, indica el patrón (o combinación) más adecuado y por qué:

- **A.** Una aplicación web monolítica de RR. HH. Quieres extraer el módulo de "gestión de vacaciones", que tiene sus propias pantallas bajo `/vacaciones/*`.
- **B.** El monolito usa una librería interna de cálculo de impuestos llamada desde 47 puntos del código. Quieres sustituirla por un nuevo `tax-service`.
- **C.** Vas a reemplazar el motor de tarificación de seguros. Un error de cálculo en producción tendría consecuencias regulatorias.
- **D.** El negocio lanza una línea de producto totalmente nueva ("seguros para mascotas") que apenas comparte reglas con el legacy, pero necesita los datos de clientes existentes.
- **E.** Una utilidad interna de 3.000 líneas que genera un informe semanal, sin cambios funcionales en 4 años y con una suite de tests que cubre todos los casos. Hay que sacarla de una plataforma que se apaga en 6 meses.

<details>
<summary>💡 Solución</summary>

- **A — Strangler Fig.** Existe un borde perfecto: las rutas `/vacaciones/*` son interceptables por un proxy. Pasos: fachada delante del monolito → construir `vacation-service` → redirigir las rutas (gradualmente si se quiere) → borrar el módulo del monolito. Con una **ACL** si el servicio nuevo necesita leer datos del legacy (empleados, calendarios).

- **B — Branch by Abstraction.** No hay borde externo: 47 llamantes *internos*. Crear la interfaz `TaxCalculator`, migrar los 47 puntos a la interfaz (refactorización segura), implementar detrás una versión que llama a `tax-service`, conmutar con feature flag, borrar la librería vieja. Strangler no aplica porque no hay nada que interceptar desde fuera.

- **C — Parallel Run** (montado sobre Branch by Abstraction o Strangler). El riesgo es alto y la salida (una prima calculada) es perfectamente comparable. Ejecutar ambos motores en sombra con tráfico real, medir el porcentaje de coincidencia, investigar discrepancias (¡algunas serán bugs del *legacy*!), y conmutar solo con evidencia cuantitativa. Atención a los efectos secundarios y al no determinismo (redondeos) en la comparación.

- **D — Bubble Context** (con su ACL). El motor es modelar un dominio **nuevo** con reglas propias: nace como contexto limpio con su propio modelo, y la ACL traduce/sincroniza los datos de clientes desde el legacy. Con el tiempo, si crece, se convierte en servicio independiente. Empezar esta línea *dentro* del modelo legacy lo contaminaría desde el día uno.

- **E — Big-Bang (reescritura directa), y es la elección correcta.** Es el caso acotado donde se justifica: pequeño (3.000 líneas), congelado (4 años sin cambios), especificado por una suite de tests completa, y con una fecha límite externa real. Montar un Strangler con convivencia para esto sería sobreingeniería. Lección: big-bang no está prohibido; está prohibido para sistemas grandes, vivos y mal especificados.

</details>

---

### Ejercicio 2.6 — Combinar patrones: plan integral

**MediaStream** (plataforma de vídeo, monolito de 12 años, 6 equipos) quiere: (1) extraer las recomendaciones, que hoy son llamadas internas desde 30 puntos del código y quieren reescribirse con un modelo de dominio nuevo basado en ML; (2) extraer la facturación, accesible solo vía las rutas `/billing/*`, cuyo cálculo de prorrateos es delicado; (3) que nada se rompa.

**Tarea:** Propón qué patrón(es) aplicar a cada extracción y en qué se diferencian los dos planes. Señala dónde pondrías ACLs y dónde un Parallel Run.

<details>
<summary>💡 Solución</summary>

**Recomendaciones — Branch by Abstraction + Bubble Context (+ ACL):**
- No hay borde externo (30 llamantes internos) → el mecanismo de conmutación es **Branch by Abstraction**: interfaz `Recommender`, migrar llamantes, implementación nueva detrás, flag, limpiar.
- Como además se quiere un **modelo de dominio nuevo** (ML, conceptos distintos), el servicio nuevo nace como **Bubble Context**: modelo limpio propio, con una **ACL** que traduce los datos del monolito (historial de visualizaciones, catálogo) a los conceptos del modelo nuevo.
- Parallel Run estricto no encaja bien aquí: las recomendaciones nuevas *quieren* ser distintas (no hay "respuesta correcta" que comparar). La validación natural es un **experimento A/B** sobre métricas de negocio (engagement), que es la versión producto del mismo espíritu: ambas implementaciones en producción, comparación con datos reales, conmutación basada en evidencia.

**Facturación — Strangler Fig + Parallel Run (+ ACL):**
- Hay borde perfecto (`/billing/*`) → **Strangler Fig**: fachada, construir `billing-service`, redirigir rutas gradualmente.
- El cálculo de prorrateos es delicado y **comparable** (importes) → **Parallel Run** en sombra antes de conmutar: el monolito sigue facturando, el servicio nuevo calcula en paralelo, se comparan importes durante varios ciclos de facturación. Cortar los efectos secundarios (no cobrar dos veces).
- **ACL** en `billing-service` para traducir lo que necesite del monolito (suscripciones, clientes) sin heredar su modelo.

**Diferencias clave entre ambos planes:**

| | Recomendaciones | Facturación |
|---|---|---|
| Mecanismo de corte | Branch by Abstraction (llamantes internos) | Strangler Fig (borde HTTP) |
| Naturaleza | Remodelado de dominio (bubble) | Reemplazo equivalente |
| Validación | A/B sobre métricas de negocio | Parallel Run sobre corrección numérica |
| ACL | Sí (datos de visualización/catálogo) | Sí (suscripciones/clientes) |

Y en ambos: paso final de **borrado** del código legacy correspondiente.

</details>

---

## Resumen de la sesión

- **Strangler Fig:** interceptar en el borde y redirigir gradualmente; reversible, visible, pero no resuelve los datos.
- **ACL:** capa de traducción que impide que el modelo legacy contamine el nuevo; pertenece al lado nuevo; se borra con el legacy.
- **Big-Bang:** casi siempre mala idea para sistemas grandes y vivos; justificable en piezas pequeñas, congeladas y bien especificadas.
- **Branch by Abstraction:** el "strangler de interior" — abstracción, dos implementaciones, flag, limpieza; todo en trunk.
- **Parallel Run:** ambos sistemas con tráfico real y comparación de resultados; la validación más fuerte para funcionalidad crítica y comparable.
- **Bubble Context:** un contexto limpio protegido por ACL para remodelar el dominio dentro del entorno legacy.
- Los patrones **se combinan**: Strangler para los bordes, BbA para las entrañas, ACL en toda frontera, Parallel Run donde el riesgo lo pida, Bubble donde haya que remodelar.

**Próxima sesión:** *dónde* cortar (DDD, bounded contexts) y el problema más duro de todos: descomponer los datos.
