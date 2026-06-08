# Formación: Migración de Monolitos a Microservicios — Temario propuesto

**Formador:** Carlos Georg Lübbe
**Formato:** Remoto · **4 sesiones × 3 h = 12 h** · ~15 participantes
**Enfoque:** Conceptual y arquitectónico — primero los patrones y sus compensaciones; las tecnologías solo como ejemplos ilustrativos, no como objeto de la formación.

---

## Visión general

Este temario conserva **todos los temas del borrador original** (Introducción, patrones de migración, Estrangulador, Capa Anticorrupción, DDD, comparativa de patrones) y lo amplía con los **temas complementarios** planteados en las preguntas de scoping (panorama de patrones más amplio, descomposición de datos, comunicación entre servicios, resiliencia y observabilidad, aspectos organizativos), junto con algunas incorporaciones que completan la imagen.

El foco es **conceptual**: tratamos principios, patrones y criterios de decisión, usando tecnologías concretas solo como ejemplos.

| Sesión | Tema |
|---|---|
| **1 (3 h)** | Fundamentos y la justificación de migrar |
| **2 (3 h)** | Patrones de migración |
| **3 (3 h)** | DDD y descomposición de datos |
| **4 (3 h)** | Comunicación, resiliencia y organización |

---

## Sesión 1 — Fundamentos y la justificación de migrar

*Objetivo: vocabulario común y un encuadre claro y honesto de cuándo (y cuándo no) migrar.*

- **Introducción** — Monolitos vs. microservicios · Motivaciones y retos de la migración · Importancia de los patrones de migración
- **Más allá de la dicotomía** — Monolito modular, estilos orientados a servicios y de microservicios como un espectro, no una elección binaria
- **Justificar la división** — Definir objetivos y criterios de éxito · Cuándo *no* merece la pena migrar · Re-platforming/modularización vs. dividir · Evitar el anti-patrón del "monolito distribuido"
- **Fundamentos de estrategia de migración** — Incremental vs. big-bang · Gestión del riesgo · Buenas prácticas generales · Cómo secuenciar una migración
- **El panorama de patrones** — Mapa de los patrones que veremos y cómo se relacionan

---

## Sesión 2 — Patrones de migración

*Objetivo: una caja de herramientas completa de patrones de migración con sus ventajas, límites y encaje típico.*

- **Patrón Estrangulador (Strangler Fig)** — Concepto y origen · Arquitectura y funcionamiento · Ventajas y limitaciones · Ejemplo
- **Capa Anticorrupción (ACL)** — Concepto y relación con DDD · La capa de traducción · Beneficios y limitaciones · Ejemplo
- **Reescritura Big-Bang** — Cuándo resulta tentadora, por qué es arriesgada, cuándo puede justificarse
- **Branch by Abstraction** — Migrar tras una interfaz estable sin una rama de larga vida
- **Parallel Run** — Ejecutar lo antiguo y lo nuevo en paralelo para validar la corrección y ganar confianza
- **Bubble Context** — Crear un contexto nuevo y limpio junto al sistema legacy
- **Comparativa de patrones** — Criterios de decisión · Compensaciones · Cómo combinar patrones · Elegir según la situación

---

## Sesión 3 — Domain-Driven Design y descomposición de datos

*Objetivo: cómo encontrar buenos límites de servicio y abordar lo más difícil: los datos.*

- **Domain-Driven Design** — Principios de DDD · Lenguaje ubicuo · Contextos delimitados · Microservicios por dominio de negocio · Ejemplo
- **Encontrar los límites** — Mapas de contexto · Descomposición por capacidad de negocio vs. por subdominio · Granularidad: ¿cuándo es demasiado pequeño?
- **Descomposición de datos** — Base de datos compartida → base de datos por servicio · Dividir datos y propiedad · El coste de los datos distribuidos
- **Consistencia sin transacciones distribuidas** — Consistencia eventual · Patrón *Saga* (orquestación vs. coreografía) · Idea de Outbox · Consultas/informes entre servicios (a nivel conceptual)

---

## Sesión 4 — Comunicación, resiliencia y organización

*Objetivo: hacer que el sistema resultante sea robusto de operar y esté alineado con los equipos que lo mantienen.*

- **Comunicación entre servicios** — Síncrona vs. asíncrona · Petición/respuesta vs. orientado a eventos y a mensajes (event-driven / message-driven, brokers y colas) · API Gateway · Descubrimiento de servicios · Contratos y versionado
- **Resiliencia** — El fallo es normal en sistemas distribuidos · Circuit breaker · Reintentos, timeouts, backoff · Bulkheads · Idempotencia
- **Observabilidad** — Logs, métricas, trazado distribuido · Saber qué ocurre a través de los servicios
- **Pruebas y entrega durante la migración** — Estrategia de pruebas para un sistema en transición · Pruebas de contrato · CI/CD e independencia de despliegue (a nivel conceptual)
- **Aspectos organizativos** — Ley de Conway · Team Topologies · Propiedad de los servicios · La migración como cambio organizativo, no solo técnico
- **Síntesis y hoja de ruta** — Unir los patrones en una hoja de ruta de migración · Errores frecuentes · Preguntas y respuestas

---

## Novedades respecto al borrador

Para mayor transparencia, los temas añadidos sobre el temario original:

| Tema añadido |
|---|
| Monolito modular y el espectro monolito↔microservicios |
| Justificar la división y evitar el monolito distribuido |
| Reescritura Big-Bang, Branch by Abstraction, Parallel Run, Bubble Context |
| Descomposición de datos: BD por servicio, Saga, consistencia eventual |
| Comunicación entre servicios: sync/async, API Gateway, descubrimiento |
| Resiliencia y observabilidad: circuit breaker, reintentos, timeouts, trazado |
| Aspectos organizativos: ley de Conway, Team Topologies, propiedad |
| Pruebas y entrega durante la migración |
| Síntesis en hoja de ruta de migración |

---

*La profundidad y los tiempos de cada tema pueden adaptarse al nivel y los objetivos del grupo; la estructura de 4×3 h deja margen para profundizar en las áreas más relevantes para el cliente y aligerar el resto.*
