# Life OS — Auditoría del proyecto y decisiones tomadas

Fecha: 2026-08-20 · Estado: **análisis, sin código**

Este documento registra (a) la crítica al planteamiento inicial, (b) los hallazgos de una
auditoría hecha con cuatro revisores independientes y (c) las decisiones que resultan de
ella. Es el documento que explica *por qué* `ARCHITECTURE.md` dice lo que dice.

---

## 1. Método

Se auditó la especificación del producto y un primer borrador de arquitectura con cuatro
revisiones independientes y sin contacto entre sí, cada una con una lente distinta:

| Revisión | Lente | Foco |
|---|---|---|
| A | Producto y UX | Núcleo del producto, fricción diaria, riesgo de abandono, tono |
| B | Arquitectura y datos | Modelo de tiempo, recurrencia, esquema, rendimiento, tests |
| C | IA aplicada | Reparto determinista/LLM, visión para fotos, costos, privacidad, evaluación |
| D | Seguridad y entrega | RLS, Supabase, datos sensibles, viabilidad, puntos de no retorno |

Que trabajen aisladas es lo que da valor: **donde coinciden, es señal; donde se
contradicen, hay una decisión real que tomar** (§5).

---

## 2. Crítica al planteamiento inicial

La especificación estaba muy por encima del promedio: alcance claro, tono definido,
límites explícitos para la IA y una metodología correcta (analizar antes de construir).
Los problemas no estaban en lo que decía, sino en lo que faltaba.

1. **Define funciones, no reglas de desempate.** Enumera qué debe hacer la app, pero casi
   nunca qué debe preferir cuando dos requisitos chocan. Caso real: una tarea lleva 12
   días sin hacerse. "Subir su prioridad" y "no castigar al usuario" son ambas reglas del
   documento y apuntan en direcciones opuestas.
2. **No hay criterio de éxito.** Sin una métrica, "asistente inteligente" es una opinión y
   ninguna versión puede declararse buena o mala. → Resuelto en §4.8.
3. **Falta un "día en tu vida" real.** Un solo guion con horarios reales vale más para
   diseñar el motor que cuatro secciones de requisitos. Sigue pendiente.
4. **Contradicción metodológica.** Pide no construir todo de una vez y a continuación
   lista diez módulos con sus campos; esa lista invita justamente a construirlo todo.
5. **El riesgo principal no aparece.** Estas apps mueren porque mantener los datos al día
   cuesta más de lo que devuelven. No había nada sobre captura rápida, recordatorios ni
   qué ocurre si un día no se registra nada.
6. **Faltaban restricciones duras**: presupuesto de IA, plazo, número de usuarios,
   integración con calendario externo, zona horaria, moneda, notificaciones, offline.
7. **La IA estaba subespecificada**: se define qué no debe hacer, pero no si es el motor
   del producto o una capa encima. Es la decisión técnica más importante del proyecto.

---

## 3. Hallazgos

### 3.1 Lo que las cuatro revisiones confirman

- **El motor de decisión debe ser determinista y la IA una capa opcional encima.**
  "¿Qué hago ahora?" es un `argmax` sobre unas decenas de filas: un LLM lo haría peor,
  mucho más lento, más caro y sin explicación auditable. Esta es la decisión que hay que
  defender contra la tentación de "meterle IA a todo".
- **La explicación del porqué debe ser una plantilla, no texto generado.** Añade latencia
  y variabilidad de tono, y arriesga producir copy culposo — justo lo prohibido.
- **El flujo de confirmación del OCR no es un extra: es el producto.** Con seis clases y
  90% de acierto por celda, hay aproximadamente un error por importación.
- **`user_id` uniforme en todas las tablas y RLS desde la primera migración.** Es lo único
  verdaderamente caro de añadir después.
- **La viabilidad real del alcance completo es de 4 a 7 meses**, no de unas semanas.

### 3.2 Errores encontrados en el borrador de arquitectura

Ordenados por gravedad. Todos están corregidos en `ARCHITECTURE.md`.

| # | Error | Corrección |
|---|---|---|
| 1 | **Recurrencia "materializar la siguiente instancia al completar"**. Una tarea mensual que nunca se completa no vuelve a existir jamás; los hábitos, que por definición tienen días sin completar, no tienen nada que materializar; no se distingue "falló" de "no tocaba"; la agenda futura queda vacía | Tareas: plantilla + instancias materializadas en ventana deslizante de 60 días con `UNIQUE(template_id, occurrence_date)`. Hábitos: no se materializan; la racha se deriva de días esperados vs. registrados |
| 2 | **Fórmula del ciclo rotativo con bug real.** `floor(diffInWeeks(...)) % n` da índice negativo para fechas anteriores al ancla (ninguna fila coincide), `startOfWeek` depende del locale y el cálculo sobre instantes se rompe con el cambio de hora | Módulo euclídeo `((n%m)+m)%m` sobre **días civiles**, sin `startOfWeek` |
| 3 | **Ciclos medidos en semanas.** No expresan turnos 4x3, 6x2 ni 7x7, que es exactamente lo que la especificación pedía | `cycle_length_days` + `day_offset`. Un horario semanal fijo es `cycle_length_days = 7`: una sola ruta de código, ahora correcta |
| 4 | **Turnos que cruzan medianoche sin modelar.** `end_time < start_time` es ambiguo y una consulta por día pierde el derrame del turno 22:00–06:00 | `duration_minutes` en vez de `end_time`; el resolvedor consulta el día D y el D−1 |
| 5 | **`shared_items` polimórfico** (`entity_type` + `entity_id`): sin clave foránea, con filas colgantes al borrar, políticas RLS con `CASE` imposibles de indexar y — lo grave — **cualquier miembro podía publicar el identificador de un dato ajeno y leerlo** | Columnas con FK real y arco exclusivo. Además, la compartición queda fuera del alcance (§4.4) |
| 6 | **"Hoy" mal definido.** La zona horaria del perfil no basta con turnos nocturnos: a las 01:30 la app diría que se rompió la racha | `day_cutoff_hour` (4:00 por defecto) y `local_date` materializado junto a `occurred_at` en cada registro |
| 7 | **`NUMERIC(14,2)` para el dinero.** El peso chileno no tiene decimales y `numeric` llega a JavaScript como texto, lo que termina en `parseFloat` y en errores de redondeo | `amount_minor bigint` + moneda congelada en cada fila |
| 8 | **Finanzas incomputables.** "Dinero disponible" y "patrimonio" no se pueden calcular solo con ingresos y gastos, y sin aportes fechados no existe la proyección de "lo alcanzarás en X meses" | Añadidas `financial_accounts`, `savings_goal_contributions` y `net_worth_snapshots` |
| 9 | **`is_space_member()` sin `SET search_path = ''`** ni revocación de ejecución, y sin filtrar invitaciones pendientes: invitar equivalía a dar acceso | Corregido en el patrón documentado, aunque la función no se implementa todavía |
| 10 | **Trigger de creación de perfil**: `raw_user_meta_data` lo controla quien se registra; copiarlo sin lista blanca es escalada de privilegios | Lista blanca explícita de campos |
| 11 | **Dashboard con ~10 consultas** más RLS sin `InitPlan` y posible región cruzada entre Vercel y Supabase, en la pantalla que se abre diez veces al día | Una función RPC `get_dashboard(p_date)`, `(select auth.uid())` en toda política, región de Vercel igual a la de Supabase |
| 12 | **PWA sin service worker en la primera fase**: en iPhone, una PWA instalada sin service worker muestra pantalla en blanco al abrirse sin señal | Service worker mínimo desde el principio (además, ahora obligatorio para notificaciones) |
| 13 | **Filtros del motor que usan campos inexistentes** (divisible, bloque de descanso, dependencia) y comportamiento indefinido cuando la duración estimada es nula, que es el caso normal | Campos declarados explícitamente y valor por defecto de 30 minutos cuando falta la estimación |
| 14 | **Pesos del motor en JSONB completos**: al añadir un factor, todas las filas antiguas quedan sin peso, y si el usuario los edita pueden no sumar 1 | Se guardan solo las diferencias respecto al valor por defecto; los valores por defecto viven en el código |
| 15 | **`spiritual_activities` + `church_events` como módulo aparte** duplican racha, registro y lógica de prioridad que los hábitos ya tienen | Oración y lectura bíblica pasan a ser hábitos con categoría `faith`; la iglesia es un evento con una regla condicional. Dos tablas menos |

### 3.3 Riesgo de producto

El diagnóstico más incómodo, y en el que coinciden las revisiones de producto y de
entrega: el peligro no es técnico, es que la app **exija más de lo que devuelve**. Diez
módulos significan diez formularios que alimentar a diario. Si "¿qué hago ahora?" se
alimenta de datos incompletos, recomienda mal; si recomienda mal dos veces seguidas, se
pierde la confianza y no se recupera.

Consecuencias asumidas en la arquitectura:

- Cada módulo debe **producir valor con datos incompletos**, nunca exigir completitud.
- El botón "Empezar" registra tiempo real (`focus_sessions`); ese registro alimenta el
  progreso de aprendizaje y de metas sin que haya que teclear porcentajes a mano.
- "Ahora no" ofrece cuatro motivos de un toque. Ese motivo es la señal que calibra el
  motor, y por eso el rechazo se diseña como una función, no como un fallo.

---

## 4. Decisiones tomadas

### 4.1 Alcance de módulos — los cuatro

Decisión del usuario: entran hábitos y fe, finanzas y ahorros, aprendizaje y metas. Se
construyen **en ese orden**, no en paralelo, y cada uno se usa una semana real antes de
empezar el siguiente. El orden no es arbitrario: hábitos y fe alimentan el día con poca
fricción; finanzas es independiente del motor y da una victoria visible; aprendizaje se
apoya en `focus_sessions`; metas necesita que existan tareas y recursos que enlazar.

Queda dicho con claridad, porque es un dato del proyecto y no una opinión: el alcance
completo son **entre cuatro y siete meses** de trabajo sostenido.

### 4.2 Calendario propio, con Google después

El MVP no depende de OAuth ni de sincronización. Pero `events` nace con
`external_source` y `external_id` desde la primera migración, para que importar Google
Calendar más adelante sea una fase y no una migración de datos.

### 4.3 Notificaciones contextuales

Decisión del usuario: avisos antes de una clase y a la hora de orar. Esto añade
infraestructura que el borrador no tenía, y **una limitación dura que conviene conocer
antes de invertir en ella**:

- En iPhone, las notificaciones web **solo funcionan si la app está instalada en la
  pantalla de inicio** (iOS 16.4+) y el permiso se pide desde un gesto del usuario. En
  Safari sin instalar, no hay notificaciones. No es evitable sin una app nativa.
- Por eso el service worker deja de ser opcional y entra en la fase 1.
- El programador de recordatorios no puede vivir en el cron de Vercel (el plan gratuito
  permite una ejecución diaria). Va en **pg_cron dentro de Supabase**, cada cinco minutos,
  llamando a una Edge Function que envía Web Push con VAPID.
- Tablas nuevas: `push_subscriptions` y `notifications_sent` (esta última evita el
  duplicado clásico de "el mismo aviso tres veces").
- Reglas de tono, no negociables: horario silencioso configurable, máximo de avisos al
  día, y **ningún aviso que mencione rachas rotas o incumplimientos**.

### 4.4 Espacio compartido: fuera del alcance

El nombre del repositorio no implica un producto para dos. Se elimina de la interfaz y de
las tablas. Lo que **sí** se mantiene, porque es lo único caro de añadir después, es
`user_id` con el mismo nombre y tipo en todas las tablas, y el patrón de RLS documentado
para el día en que haga falta. Se descarta explícitamente el diseño polimórfico.

### 4.5 Módulo de fe fusionado con hábitos

Oración y lectura bíblica se modelan como hábitos con categoría `faith` (los detalles de
libro y capítulo van en `metadata`); la iglesia es un evento con una regla condicional
evaluada contra la disponibilidad real. Se ganan rachas unificadas y se pierden dos
tablas. **Esta decisión conviene confirmarla**, porque se aparta de la lista de entidades
de la especificación original.

### 4.6 Privacidad frente al LLM, verificable por el compilador

No basta con una promesa en la documentación:

- Al modelo **nunca** se le envían montos, saldos, comercios, notas de oración, notas
  personales, apellidos, correo, RUT ni identificadores internos.
- Los identificadores viajan como índices locales (`t1`, `t2`…). Toda referencia devuelta
  que no estuviera en el contexto enviado se descarta entera: así es estructuralmente
  imposible que la IA invente una tarea.
- El gateway de IA solo acepta tipos `Redacted*`, que no tienen los campos prohibidos:
  **el verificador es el compilador**, no la disciplina.
- Un test en integración continua serializa el payload final de cada prompt y comprueba
  que no aparecen valores señuelo.
- La IA es opcional y se puede apagar por completo con una variable de entorno, sin
  desplegar, degradando al motor determinista.

Costo estimado con uso realista: **menos de un dólar por usuario y mes**, dominado por la
importación de horarios. Las recomendaciones del día a día cuestan cero porque no usan
LLM.

### 4.7 Horario real: los bloques fechados sustituyen a los ciclos

Se analizó una carta de horario mensual real (supermercado, operador de sala de venta,
agosto de 2026). El hallazgo obliga a reescribir el modelo de tiempo por tercera vez, y
esta vez **para simplificarlo**.

Lo que muestra el documento:

| Semana | Lun | Mar | Mié | Jue | Vie | Sáb | Dom |
|---|---|---|---|---|---|---|---|
| 10–16 ago | libre | libre | 9:00–16:00 | 9:00–16:00 | 9:00–16:00 | 9:00–16:00 | 11:30–19:30 |
| 17–23 ago | 13:00–21:30 | libre | 13:00–21:30 | 13:00–21:30 | 13:00–21:30 | 13:00–21:30 | libre |
| 24–30 ago | 9:00–16:00 | libre | 9:00–16:00 | 9:00–16:00 | 9:00–16:00 | libre | 11:30–16:00 |

*(Lectura preliminar sujeta a confirmación: los totales por semana no cuadran con las 45
horas que declara el contrato, así que probablemente falte interpretar una fila.)*

**Conclusiones:**

1. **No hay ciclo.** No es semana A/B ni 4x3: es una malla asignada mes a mes, con turnos
   distintos cada semana y días libres irregulares. El propio documento advierte que
   "podrá sufrir modificaciones… por situaciones del día a día". Un modelo de patrones
   cíclicos, que es lo que asumían las dos versiones anteriores de la arquitectura, **no
   representa esto**.
2. **Los bloques fechados pasan a ser la representación canónica** (`time_blocks`). Los
   patrones cíclicos siguen existiendo para las clases, pero como *generadores* de
   bloques, no como algo que se resuelva en tiempo real.
3. **Se eliminan dos tablas y una capa entera de lógica**: `schedule_exceptions` (editar
   una excepción es ahora editar el bloque) y `events` (un evento es un bloque con otro
   `kind`). Desaparecen también la ambigüedad de los turnos que cruzan medianoche y la
   regla de "consultar también el día anterior".
4. **La regla de la iglesia necesitaba este dato.** En dos de los tres domingos hay turno,
   y ambos empiezan a las 11:30 — justo cuando terminaría un culto de 10:30. La condición
   correcta no es "¿trabaja ese día?" sino "¿se solapa algún bloque con la ventana del
   culto más el traslado?". Un modelo por día entero habría fallado en los dos casos.
5. **La foto contiene datos personales** (nombre completo, RUT, local, nombre de la
   jefatura). Confirma la política de borrar la imagen al confirmar la importación y de no
   enviarla a ningún servicio que no sea el modelo de visión.

Es el mejor argumento a favor de haber pedido material real antes de programar: el modelo
que aguanta la realidad resultó ser **más simple** que el que la anticipaba.

### 4.8 Criterio de éxito

Métrica única, medible desde la primera semana:

> **Aceptación de la primera recomendación.** De cada cinco veces que la app propone algo,
> ¿cuántas se empiezan? Objetivo: tres de cada cinco. Por debajo de dos, el motor no
> sirve y ningún modelo de lenguaje lo va a arreglar.

Se complementa con un **conjunto dorado de unas treinta situaciones congeladas** (huecos
libres, tareas y hora, junto con la respuesta que se considera correcta) que se ejecuta
en integración continua cada vez que se toca un peso. Cuesta una tarde y evita la
calibración a ciegas.

---

## 5. Dónde las revisiones se contradicen

Vale la pena registrarlo: son las decisiones que no tienen una respuesta obvia.

| Tema | Postura A | Postura B | Decisión |
|---|---|---|---|
| Estados como `enum` de Postgres o `text` + `CHECK` | Enum: se generan tipos de unión en TypeScript automáticamente | Texto: los enums son dolorosos de alterar | **Enum** para dominios cortos y estables (estado de tarea, prioridad, tipo de transacción), porque el tipado automático evita una clase entera de errores; **tabla** para todo lo que el usuario configura (categorías). Añadir un valor a un enum es fácil; lo doloroso es renombrar, y estos no se renombran |
| Caché de datos en cliente | Innecesaria en el MVP | Se echará de menos al deslizar entre días en la agenda | Sin caché de cliente hasta la fase de agenda; se reevalúa **ahí**, no al final |
| Cifrado de columnas financieras | Suena prudente | Rompe las agregaciones y la clave viviría junto al servidor: seguridad de teatro | **No se hace**. TLS y cifrado en reposo del proveedor, dicho con honestidad |

---

## 6. Lo que sigue pendiente de ti

1. **Confirmar la lectura de la carta de turnos** (§4.7): sobre todo si falta una fila por
   día y a qué corresponde, porque las horas semanales no cuadran con las 45 del contrato.
2. **Qué ocupa el resto de tu día**: estudios o clases con su horario, hora de dormir y de
   levantarte, y cuánto tardas en llegar al trabajo. El turno es solo una parte; el motor
   necesita saber qué queda alrededor.
3. **Confirmar la fusión de fe con hábitos** (§4.5).
4. **Una foto de un horario de clases**, si estudias. Es un formato distinto al de la
   carta de turnos (rejilla repetible frente a malla fechada) y conviene probar los dos.
5. **Aprobar el orden de construcción** de `ARCHITECTURE.md` §16.
