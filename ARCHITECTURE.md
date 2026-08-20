# Life OS — Arquitectura

> **Documento de diseño, no de implementación.** Nada de esto está construido. Su objetivo
> es que puedas revisarlo y aprobarlo antes de escribir código.
>
> Esta es la **versión 2**, corregida tras la auditoría registrada en
> [`AUDITORIA.md`](./AUDITORIA.md). Quince errores de la versión 1 están corregidos aquí;
> los más graves eran el modelo de recurrencia, la fórmula de los horarios rotativos y el
> modelo de compartición. Léela junto a ese documento: allí está el *por qué*.
>
> `DATABASE.md`, `AI.md`, `SECURITY.md` y `ROADMAP.md` se escribirán **después** de que
> apruebes esto, para que no nazcan desactualizados.

Índice:

1. [Principio rector](#1-principio-rector)
2. [Decisiones de partida](#2-decisiones-de-partida)
3. [Stack](#3-stack)
4. [Arquitectura general](#4-arquitectura-general)
5. [Estructura de carpetas](#5-estructura-de-carpetas)
6. [Autenticación y autorización](#6-autenticación-y-autorización)
7. [Modelo de tiempo](#7-modelo-de-tiempo)
8. [Recurrencia: tareas y hábitos](#8-recurrencia-tareas-y-hábitos)
9. [Tareas postergadas](#9-tareas-postergadas)
10. [Motor de priorización](#10-motor-de-priorización)
11. [Notificaciones](#11-notificaciones)
12. [Navegación](#12-navegación)
13. [Dashboard](#13-dashboard)
14. [Modelo de datos](#14-modelo-de-datos)
15. [Arquitectura de IA](#15-arquitectura-de-ia)
16. [MVP y orden de construcción](#16-mvp-y-orden-de-construcción)
17. [Riesgos técnicos](#17-riesgos-técnicos)
18. [Dependencias](#18-dependencias)

---

## 1. Principio rector

Toda decisión se subordina a una pregunta de producto: **¿qué importa realmente hoy?**
De ahí salen cuatro reglas que atraviesan el sistema.

**R1 — El motor de decisión es determinista; la IA es una capa encima.**
Elegir qué hacer ahora es un `argmax` sobre unas decenas de filas. Un modelo de lenguaje
lo haría peor, más lento, más caro y sin explicación auditable. Si la IA está caída o
apagada, la app sigue respondiendo la pregunta central. La IA aporta visión (fotos de
horarios), lenguaje natural de entrada y planificación de día completo — no la lógica
base.

**R2 — La IA propone con referencias cerradas; el usuario confirma.**
No basta con "la IA no escribe en tablas de dominio": eso no impide que invente una
tarea. Al modelo se le pasan **índices locales** (`t1`, `t2`…), nunca identificadores
reales, y **toda referencia devuelta que no estuviera en el contexto enviado se descarta
entera**. Inventar deja de ser posible por construcción, no por buena voluntad.

**R3 — Privado por defecto, y verificable.** RLS en cada tabla desde la primera
migración. La privacidad frente al LLM se garantiza con tipos y con un test que inspecciona
el payload, no con una promesa en la documentación.

**R4 — La app debe dar valor con datos incompletos.** Es la regla que decide si el
proyecto sobrevive al mes. Nada puede exigir que el usuario mantenga diez módulos al día:
cada módulo funciona con lo que haya, y el registro se deriva del uso siempre que se
pueda (el botón "Empezar" mide tiempo real en vez de pedir un porcentaje escrito a mano).

---

## 2. Decisiones de partida

Tomadas y cerradas. Cambiarlas más adelante cuesta, así que están aquí explícitas.

| Tema | Decisión |
|---|---|
| Módulos | Tareas y horarios (núcleo, siempre activos) + **los cuatro**: hábitos y fe · finanzas y ahorros · aprendizaje · metas. Se construyen **en serie**, cada uno usado una semana real antes del siguiente |
| Calendario | **Propio**. `events` incluye `external_source` y `external_id` desde el día 1 para importar Google Calendar en una fase posterior sin migrar datos |
| Notificaciones | **Sí, contextuales**: antes de una clase o turno, y a la hora de un hábito. Con horario silencioso y tope diario. Ver §11 |
| Espacio compartido | **Fuera del alcance.** Se conserva `user_id` uniforme en todas las tablas y el patrón de RLS documentado, que es lo único caro de añadir después |
| Fe | Oración y lectura bíblica se modelan **como hábitos** con categoría `faith`; la iglesia es un evento con regla condicional. *Pendiente de tu confirmación* |
| Moneda | Una por perfil, `CLP` por defecto. `amount_minor bigint` + moneda congelada en cada fila. Sin conversión entre monedas |
| Zona horaria | Una por perfil (IANA). Nótese que Chile tiene tres: `America/Santiago`, `America/Punta_Arenas` y `Pacific/Easter` |
| Offline | Online-first. Service worker desde la fase 1 (obligatorio para notificaciones en iPhone y para no mostrar pantalla en blanco sin señal), sin cola de escrituras |
| IA | Opcional, apagable por variable de entorno, con tope de gasto. Coste estimado: menos de 1 USD por usuario y mes |

---

## 3. Stack

| Capa | Elección | Por qué |
|---|---|---|
| Framework | **Next.js 15+, App Router** | Server Components reducen el JavaScript que paga el móvil; Server Actions evitan escribir una API paralela |
| Lenguaje | **TypeScript** estricto | Tipos generados desde el esquema: la base de datos es la fuente de verdad |
| Estilos | **Tailwind CSS v4** | Configuración en CSS, sin archivo de configuración pesado |
| Componentes | **shadcn/ui** | El código vive en tu repositorio: nada opaco, accesibilidad resuelta |
| Backend | **Supabase** (Postgres + Auth + Storage + RLS + Edge Functions + pg_cron) | Autorización por fila sin escribir backend; el programador de notificaciones vive aquí |
| Validación | **Zod** | Un esquema valida el formulario, la acción del servidor y la salida del modelo |
| Fechas | **date-fns** + **@date-fns/tz** | Ligero. Todo cálculo civil se hace en la zona del perfil, nunca sobre instantes crudos |
| IA | **Anthropic API**, solo en servidor | Visión para horarios, extracción de lenguaje natural, plan del día |
| Hosting | **Vercel**, en la misma región que el proyecto Supabase | Región cruzada añade más de 100 ms **por consulta**: es la decisión que más afecta a la sensación de rapidez |
| PWA | Manifest + **Serwist** desde la fase 1 | Instalable y con notificaciones en iPhone |

**Tres precisiones.**

**a) Sin librería de estado global ni caché de datos en cliente, por ahora.** Server
Components + Server Actions + `revalidateTag` (etiquetas finas, **no** `revalidatePath`,
que invalida la pantalla entera y convierte seis hábitos marcados en decenas de
consultas) y `useOptimistic` para el feedback inmediato. Se reevalúa al llegar a la
agenda, donde deslizar entre días con ida y vuelta al servidor se sentirá mal en móvil.

**b) Server Actions para casi todo**, con excepciones donde no encajan: webhooks, subida
de imágenes (van del navegador a Storage con URL firmada, sin atravesar la función: el
límite por defecto de una Server Action es 1 MB) y la visión, que tarda entre veinte y
sesenta segundos y necesita un trabajo asíncrono con estado y consulta de progreso.

**c) La animación empieza en CSS.** `motion` solo si aparece una interacción que lo
justifique; son unos 30 KB que el móvil paga en cada carga.

---

## 4. Arquitectura general

```
┌──────────────────────────────────────────────────────────────┐
│  CLIENTE (PWA instalable)                                    │
│  Server Components + islas de cliente + service worker       │
│  Sin claves. Sin lógica de autorización.                     │
└───────────────┬──────────────────────────────────────────────┘
                │ Server Actions · Route Handlers
┌───────────────▼──────────────────────────────────────────────┐
│  SERVIDOR (Next.js)                                          │
│  ① Aplicación   Server Actions + validación Zod              │
│  ② Dominio      Motores puros, sin entrada/salida:           │
│     time · availability · scoring · planner · streaks ·      │
│     finance                                                  │
│  ③ Datos        Cliente Supabase con la sesión del usuario   │
│  ④ Redacción    Constructor de contexto: tipos Redacted*     │
│  ⑤ IA           Anthropic SDK — solo alcanzable desde ④      │
└───────────────┬──────────────────────────────────────────────┘
                │ PostgREST con el JWT del usuario
┌───────────────▼──────────────────────────────────────────────┐
│  SUPABASE                                                    │
│  Postgres + RLS  ← autoridad final de acceso                 │
│  Auth · Storage privado · Edge Functions · pg_cron           │
└──────────────────────────────────────────────────────────────┘
```

La capa ④ no es burocracia: es lo que convierte "no enviamos datos sensibles" en algo que
el compilador comprueba. La capa ⑤ es inalcanzable sin pasar por ella.

**La capa ② es el corazón del producto y se prueba sin base de datos**: funciones puras
que reciben datos y devuelven decisiones. Con una condición estricta — **ninguna llama a
`new Date()` ni consulta la zona horaria del entorno por dentro**. `now` y `timezone` se
inyectan siempre. Sin eso, los casos de cambio de hora son imposibles de probar y los
tests fallan según la máquina donde corran.

**Regla de dependencias:** `app/ → modules/* → core/* → lib/*`. Un módulo nunca importa
de otro módulo; lo que comparten sube a `core/`.

---

## 5. Estructura de carpetas

```
life-os/
├── src/
│   ├── app/
│   │   ├── (auth)/            login · registro · recuperar · auth/callback
│   │   ├── (app)/             área privada
│   │   │   ├── hoy/           Dashboard
│   │   │   ├── ahora/         "¿Qué debería hacer ahora?"
│   │   │   ├── agenda/        día · semana · horarios · importar foto
│   │   │   ├── tareas/
│   │   │   ├── habitos/       incluye fe (oración, Biblia, iglesia)
│   │   │   ├── finanzas/      incluye ahorros
│   │   │   ├── aprendizaje/
│   │   │   ├── metas/
│   │   │   └── perfil/        módulos · avisos · IA · tema
│   │   ├── api/               webhooks · trabajos asíncronos de visión
│   │   └── manifest.ts
│   │
│   ├── modules/               tasks · schedules · habits · finance ·
│   │   └── <módulo>/          learning · goals
│   │       ├── components/    UI del módulo
│   │       ├── actions.ts     'use server'
│   │       ├── queries.ts     lecturas
│   │       ├── schemas.ts     Zod
│   │       └── types.ts
│   │
│   ├── core/
│   │   ├── time/              día lógico, ciclos, conversión de zona
│   │   ├── availability/      bloques ocupados → huecos libres
│   │   ├── scoring/           motor de prioridad
│   │   ├── planner/           propuesta de día
│   │   ├── streaks/           rachas de hábitos
│   │   ├── finance/           ahorro, proyecciones
│   │   ├── redaction/         tipos Redacted* y constructor de contexto
│   │   └── ai/                cliente, prompts versionados, esquemas, límites
│   │
│   ├── components/{ui,shared}/
│   ├── lib/{supabase,env,format}/
│   └── types/database.ts      generado; no editar
│
├── supabase/
│   ├── migrations/            SQL versionado, incluye políticas RLS
│   ├── functions/             Edge Functions (envío de notificaciones)
│   └── tests/                 tests de RLS con dos usuarios
└── docs/
```

---

## 6. Autenticación y autorización

**Autenticación.** Supabase Auth con `@supabase/ssr`; sesión en **cookies httpOnly**, no
en `localStorage` (inmune a robo por XSS). MVP: correo con contraseña y enlace mágico;
Google OAuth más adelante. El middleware refresca el token en cada navegación.

En el servidor se usa **siempre `auth.getUser()`**, que valida contra Supabase, nunca
`getSession()`, cuyo contenido es manipulable.

**Autorización, tres anillos.** RLS en Postgres es la autoridad final: aunque un fallo del
servidor pidiera datos ajenos, la base los niega. Las Server Actions validan con Zod y
**nunca aceptan un `user_id` que venga del cliente**. La interfaz solo oculta; jamás es un
control de seguridad.

**Detalles que deciden si el RLS funciona de verdad:**

- Escribir `(select auth.uid()) = user_id`, no `auth.uid() = user_id`. Sin el subselect,
  la política se reevalúa fila por fila y la consulta se vuelve un orden de magnitud más
  lenta. Es *la* optimización de RLS en Supabase.
- Índice en `user_id` en todas las tablas y en toda clave foránea (Postgres no los crea).
- Toda vista en el esquema público con `security_invoker = true`: sin eso, una vista
  **ignora el RLS** de sus tablas y queda expuesta por PostgREST.
- Un test en integración continua que falle si existe cualquier tabla pública sin RLS
  activado. Con veinte tablas, olvidarlo una vez es cuestión de tiempo.
- La clave `service_role` no aparece en el runtime de la aplicación: solo en migraciones y
  en el borrado de cuenta, aislada en un archivo con `import 'server-only'`. Nunca para
  saltarse una política que estorba.

**Creación de perfil.** Un trigger `on_auth_user_created` inserta la fila en `profiles`,
con `security definer set search_path = ''` y **lista blanca** de campos: los metadatos
del registro los controla quien se registra, y copiarlos en bloque es escalada de
privilegios.

---

## 7. Modelo de tiempo

El punto más delicado del producto. Si el cálculo del tiempo libre está mal, **todas** las
recomendaciones están mal y el usuario no sabrá por qué. La versión 1 tenía aquí tres
errores; esta es la corrección.

### 7.1 Ciclos medidos en días, no en semanas

```
schedules
  cycle_length_days   -- 14 = semana A/B · 7 = horario fijo · 5, 21… = turnos rotativos
  anchor_date         -- día en que empieza el desplazamiento 0
  valid_from / valid_to -- al cambiar de semestre o de trabajo no se pierde el histórico
  kind, priority

schedule_items
  day_offset          -- 0 .. cycle_length_days-1
  start_time          -- hora local
  duration_minutes    -- NO end_time
  kind                -- class | work | rest | sleep | commute
  blocks_availability
```

Medir el ciclo **en días** es lo que permite representar turnos 4x3, 6x2 o 7x7, que no se
alinean con la semana. Un horario semanal normal es `cycle_length_days = 7`: el mismo
código, sin ramas especiales.

```ts
dayOffset = mod(civilDaysBetween(anchorDate, fecha), cycleLengthDays)
// mod euclídeo: ((n % m) + m) % m
```

Tres correcciones respecto a la versión 1, todas con consecuencias reales:

- **Módulo euclídeo.** Con el `%` de JavaScript, cualquier fecha anterior al ancla da un
  índice negativo y **ninguna** fila coincide: el horario aparece vacío sin error.
- **Días civiles, no semanas ni instantes.** `startOfWeek` depende del idioma configurado
  y la diferencia en semanas sobre instantes se rompe con el cambio de hora.
- **`duration_minutes` en vez de `end_time`.** Un turno de 22:00 a 06:00 con hora de fin
  es ambiguo y una consulta "del día de hoy" pierde el derrame. El resolvedor consulta el
  día D **y el D−1**.

### 7.2 Excepciones

```
schedule_exceptions(schedule_id, schedule_item_id NULL,
                    exception_date, type: cancel|move|add,
                    start_time, duration_minutes)
```

Con `schedule_item_id` nulo, la excepción cubre el día entero: feriado, licencia, viaje.

### 7.3 El "día lógico"

`profiles.day_cutoff_hour`, 4:00 por defecto. Sin esto, quien sale del turno a las 22:00 y
ora a la 01:30 ve que "rompió la racha" — un fallo pequeño que destruye la confianza en un
módulo delicado.

Todo registro guarda **`occurred_at timestamptz` y `local_date date` materializado en la
escritura**. Calcular el día lógico con una función no sirve: no es inmutable y por tanto
no se puede indexar.

### 7.4 Disponibilidad

```ts
getFreeSlots({ now, timezone, dayWindow, busyBlocks,
               commuteMinutes, bufferMinutes, minSlotMinutes })
```

Función pura. Reglas que la hacen creíble:

- Las duraciones se calculan por **diferencia de instantes** tras convertir a la zona del
  usuario. Nunca restando horas locales: hay días de 23 y de 25 horas, y en el día en que
  empieza el horario de verano **las 00:00 no existen**.
- **Traslados**: un hueco de 30 minutos entre la universidad y el turno no es tiempo útil.
  Un colchón global es demasiado grosero; el traslado se descuenta por bloque.
- Huecos menores que `minSlotMinutes` (15 por defecto) no se ofrecen.

---

## 8. Recurrencia: tareas y hábitos

La versión 1 proponía "materializar la siguiente instancia al completar la actual". Es
incorrecto y rompe la función más importante de la especificación: una tarea mensual que
nunca se completa **no vuelve a existir jamás**, los hábitos no tienen nada que
materializar los días que fallan, y no se distingue "no lo hice" de "no tocaba".

**Tareas — plantilla e instancias.**

```
task_templates(id, user_id, rrule, dtstart, until, timezone, defaults…)
tasks(template_id, occurrence_date, is_detached, UNIQUE(template_id, occurrence_date))
```

Las instancias se materializan en una **ventana deslizante** de sesenta días mediante
un `upsert` idempotente, disparado tanto por un trabajo programado como perezosamente al
abrir la agenda. La restricción `UNIQUE` hace inocuo ejecutarlo dos veces y elimina la
dependencia dura del cron. Editar la plantilla regenera solo las instancias futuras que
el usuario no haya modificado a mano.

**Hábitos — no se materializan.** Generar 365 filas por hábito y año no aporta nada.

```
habits(schedule_kind, days_of_week[], target_per_period, category)
habit_logs(habit_id, local_date, status: done|skipped|excused, value,
           metadata jsonb, UNIQUE(habit_id, local_date))
```

La racha se deriva de los **días esperados** (función pura de la regla) frente a los días
registrados. `skipped` y `excused` **no rompen la racha**: es lo que evita que la app se
convierta en una máquina de presión, y es también la salida honesta para "hoy trabajé el
domingo". La racha no se considera rota hasta que pasa el corte del día lógico.

`current_streak` y `longest_streak` se cachean en la fila del hábito mediante trigger: el
dashboard no puede recorrer el historial completo en cada carga.

---

## 9. Tareas postergadas

Es la función que la especificación marca como más importante, y el diseño correcto es
menos obvio de lo que parece.

**`postponed` no es un estado.** Es un predicado derivado del tiempo. Modelarlo como
estado obliga a mutar filas cada noche con un trabajo programado que, si falla una vez,
pierde un dato irrecuperable. Los estados son `pending`, `in_progress`, `done`,
`cancelled`.

**Separar dos fechas que la versión 1 confundía:** `scheduled_date` (cuándo pienso
hacerla) y `due_date` (cuándo vence). Postergar mueve la primera; la segunda no se toca.

```
postponed_days = today_local − COALESCE(scheduled_date, due_date)
```

Calculado al vuelo: cero infraestructura, imposible de desincronizar.

**Y además un registro de hechos**, porque la historia no se puede inventar
retroactivamente y es lo que permitirá más adelante aprender qué tipo de tareas se
posterga el usuario:

```
task_events(task_id, kind, from_date, to_date, occurred_at,
            source: user|system|ai)
```

Se escribe solo en acciones explícitas: reprogramar, pulsar "ahora no", cambiar la fecha.
Se añaden `original_scheduled_date` y `reschedule_count` a la tarea.

**El tope y la salida.** A partir del quinto día la prioridad **deja de subir** y la tarea
entra en un flujo de revisión con tres salidas: reprogramar, dividir o soltar. Soltar no
es un fracaso y el texto debe reflejarlo. Sin este tope, una tarea olvidada acaba
dominando el dashboard para siempre y el usuario deja de mirarlo.

---

## 10. Motor de priorización

En `core/scoring/`. Simple, transparente y extensible: el usuario debe poder entender por
qué algo subió.

**Filtros duros** (antes de puntuar): módulo desactivado · completada o cancelada ·
dependencia sin resolver · fecha de inicio futura · no cabe en el hueco y no es divisible.

Cuando `estimated_minutes` es nulo —el caso normal— se asume **30 minutos** en vez de
descartar la tarea. La versión 1 no definía esto y el filtro habría eliminado casi todo.

**Fórmula**, cada factor normalizado entre 0 y 1:

```
score = 100 × ( 0.28·urgencia + 0.24·importancia + 0.14·postergación
              + 0.14·encaje  + 0.10·alineación  + 0.10·contexto )
```

| Factor | Cálculo |
|---|---|
| urgencia | vencida 1.0 · hoy 0.95 · mañana 0.8 · 2-3 días 0.6 · 4-7 0.4 · más 0.2 · sin fecha 0.15 |
| importancia | prioridad declarada: 0.25 / 0.5 / 0.75 / 1.0 |
| postergación | `min(1, días/5)` — satura a los cinco días (§9) |
| encaje | holgado 1.0 · justo 0.8 · divisible 0.6 · no cabe → filtrado |
| alineación | vinculada a una meta activa 1.0 · sin vincular 0.3 |
| contexto | **coste de cambio de contexto**: penaliza proponer 25 minutos de estudio profundo veinte minutos antes de un turno de ocho horas. Más adelante, coincidencia con la franja en que el usuario suele completar esa categoría |

Los pesos viven en `profiles.scoring_weights` **guardando solo las diferencias** respecto
al valor por defecto, que vive en el código. Guardar el objeto completo significa que al
añadir un factor las filas antiguas quedan sin peso, y que un usuario puede dejar los
pesos sin sumar 1 y sacar el score fuera de rango.

**La explicación es una plantilla**, no texto generado: se toman los dos factores que más
aportaron y se rellena una frase. Determinista, instantánea, gratis y con el tono bajo
control. Un modelo de lenguaje aquí añade latencia, variabilidad y riesgo de producir
copy culposo.

**Guardarraíles.** Máximo tres focos importantes al día. Con menos de quince minutos
libres no se propone una tarea. Nunca se invade un bloque de descanso o sueño. El texto
jamás menciona rachas rotas ni incumplimientos como reproche.

---

## 11. Notificaciones

Avisos contextuales: antes de una clase o turno, y a la hora de un hábito.

**La limitación que hay que conocer antes de invertir aquí:** en iPhone, las
notificaciones web solo funcionan si la app está **instalada en la pantalla de inicio**
(iOS 16.4+), y el permiso debe pedirse desde un gesto del usuario. En Safari sin instalar
no hay notificaciones, y no hay forma de evitarlo sin una app nativa. El onboarding tiene
que explicar el paso de instalación, porque de él depende que la función exista.

**Arquitectura.** Web Push con VAPID, sin proveedor externo. El programador **no** puede
vivir en el cron de Vercel: el plan gratuito permite una ejecución al día. Va en
**pg_cron dentro de Supabase**, cada cinco minutos, llamando a una Edge Function.

```
push_subscriptions(user_id, endpoint, keys, user_agent, created_at)
notifications_sent(user_id, kind, ref_id, local_date, sent_at)
notification_settings  → en profiles: horario silencioso, tope diario, qué avisar
```

`notifications_sent` evita el fallo clásico de enviar el mismo aviso tres veces cuando el
programador se solapa o reintenta.

**Reglas de tono, no negociables:** horario silencioso, tope diario, y ningún aviso que
mencione rachas rotas o incumplimientos. Un recordatorio dice "es tu momento de oración",
nunca "llevas dos días sin orar".

---

## 12. Navegación

**Móvil — cinco destinos.** Cinco es el máximo antes de que el objetivo táctil sea
incómodo en un iPhone.

```
┌───────────────────────────────────────────────┐
│              contenido de la ruta             │
│                    ( ✨ )   ← ¿Qué hago ahora?│
├───────────────────────────────────────────────┤
│  Hoy    Agenda    Tareas     Vida     Perfil  │
└───────────────────────────────────────────────┘
```

**Vida** es un hub con los módulos activos: hábitos y fe · finanzas · aprendizaje · metas.
Un usuario con pocos módulos ve una lista corta. El **botón flotante ✨** está siempre a
un toque porque es la función central del producto.

**Desktop:** el mismo árbol se vuelve barra lateral con los módulos expandidos (sin el
nivel "Vida") y las pantallas pasan a dos o tres columnas. Sin rutas distintas ni segundo
código de navegación.

**Rutas profundas** (`/tareas/[id]`) se abren como panel deslizante en móvil y lateral en
desktop, conservando URL propia.

---

## 13. Dashboard

Objetivo: que en **una pantalla sin scroll** el usuario sepa qué hacer. Todo lo demás va
bajo el pliegue.

```
╭───────────────────────────────────────────────╮
│  Buenos días, [nombre]              [avatar]  │
│  Jueves 20 de agosto                          │
├───────────────────────────────────────────────┤
│   TU SIGUIENTE MEJOR ACCIÓN                   │
│                                               │
│   📚  Estudiar programación · 45 min          │
│                                               │
│   Es importante, tienes tiempo hasta las      │
│   13:40 y llevas 2 días postergándolo.        │
│                                               │
│   [ Empezar ]          [ Ahora no ]           │
├───────────────────────────────────────────────┤
│  MI DÍA                              ver todo │
│  ▸ 09:00  Libre — 2h 40min                    │
│  ● 14:00  Trabajo                    8h       │
│  ○ 22:30  Oración                             │
├───────────────────────────────────────────────┤
│  PROGRESO                                     │
│  [🔥 Hábitos 4/6] [💰 Ahorro 32%] [🎯 Metas]  │
╰───────────────────────────────────────────────╯
```

**Una sola recomendación, no una lista.** Una lista de cinco prioridades devuelve la
decisión al usuario, que es justo el problema que la app resuelve.

**"Ahora no" ofrece cuatro motivos de un toque**: *no tengo tiempo* (falla el cálculo de
disponibilidad) · *no me apetece* · *ya no aplica* (datos sucios) · *urge otra cosa*
(pesos mal calibrados). Ese motivo es la señal más valiosa del sistema: convierte un
rechazo en calibración.

**El tiempo libre se muestra como un ítem más.** Ver "2h 40min libres" es lo que hace que
la recomendación parezca razonable en vez de arbitraria.

**"Empezar" registra tiempo real** en `focus_sessions`. De ahí sale el progreso de
aprendizaje y de metas sin que nadie teclee un porcentaje.

Máximo tres tarjetas de progreso visibles. Estados vacíos con acción, nunca pantalla en
blanco. Esqueletos de carga de la misma altura que el contenido final, para que no salte
el layout.

**Rendimiento — no es un detalle.** Esta pantalla necesita perfil, tareas de hoy,
vencidas, hábitos con sus registros, horario resuelto con excepciones y día anterior,
eventos, metas, agregados del mes y la recomendación. Con consultas separadas son unas
diez idas y vueltas. Se resuelve con **una función `get_dashboard(p_date)` en Postgres**
que devuelve un único JSON. Es la pantalla que se abre diez veces al día.

`/ahora` muestra lo mismo con contexto explícito ("tienes 35 minutos hasta tu turno"),
tres alternativas y un selector de "tengo 15 / 30 / 60 minutos" para corregir la
suposición de tiempo disponible.

---

## 14. Modelo de datos

El detalle completo con SQL irá en `DATABASE.md` tras la aprobación. Aquí, las decisiones
estructurales.

**Tablas.** `profiles` · `task_templates` · `tasks` · `task_events` · `task_categories` ·
`events` · `schedules` · `schedule_items` · `schedule_exceptions` · `schedule_imports` ·
`schedule_import_items` · `habits` · `habit_logs` · `church_settings` ·
`financial_accounts` · `transactions` · `financial_categories` · `savings_goals` ·
`savings_goal_contributions` · `net_worth_snapshots` · `learning_resources` ·
`learning_progress` · `goals` · `goal_links` · `focus_sessions` · `ai_recommendations` ·
`ai_calls` · `push_subscriptions` · `notifications_sent`.

Frente a la lista original: **fuera** `spiritual_activities` y `church_events` (absorbidas
por los hábitos), y fuera `shared_spaces` / `shared_space_members` / `shared_items`.
**Dentro**, las que faltaban para que las funciones pedidas fueran calculables.

**Decisiones de tipos.**

- **Dinero**: `amount_minor bigint` y `currency char(3)` **congelada en la fila** — cambiar
  la moneda del perfil no debe reescribir la historia. Nunca `numeric` (llega a
  JavaScript como texto y acaba en `parseFloat`) y nunca coma flotante.
- **Una sola tabla `transactions`** con `kind` (`income` | `expense` | `transfer`) y
  `amount_minor` siempre positivo. Sin signos mezclados.
- **`financial_accounts`, `savings_goal_contributions` y `net_worth_snapshots`** existen
  porque sin ellas *no se pueden calcular* el dinero disponible, el patrimonio ni la
  proyección de "lo alcanzarás en X meses" que la especificación pide.
- **Estados**: enum de Postgres para dominios cortos y estables (estado de tarea,
  prioridad, tipo de transacción), porque genera tipos de unión en TypeScript
  automáticamente. **Tabla** para todo lo que el usuario configura: las categorías nunca
  son enum ni texto libre.
- **`learning_resources.progress_percent`** es una caché mantenida por trigger desde
  `learning_progress`. Escribirla desde dos sitios garantiza que diverjan.
- **`deleted_at`** (borrado suave) donde la IA puede proponer cambios, para que "nunca
  borra sin confirmación" sea además reversible.
- **`goal_links`**: sin polimorfismo. Columnas con clave foránea real y arco exclusivo
  (`CHECK (num_nonnulls(...) = 1)`). Son cuatro tipos enlazables, no cuarenta.
- **`enabled_modules`** con dominio comprobado: un error de tipeo no puede desactivar un
  módulo en silencio.
- `user_id` con el mismo nombre y tipo en todas las tablas, con clave foránea a
  `auth.users ON DELETE CASCADE` — sin eso no se puede borrar una cuenta. `updated_at` por
  trigger en todas.

**Índices que importan desde el principio:** `tasks(user_id, scheduled_date)` parcial
sobre pendientes · `tasks(user_id, due_date)` parcial · `habit_logs(habit_id, local_date)`
único · `schedule_items(schedule_id, day_offset)` ·
`transactions(user_id, occurred_on DESC)` · toda clave foránea. Los agregados se hacen por
rango de fechas, nunca con `date_trunc` sobre la columna, que anula el índice.

---

## 15. Arquitectura de IA

Cinco niveles, de más a menos determinista. `AI.md` desarrollará prompts y esquemas.

| Nivel | Qué hace | Modelo |
|---|---|---|
| L0 | Recomendación, disponibilidad, rachas, proyecciones | **Ninguno.** Milisegundos, gratis, explicable |
| L1 | Lenguaje natural → tarea o evento estructurado | Modelo pequeño y rápido |
| L2 | Propuesta de plan del día | Modelo intermedio, **precomputado de madrugada**, nunca un spinner en la pantalla principal |
| L3 | Foto de horario → estructura | Modelo grande con visión, dos pasadas |
| L4 | Resumen semanal | Modelo intermedio, por lotes |

**Dónde el modelo aporta de verdad:** la visión de horarios (ningún OCR clásico entiende
celdas fusionadas más semana A/B) y la entrada en lenguaje natural, que es lo que hace que
la app no se sienta un formulario. **Dónde es decoración cara:** reescribir la explicación
del porqué, un chat genérico (que además es la vía natural de fuga de datos financieros y
personales) y las recomendaciones de aprendizaje abiertas, que inventan cursos que no
existen — para eso sirve el grafo de prerrequisitos que la propia especificación describe.

### 15.1 Fotos de horarios

El requisito más difícil, y donde conviene ser honesto sobre lo que se puede esperar.

1. **Filtro barato primero**: rechazar imágenes pequeñas; el primer campo de la respuesta
   es "¿esto es un horario?", para cortar antes de la pasada cara.
2. **Recorte y rotación en el cliente**, nada más. Lo que sí ayuda: enviar la imagen
   completa **más dos a cuatro cuadrantes con solape**; subir la resolución de una sola
   imagen no aporta porque se reescala.
3. **Transcribir antes de interpretar**: primero el texto literal de cada celda, después
   los elementos normalizados anclados a su celda. Prohibición explícita de completar
   patrones: rellenar el viernes vacío porque de lunes a jueves hay clase es el fallo más
   frecuente.
4. **Dos pasadas independientes.** La confianza que el modelo declara está mal calibrada;
   la señal fuerte es el **acuerdo entre pasadas**. Coinciden → verde. Discrepan → ámbar,
   mostrando ambas opciones como dos botones, lo que convierte "¿está bien?" en "elige".
5. **Validadores deterministas, sin modelo**: fin posterior al inicio salvo turno
   nocturno; sin solapes; duración entre 15 minutos y 14 horas; minutos múltiplos de cinco
   (`1400` → `14:00` es un error típico); número de columnas de la cabecera frente a días
   emitidos. Los días se resuelven **por posición de columna**, jamás por interpretación
   semántica de "L M M J V".
6. **Confirmación como calendario semanal**, no como JSON ni formulario. No se puede
   guardar con elementos en rojo; los ámbar se guardan marcados para revisar. **No hay
   botón de "aceptar todo"** mientras haya ámbar. Todo con un identificador de importación
   para **revertir la importación completa** de un toque.
7. **Fallos parciales**: se importa lo bueno y se reintenta solo la región dudosa, que es
   una llamada barata y más precisa.

**Expectativa realista** (celdas correctas / importaciones sin editar nada): captura de
pantalla limpia 92–97% / 70–80% · foto de papel bien encuadrada 85–93% / 40–55% · foto
torcida y tabla densa 65–85% / 15–30% · **manuscrito: no prometerlo**. Las horas salen
bien; las siglas de asignatura son lo peor. Con seis clases y 90% por celda hay
aproximadamente **un error por importación**: por eso la pantalla de confirmación no es un
accesorio, es la función.

### 15.2 Guardarraíles y costos

Índices locales en vez de identificadores (R2) · triple validación: salida estructurada →
Zod → invariantes de dominio, y **un plan que viole las invariantes se rechaza entero, no
se repara** · sin ruta a borrado o modificación, garantizado en la base de datos y no solo
en el código · deshacer transaccional durante 24 horas · versión de prompt y modelo
guardados con cada propuesta · tabla `ai_calls` con tokens, costo y latencia, sin el
contenido · tope duro por usuario y día · interruptor `AI_ENABLED=false` que degrada al
motor determinista sin desplegar.

Coste estimado con uso realista: **menos de un dólar por usuario y mes**, dominado por la
importación de horarios. El uso diario cuesta cero porque no pasa por un modelo.

---

## 16. MVP y orden de construcción

Cada fase termina con algo usable. **Regla: usar cada módulo una semana real antes de
empezar el siguiente.** Es la única defensa contra construir diez módulos que no se usan.

| Fase | Contenido | Estimación |
|---|---|---|
| **0** | Migraciones, RLS con test en CI, auth, perfil, sistema de diseño, PWA instalable con service worker | 1–2 sem |
| **1** | `core/time` y `availability`; horarios rotativos manuales; agenda día y semana | 2–4 sem |
| **2** | Tareas: plantillas, instancias, postergación, `task_events` | 2 sem |
| **3** | **Motor de prioridad + `/hoy` + `/ahora`.** ← *aquí el producto existe: úsalo dos semanas antes de seguir* | 1–2 sem |
| **4** | Notificaciones (Web Push, pg_cron, ajustes de tono) | 1 sem |
| **5** | Hábitos y fe, con la regla condicional de iglesia | 1–2 sem |
| **6** | Finanzas y ahorros | 1–2 sem |
| **7** | Aprendizaje, apoyado en `focus_sessions` | 1 sem |
| **8** | Metas y enlaces | 1 sem |
| **9** | IA: entrada en lenguaje natural → plan del día → **fotos de horarios** | 2–4 sem |

**El MVP es la fase 3.** Auth, perfil, horarios, tareas con postergación, dashboard y
"¿qué hago ahora?" — sin IA, sin finanzas, sin hábitos. En ese punto la app ya responde la
pregunta central, que es lo único que la define.

**Fuera del MVP, explícitamente:** IA en cualquier forma, los cuatro módulos, compartición,
offline con cola de escrituras, importación de Google Calendar, gráficos.

**Una excepción al orden:** conviene probar la visión con **tres o cuatro fotos reales**
en la fase 0, con un script de línea de comandos. Es una tarde y dice si esa función es
viable tal como la imaginas, antes de diseñar la pantalla de confirmación que la sostiene.

Alcance completo: **cuatro a siete meses** de trabajo sostenido. El multiplicador oculto es
el nivel de acabado exigido por módulo (estados vacíos, de carga, de error, responsive):
duplica el coste de cada pantalla.

---

## 17. Riesgos técnicos

| # | Riesgo | Por qué duele | Mitigación |
|---|---|---|---|
| 1 | **Modelo de tiempo incorrecto** (ciclos, medianoche, cambio de hora, día lógico) | Produce resultados *silenciosamente* falsos; toda recomendación depende de él y el usuario deja de confiar sin saber por qué | §7 + tests de tabla con fechas de cambio de hora, turnos nocturnos y fechas anteriores al ancla |
| 2 | **Abandono por fricción** | El riesgo real del proyecto no es técnico | R4: valor con datos incompletos, registro derivado del uso, notificaciones, módulos en serie |
| 3 | **Horario mal importado y aceptado sin mirar** | Toda recomendación posterior es basura | Confianza por celda, ámbar sin premarcar, sin "aceptar todo", revertir importación |
| 4 | **Rendimiento del dashboard** | Es la pantalla que se abre diez veces al día | RPC único, `(select auth.uid())`, índices, misma región |
| 5 | **Fuga de datos sensibles al modelo** | Finanzas y vida espiritual; casi siempre por una función de chat añadida tarde | Tipos `Redacted*` + test de fuga en CI + opt-in |
| 6 | **RLS olvidado en una tabla nueva** | Con veinte tablas ocurrirá | Test en CI que falla si alguna tabla pública no tiene RLS |
| 7 | **Calibración eterna del motor** | Se puede tocar pesos para siempre sin mejorar | Conjunto dorado de treinta situaciones + métrica de aceptación |
| 8 | **Agotamiento en el módulo cuatro o cinco** | El patrón conocido en proyectos personales de este tamaño | Fases cortas que terminan en algo usable; el MVP en la fase 3 |
| 9 | **Notificaciones que no llegan en iPhone** | Depende de que la app esté instalada | Onboarding que lo explica; sin ese paso, la función no existe |
| 10 | **Costo de IA disparado** | Reintentos sin tope o un chat abierto | `ai_calls`, tope duro, interruptor global |

---

## 18. Dependencias

Criterio: entra una dependencia solo si resuelve un problema que ya tenemos.

**Fase 0–3 (MVP).** `next` · `react` · `typescript` · `tailwindcss` v4 ·
`@supabase/supabase-js` · `@supabase/ssr` · `zod` · `date-fns` + `@date-fns/tz` ·
`lucide-react` · `clsx` · `tailwind-merge` · `class-variance-authority` · `@radix-ui/*`
(los instala shadcn/ui bajo demanda) · `@serwist/next` · `vitest`.

**Fase 4.** `web-push` (envío con VAPID desde la Edge Function).

**Fase 9.** `@anthropic-ai/sdk` (solo servidor) · `sharp` (reencodificar imágenes: además
de normalizar, **borra los metadatos EXIF**, y una foto de un horario lleva
geolocalización).

**Según haga falta.** `recharts`, solo si un SVG propio no basta para finanzas ·
`@tanstack/react-query`, reevaluado en la fase de agenda · `rrule`, solo si aparece una
recurrencia que el modelo de ciclos no cubra.

**Explícitamente fuera:** `redux` · `zustand` · un ORM sobre Supabase (duplicaría la fuente
de verdad y complicaría el RLS) · `moment` · `lodash` · `framer-motion` · librerías de
calendario pesadas, que pesan más que nuestra propia vista y son difíciles de hacer
nativas en móvil.

---

## 19. Qué necesito de ti para empezar

1. **Un día real tuyo, hora a hora**, incluyendo un día de turno y un domingo.
2. **Confirmar la fusión de fe con hábitos** (§2).
3. **Tres o cuatro fotos reales de horarios**, para la prueba de viabilidad de la fase 0.
4. **Aprobar el orden de construcción** (§16) y el alcance del MVP.
