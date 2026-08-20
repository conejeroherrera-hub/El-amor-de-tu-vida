# Life OS — Arquitectura

> Documento de **diseño**, no de implementación. Nada de lo descrito aquí está construido
> todavía. Su objetivo es que puedas revisar y aprobar las decisiones antes de programar.

Índice:

1. [Principio rector](#1-principio-rector)
2. [Contradicciones detectadas y cómo se resuelven](#2-contradicciones-detectadas-y-cómo-se-resuelven)
3. [Stack definitivo recomendado](#3-stack-definitivo-recomendado)
4. [Arquitectura general](#4-arquitectura-general)
5. [Estructura de carpetas](#5-estructura-de-carpetas)
6. [Autenticación y autorización](#6-autenticación-y-autorización)
7. [Modelo de tiempo: horarios, eventos y disponibilidad](#7-modelo-de-tiempo-horarios-eventos-y-disponibilidad)
8. [Navegación](#8-navegación)
9. [Diseño conceptual del Dashboard](#9-diseño-conceptual-del-dashboard)
10. [Sistema de priorización](#10-sistema-de-priorización)
11. [Módulos configurables](#11-módulos-configurables)
12. [Espacios compartidos](#12-espacios-compartidos)
13. [Sistema de diseño](#13-sistema-de-diseño)
14. [Dependencias necesarias](#14-dependencias-necesarias)

Documentos hermanos: [`DATABASE.md`](./DATABASE.md) · [`AI.md`](./AI.md) ·
[`SECURITY.md`](./SECURITY.md) · [`ROADMAP.md`](./ROADMAP.md) (incluye MVP y riesgos).

---

## 1. Principio rector

Toda decisión técnica de este documento se subordina a una sola pregunta de producto:

> **¿Qué importa realmente hoy?**

De ahí salen tres reglas de arquitectura que atraviesan todo el sistema:

**R1 — El motor de decisión es determinista; la IA es una capa opcional encima.**
Calcular "qué hacer ahora" no requiere un LLM: requiere saber qué tiempo libre hay y qué
tarea tiene mayor score. Si el LLM falla, está caído, o el usuario lo desactiva, la app
sigue respondiendo la pregunta central. La IA agrega lenguaje natural, planificación
compleja y visión (fotos de horarios), no la lógica base.

**R2 — La IA propone, el usuario confirma, la base de datos registra.**
Ninguna salida de IA escribe directamente en tablas de dominio. Escribe en tablas de
propuestas (`ai_recommendations`, `schedule_imports`) y solo una acción explícita del
usuario la promueve a dato real.

**R3 — Privado por defecto.** Row Level Security en cada tabla. Compartir es una acción
explícita, por ítem o por espacio, nunca un default ni un efecto colateral.

---

## 2. Contradicciones detectadas y cómo se resuelven

El brief es coherente en lo grande, pero hay 14 puntos donde dos requisitos chocan o
donde falta una definición sin la cual no se puede modelar. Los resuelvo aquí con una
recomendación; cualquiera es reversible si prefieres lo contrario.

| # | Tensión | Resolución propuesta |
|---|---------|----------------------|
| 1 | "Cada usuario accede **únicamente** a sus datos" (§21) vs. **espacios compartidos** (§17) | RLS con dos caminos: `owner_id = auth.uid()` **O** el registro está publicado en un espacio del que el usuario es miembro (`shared_items`). Sin publicación explícita, nadie más lo ve. Compartir nunca da permiso de borrado al otro usuario. |
| 2 | "Planificar mi día automáticamente" (§15) vs. "la IA nunca crea eventos sin confirmación" (§18) | El plan generado es un **borrador** (`ai_recommendations.kind='day_plan'`, estado `proposed`). Se ve como propuesta editable; "Aceptar plan" es lo que crea eventos. |
| 3 | Dashboard con 6 tarjetas fijas (§3) vs. módulos configurables (§2) | El dashboard se **compone** de las tarjetas de los módulos activos. Un usuario con 3 módulos ve 3 tarjetas, no 6 vacías. Orden configurable. |
| 4 | Horarios rotativos A/B (§4) vs. calendario de eventos simple | No basta RRULE. Modelo de **ciclo**: `schedules.cycle_length_weeks` + `schedule_items.week_index`. Semana A = índice 0, semana B = índice 1. Soporta ciclos de 3+ semanas y turnos. |
| 5 | Subir tareas postergadas de prioridad (§6) vs. "no castigar / no llenar el día" (§1, §13) | La escalada por postergación tiene **techo** (satura a los 5 días) y a partir del día 5 la app deja de subirla y ofrece tres salidas: reprogramar, dividir o soltar. Nunca lenguaje culposo. |
| 6 | "La app aprende qué sueles postergar" (§6) vs. "no compleja al inicio" (§7, §25) | Fase 1: registro de postergaciones (`task_postponements`). Fase 3: estadísticas agregadas por categoría/hora (sin ML). ML real: fuera de roadmap por ahora. La tabla de log se crea desde el inicio porque **los datos históricos no se pueden inventar después**. |
| 7 | "Voy a la iglesia los domingos **si no trabajo**" (§12) con horario rotativo | Regla condicional evaluada contra la resolución de horarios del día. Requiere que el trabajo esté modelado como `schedule` (no como texto libre). Se implementa como regla declarativa en `church_settings`, no como código especial. |
| 8 | Metas financieras (§9) vs. metas generales (§14) | Dos tablas: `savings_goals` (con moneda, monto objetivo, proyección) y `goals` (cualitativas). Una `savings_goal` puede **vincularse** a una `goal` vía `goal_links`. Evita meter montos en un modelo genérico. |
| 9 | Moneda no especificada; ejemplos en pesos chilenos | Default `CLP`, configurable por perfil. Montos como `NUMERIC(14,2)` (exacto, nunca float). **Multi-moneda con conversión: fuera del MVP** — una moneda por perfil. |
| 10 | Zonas horarias no mencionadas, pero rachas/"hoy"/recordatorios dependen de ellas | `profiles.timezone` (IANA). Todo instante se guarda en `timestamptz` (UTC); todo "día" del usuario (racha de hábito, `due_date`) se guarda como `date`/`time` **local** y se resuelve con la timezone del perfil. Mezclar ambos es la fuente #1 de bugs en apps de hábitos. |
| 11 | "Rápida" + PWA (§19, §20) vs. datos siempre en Supabase | MVP **online-first**: shell de la app cacheado, datos desde el servidor. Offline real (cola de escrituras, resolución de conflictos) es un proyecto en sí mismo → Fase 5, solo si lo necesitas. |
| 12 | `learning_progress` como tabla (§21) vs. progreso como campo | Ambos: `learning_resources.progress_percent` es el estado actual (lectura barata), `learning_progress` es el **log de sesiones** (minutos, fecha) que alimenta estadísticas y el motor de prioridad. |
| 13 | IA en todo (§18) vs. costo, latencia y privacidad de datos financieros | La IA es **opt-in por usuario** y por operación. Al LLM nunca se le envían montos crudos ni transacciones: solo agregados y porcentajes. Ver [`AI.md`](./AI.md) §5. |
| 14 | "Un módulo de repetición de tareas" (§5) sin definir si genera instancias | Tarea recurrente = plantilla (`tasks.recurrence`) que **materializa la siguiente instancia al completar** la actual. No se pre-generan 365 filas. La postergación se mide sobre la instancia, no sobre la plantilla. |

---

## 3. Stack definitivo recomendado

Confirmo tu propuesta con versiones y tres precisiones importantes.

| Capa | Elección | Por qué |
|------|----------|---------|
| Framework | **Next.js 15+, App Router** | Server Components reducen JS enviado al móvil; Server Actions eliminan la necesidad de escribir una API REST paralela. |
| Lenguaje | **TypeScript** en modo `strict` | Tipos generados desde el esquema de Supabase → el schema de la BD es la fuente de verdad. |
| Estilos | **Tailwind CSS v4** | Config en CSS, sin `tailwind.config.js` pesado. |
| Componentes | **shadcn/ui** (Radix + Tailwind) | Copias el código a tu repo: cero dependencia opaca, accesibilidad resuelta, personalizable al 100%. |
| Backend | **Supabase** (Postgres + Auth + Storage + RLS) | Auth y autorización a nivel de fila sin escribir backend. |
| Base de datos | **PostgreSQL** con RLS | Ver [`DATABASE.md`](./DATABASE.md). |
| Validación | **Zod** | Un solo esquema valida el formulario, la Server Action y la salida del LLM. |
| Fechas | **date-fns** + **@date-fns/tz** | Ligero y tree-shakeable. |
| IA | **Anthropic API** (`@anthropic-ai/sdk`), solo server-side | Vision para fotos de horarios + generación de texto. Ver [`AI.md`](./AI.md). |
| Hosting | **Vercel** | Integración natural con Next.js; el runtime nunca ve la `service_role` key. |
| PWA | Manifest + iconos desde Fase 1; **Serwist** (service worker) en Fase 4 | Instalable en iPhone desde el principio; caché avanzada cuando haga falta. |

### Tres precisiones sobre el stack

**a) Sin librería de estado global, y sin TanStack Query en el MVP.**
Con Server Components + Server Actions + `revalidatePath`, el estado del servidor es el
estado. Para feedback instantáneo (marcar tarea completada) se usa `useOptimistic` de
React. Introducir Zustand/Redux/React Query ahora sería resolver un problema que no
tenemos. Si en Fase 4 la vista de calendario necesita caché cliente agresiva, se
reevalúa entonces.

**b) Server Actions en lugar de rutas API.**
Se crean Route Handlers solo para lo que realmente lo requiere: webhooks, cron jobs
(Vercel Cron) y subida/streaming de archivos grandes.

**c) La animación empieza con CSS.**
Tailwind + `@keyframes` + `view-transition` cubren el 90% de las "animaciones sutiles"
que pides. `motion` (Framer Motion) solo si aparece una interacción concreta que lo
justifique — es ~30 KB que el móvil paga en cada carga.

---

## 4. Arquitectura general

### 4.1 Vista de capas

```
┌──────────────────────────────────────────────────────────────┐
│  CLIENTE (navegador / PWA instalada)                         │
│  React Server Components + islas de cliente                  │
│  Sin claves. Sin lógica de autorización.                     │
└───────────────┬──────────────────────────────────────────────┘
                │ Server Actions (RPC tipado) / Route Handlers
┌───────────────▼──────────────────────────────────────────────┐
│  SERVIDOR (Next.js en Vercel)                                │
│                                                              │
│  ① Capa de aplicación   Server Actions + validación Zod      │
│  ② Capa de dominio      Motores puros, sin I/O:              │
│      · availability     (bloques libres del día)             │
│      · scoring          (prioridad de tareas)                │
│      · planner          (propuesta de día)                   │
│      · finance          (ahorro, proyecciones)               │
│      · streaks          (rachas de hábitos)                  │
│  ③ Capa de datos        Cliente Supabase con sesión de usuario│
│  ④ Capa de IA           Anthropic SDK, solo aquí             │
└───────────────┬──────────────────────────────────────────────┘
                │ PostgREST con JWT del usuario
┌───────────────▼──────────────────────────────────────────────┐
│  SUPABASE                                                    │
│  Postgres + Row Level Security  ← autoridad final de acceso  │
│  Auth (sesiones en cookies httpOnly)                         │
│  Storage (bucket privado: fotos de horarios)                 │
└──────────────────────────────────────────────────────────────┘
```

**La capa ② es el corazón del producto y es 100% testeable sin base de datos**: funciones
puras que reciben datos y devuelven decisiones. Es lo primero que se debe construir bien,
porque de ahí sale la respuesta a "¿qué debería hacer ahora?".

### 4.2 Organización por módulos, no por tipo de archivo

Cada módulo de vida (tareas, finanzas, hábitos…) vive en su propia carpeta con su UI,
sus acciones, sus consultas y sus esquemas juntos. Razón: los módulos se activan y
desactivan por usuario y se construyen uno por uno en el roadmap; tenerlos aislados
permite avanzar sin romper lo anterior y borrar un módulo entero si sobra.

Regla de dependencias (se verifica en revisión de código):

```
app/  ──►  modules/*  ──►  core/*  ──►  lib/*
                │
                └─► un módulo NO importa de otro módulo.
                    Lo que comparten sube a core/.
```

`core/` contiene los motores de dominio y los tipos que varios módulos necesitan (por
ejemplo: el planificador necesita tareas, hábitos y horarios; no importa esos módulos —
recibe sus datos como argumentos tipados).

---

## 5. Estructura de carpetas

```
life-os/
├── src/
│   ├── app/
│   │   ├── (marketing)/                 # landing pública (Fase 4)
│   │   ├── (auth)/
│   │   │   ├── login/page.tsx
│   │   │   ├── registro/page.tsx
│   │   │   ├── recuperar/page.tsx
│   │   │   └── auth/callback/route.ts   # intercambio de código OAuth
│   │   ├── (app)/                       # área privada; layout verifica sesión
│   │   │   ├── layout.tsx               # nav inferior (móvil) / sidebar (desktop)
│   │   │   ├── hoy/page.tsx             # Dashboard — pantalla principal
│   │   │   ├── ahora/page.tsx           # "¿Qué debería hacer ahora?"
│   │   │   ├── agenda/
│   │   │   │   ├── page.tsx             # día / semana
│   │   │   │   └── horarios/            # crear, editar, importar por foto
│   │   │   ├── tareas/
│   │   │   ├── finanzas/                # incluye /ahorros
│   │   │   ├── aprendizaje/
│   │   │   ├── fe/                      # oración, Biblia, iglesia
│   │   │   ├── habitos/
│   │   │   ├── metas/
│   │   │   ├── compartido/
│   │   │   └── perfil/                  # datos, módulos, preferencias, IA
│   │   ├── manifest.ts                  # PWA
│   │   └── layout.tsx                   # tema, fuentes, providers
│   │
│   ├── modules/
│   │   ├── tasks/
│   │   │   ├── components/              # TaskCard, TaskForm, TaskList…
│   │   │   ├── actions.ts               # 'use server'
│   │   │   ├── queries.ts               # lecturas (server-only)
│   │   │   ├── schemas.ts               # Zod
│   │   │   └── types.ts
│   │   ├── schedules/ finance/ learning/ faith/ habits/ goals/ sharing/
│   │   └── …                            # misma estructura interna
│   │
│   ├── core/
│   │   ├── availability/                # bloques ocupados → huecos libres
│   │   ├── scoring/                     # motor de prioridad (§10)
│   │   ├── planner/                     # propuesta de día
│   │   ├── ai/
│   │   │   ├── client.ts                # Anthropic SDK (server-only)
│   │   │   ├── prompts/                 # prompts versionados
│   │   │   ├── schemas/                 # Zod para validar salidas del LLM
│   │   │   └── guardrails.ts            # límites, redacción de datos, costos
│   │   └── time/                        # timezone, "hoy" del usuario, ciclos
│   │
│   ├── components/
│   │   ├── ui/                          # shadcn/ui
│   │   └── shared/                      # EmptyState, ErrorState, PageHeader…
│   │
│   ├── lib/
│   │   ├── supabase/{client,server,middleware}.ts
│   │   ├── env.ts                       # validación de env vars con Zod
│   │   └── format.ts                    # dinero, fechas, duraciones
│   │
│   └── types/database.ts                # generado desde Supabase (no editar)
│
├── supabase/
│   ├── migrations/                      # SQL versionado, incluye políticas RLS
│   └── seed.sql                         # categorías por defecto
│
├── docs/                                # ARCHITECTURE, DATABASE, AI, SECURITY, ROADMAP
└── public/icons/                        # iconos PWA
```

---

## 6. Autenticación y autorización

### 6.1 Autenticación

- **Supabase Auth** con `@supabase/ssr`: sesión en **cookies httpOnly**, no en
  `localStorage` (inmune a robo por XSS).
- MVP: **email + contraseña** y **magic link**. Google OAuth en Fase 4 (añadirlo después
  no rompe cuentas existentes si el email coincide).
- `middleware.ts` refresca el token en cada navegación y redirige a `/login` sin sesión.
- En servidor **siempre `supabase.auth.getUser()`** (valida contra Supabase), nunca
  `getSession()` para decisiones de acceso: el contenido de la cookie es manipulable.

### 6.2 Autorización — tres anillos

1. **RLS en Postgres (autoridad final).** Aunque un bug del servidor pidiera datos
   ajenos, la base los niega. Cada tabla con RLS activado y política explícita.
2. **Server Actions.** Toda acción empieza por resolver el usuario y validar el input
   con Zod. Nunca se confía en un `user_id` que venga del cliente.
3. **UI.** Solo oculta; jamás es un control de seguridad.

### 6.3 Registro y creación de perfil

`auth.users` es de Supabase y no se toca. Un **trigger** `on_auth_user_created` inserta
la fila correspondiente en `profiles`. Así nunca existe un usuario sin perfil.

El onboarding pregunta lo mínimo: nombre, zona horaria, moneda y **qué módulos usa**.
Sin suposiciones sobre estudio o trabajo (§2 del brief).

---

## 7. Modelo de tiempo: horarios, eventos y disponibilidad

Este es el punto técnicamente más delicado del producto: si el cálculo del tiempo libre
está mal, todas las recomendaciones están mal.

### 7.1 Tres fuentes de "ocupación"

| Fuente | Tabla | Naturaleza |
|--------|-------|------------|
| Horario recurrente | `schedules` + `schedule_items` | Patrón cíclico (clases, turnos) |
| Excepción | `schedule_exceptions` | "Este jueves cambié turno / no hay clase" |
| Evento puntual | `events` | Cita médica, culto, reunión |

### 7.2 Ciclos rotativos

```
schedules
  cycle_length_weeks  = 2         -- semana A y semana B
  anchor_date         = 2026-01-05 -- lunes en que empieza el índice 0
  timezone            = America/Santiago

schedule_items
  week_index  0 | 1               -- 0 = semana A, 1 = semana B
  weekday     0..6                -- 0 = lunes
  start_time / end_time           -- hora local
```

El índice de semana para una fecha se calcula así:

```ts
weekIndex = floor(diffInWeeks(startOfWeek(fecha), anchorDate)) % cycleLengthWeeks
```

Un ciclo de 1 semana es un horario fijo normal: **el mismo modelo cubre ambos casos**, no
hay dos rutas de código. "Miércoles libre" simplemente no tiene fila.

### 7.3 Cálculo de disponibilidad (`core/availability`)

Función pura, sin base de datos:

```ts
getFreeSlots({
  date, timezone,
  dayWindow,        // p.ej. 07:00–23:00, del perfil
  busyBlocks,       // horarios resueltos + excepciones + eventos
  bufferMinutes,    // 10 min de colchón antes/después (por defecto)
  minSlotMinutes,   // ignora huecos < 15 min: no son tiempo útil
}): FreeSlot[]
```

Salida: lista de huecos con inicio, fin, duración y etiqueta de momento del día
(mañana/tarde/noche). Es el insumo de **"¿qué debería hacer ahora?"**, del planificador
y de la regla de iglesia.

Que sea pura significa que se puede probar con decenas de casos borde (turno que cruza
medianoche, cambio de horario de verano, evento que solapa una clase) sin levantar nada.

---

## 8. Navegación

**Móvil (diseño primario) — barra inferior de 5 destinos.** Cinco es el máximo antes de
que los objetivos táctiles sean incómodos en un iPhone.

```
┌───────────────────────────────────────────────┐
│                                               │
│              contenido de la ruta             │
│                                               │
│                    ( ✨ )   ← FAB "¿Qué hago  │
│                              ahora?"          │
├───────────────────────────────────────────────┤
│  Hoy    Agenda    Tareas     Vida     Perfil  │
│  ◉        ▤         ✓         ◇         ○     │
└───────────────────────────────────────────────┘
```

- **Hoy** — Dashboard. Es la pantalla de arranque.
- **Agenda** — día y semana; acceso a horarios e importación por foto.
- **Tareas** — bandeja, filtros, postergadas.
- **Vida** — *hub* con los módulos **activos**: Finanzas · Aprendizaje · Fe · Hábitos ·
  Metas · Compartido. Un usuario con pocos módulos ve una lista corta.
- **Perfil** — cuenta, módulos, preferencias, ajustes de IA, tema.

El **FAB ✨** flota sobre Hoy/Agenda/Tareas y abre `/ahora`. Es la función central del
producto (§16 del brief): merece estar siempre a un toque.

**Desktop.** El mismo árbol se convierte en sidebar fija con los módulos expandidos
(sin nivel "Vida" intermedio) y las pantallas pasan de 1 a 2–3 columnas. No hay rutas
distintas ni un segundo código de navegación.

**Rutas profundas** (`/tareas/[id]`, `/metas/[id]`) se abren como *sheet* deslizante en
móvil y como panel lateral en desktop, conservando URL propia — compartible y con
botón atrás funcional.

---

## 9. Diseño conceptual del Dashboard (`/hoy`)

Objetivo: que en **una pantalla sin scroll** el usuario sepa qué hacer ahora. Todo lo
demás está bajo el pliegue.

```
╭───────────────────────────────────────────────╮
│  Buenos días, Sebastián            [avatar]   │  ← saludo por hora local
│  Jueves 20 de agosto                          │
├───────────────────────────────────────────────┤
│                                               │
│   TU SIGUIENTE MEJOR ACCIÓN                   │  ← 1 sola recomendación
│                                               │
│   📚  Estudiar programación                   │
│       45 minutos                              │
│                                               │
│   Es importante, tienes tiempo libre hasta    │  ← el "porqué", en 1 frase
│   las 13:40 y llevas 2 días postergándolo.    │
│                                               │
│   [ Empezar ]          [ Ahora no ]           │
│                                               │
├───────────────────────────────────────────────┤
│  MI DÍA                              ver todo │
│  ─────────────────────────────────────────    │
│  ▸ 09:00  Libre — 2h 40min                    │  ← huecos visibles: el tiempo
│  ● 14:00  Trabajo                    8h       │    libre es información
│  ○ 22:30  Oración                             │
├───────────────────────────────────────────────┤
│  PROGRESO                                     │
│  ┌───────────┐ ┌───────────┐ ┌───────────┐    │
│  │ 🔥 Hábitos│ │ 💰 Ahorro │ │ 🎯 Metas  │    │  ← solo módulos activos,
│  │   4/6     │ │   32%     │ │  2 activas│    │    scroll horizontal
│  └───────────┘ └───────────┘ └───────────┘    │
╰───────────────────────────────────────────────╯
```

Decisiones de diseño y su razón:

- **Una sola recomendación, no una lista.** Una lista de 5 "prioridades" vuelve a poner
  la decisión sobre el usuario, que es justo el problema que la app resuelve.
- **"Ahora no" sin fricción.** Descarta esa sugerencia y pide la siguiente. Se registra
  el rechazo (`ai_recommendations.feedback`): es la señal más valiosa para calibrar.
- **El tiempo libre se muestra como un ítem más.** Ver "2h 40min libres" es lo que hace
  que la recomendación se sienta razonable en vez de arbitraria.
- **Máximo 3 tarjetas de progreso visibles**, resto con scroll horizontal. §19: no
  sobrecargar.
- **Estados vacíos con acción**, nunca una pantalla en blanco: sin tareas → "Tu día está
  despejado. ¿Aprovechas para avanzar en [meta activa]?".
- **Estado de carga con skeletons** de la misma altura que el contenido final (sin saltos
  de layout).
- **Saludo por hora local** del perfil, no del navegador.

Pantalla `/ahora` (**¿Qué debería hacer ahora?**): misma tarjeta pero con contexto
explícito ("Tienes 35 minutos hasta tu turno"), 3 alternativas por si la primera no
encaja, y un selector "tengo 15 / 30 / 60 min" para que el usuario corrija la suposición
de tiempo disponible.

---

## 10. Sistema de priorización

Vive en `core/scoring/`. Primera versión **simple, transparente y extensible** (§7 del
brief). Sin ML, sin cajas negras: el usuario debe poder entender por qué algo subió.

### 10.1 Filtros duros (antes de puntuar)

Se descarta lo que no puede hacerse ahora:

- módulo del ítem desactivado;
- tarea completada, cancelada o programada en otro horario;
- dependencia sin completar (`blocked_by_task_id`);
- no cabe en el hueco disponible **y** no es divisible;
- fecha de inicio futura (`start_date > hoy`).

### 10.2 Fórmula

Cada factor se normaliza a 0–1 y se combina en un score de 0 a 100:

```
score = 100 × ( 0.30·urgencia
              + 0.25·importancia
              + 0.15·postergación
              + 0.15·encaje
              + 0.10·alineación_meta
              + 0.05·contexto )
```

| Factor | Cómo se calcula |
|--------|-----------------|
| **urgencia** | Vencida → 1.0 · hoy → 0.95 · mañana → 0.8 · 2-3 días → 0.6 · 4-7 → 0.4 · >7 → 0.2 · sin fecha → 0.15 |
| **importancia** | Prioridad declarada: baja 0.25 · media 0.5 · alta 0.75 · crítica 1.0 |
| **postergación** | `min(1, días_postergada / 5)` — **satura a los 5 días** (ver §2.5) |
| **encaje** | Cabe con holgura → 1.0 · cabe justo → 0.8 · divisible en el hueco → 0.6 · no cabe → filtrada |
| **alineación_meta** | Vinculada a una meta activa → 1.0 · sin vincular → 0.3 |
| **contexto** | MVP: 0.5 fijo. Fase 3: coincidencia con el momento del día en que el usuario suele completar esa categoría |

Los pesos viven en `profiles.scoring_weights` (JSONB) con estos valores por defecto:
son **datos, no código**, así que calibrarlos no requiere despliegue.

### 10.3 Explicación en lenguaje natural (sin LLM)

Se toman los **2 factores que más aportaron** al score y se rellena una plantilla:

> "Es importante, tienes tiempo disponible y llevas 2 días postergándolo."

Determinista, instantáneo y gratis. El LLM puede pulir esta frase después (opcional),
pero el contenido lo decide el motor.

### 10.4 Guardarraíles anti-agobio

Reglas explícitas, no accidentales:

- Máximo **3 focos importantes** propuestos por día.
- Si el hueco libre es < 15 min: no se recomienda tarea, se sugiere una micro-acción o
  descanso.
- Una tarea postergada 5+ días **deja de escalar** y pasa a un flujo de revisión:
  *reprogramar · dividir · soltar*. Soltar no es un fracaso y el copy lo refleja.
- Nunca se recomienda ocupar un bloque marcado como descanso, sueño o tiempo compartido.
- El lenguaje jamás menciona rachas rotas ni incumplimientos como reproche (§12, §13).

### 10.5 Extensibilidad

`scoreTask()` recibe `(item, context, weights)` y devuelve
`{ score, factors, explanation }`. Añadir un factor = añadir una función pura y un peso.
Cada recomendación mostrada guarda la **versión del algoritmo** que la generó, para poder
comparar calibraciones más adelante.

---

## 11. Módulos configurables

`profiles.enabled_modules text[]` (no una tabla aparte: es una lista corta de flags que
siempre se lee junto al perfil).

```
'tasks' | 'schedule' | 'finance' | 'savings' | 'learning'
| 'faith' | 'habits' | 'goals' | 'sharing'
```

- `tasks` y `schedule` están **siempre activos**: son el sustrato del motor de decisión.
- Desactivar un módulo **oculta**, no borra. Reactivarlo devuelve los datos intactos.
- La comprobación ocurre en tres puntos: navegación (qué se ve), Server Action (qué se
  puede escribir) y motor de scoring (qué entra en las recomendaciones).

---

## 12. Espacios compartidos

Modelo mínimo que cumple "nada se comparte automáticamente" (§17):

```
shared_spaces            un espacio (p. ej. "Nosotros")
shared_space_members     usuario + rol (owner | member) + estado de invitación
shared_items             qué se publicó: (space_id, entity_type, entity_id, permission)
```

- `entity_type` ∈ `goal | task | event | habit | church_event`.
- `permission` ∈ `view | edit`. **Nunca `delete`**: el dueño es el único que borra.
- Compartir es un acto explícito por ítem; no existe "compartir todo el módulo".
- Dejar de compartir = borrar la fila de `shared_items`; el dato original no se toca.
- Las políticas RLS de lectura compartida usan una función `is_space_member()` marcada
  `SECURITY DEFINER` para **evitar recursión infinita de RLS** (error clásico al hacer
  que la política de una tabla consulte otra tabla con RLS).

Invitación por email en Fase 5; el modelo ya lo contempla con `status='pending'`.

---

## 13. Sistema de diseño

Objetivo (§19): *centro de control personal*, no software empresarial.

- **Tokens** en CSS variables (`--bg`, `--surface`, `--text`, `--accent`…), con modo
  claro y oscuro definidos desde el día 1. El modo oscuro **no** es un filtro invertido:
  es una paleta propia.
- **Tipografía**: una sola familia variable (Inter o Geist) con jerarquía por peso y
  tamaño. La jerarquía visual la crea el espacio en blanco, no los bordes.
- **Color**: fondo neutro + **un** color de acento. Gradientes: solo en la tarjeta de
  recomendación, como acento único (§19: no abusar).
- **Tarjetas**: solo cuando agrupan información heterogénea. Las listas son listas, no
  15 tarjetas apiladas.
- **Movimiento**: 150–250 ms, `ease-out`, y respeto estricto a
  `prefers-reduced-motion`.
- **iPhone**: `safe-area-inset` en la nav inferior, objetivos táctiles ≥ 44 px,
  inputs con `font-size ≥ 16px` (evita el zoom automático de iOS), `theme-color` por
  esquema de color.
- **Accesibilidad**: contraste AA, foco visible, navegación por teclado en desktop.

---

## 14. Dependencias necesarias

Criterio (§20, §25): entra una dependencia solo si resuelve un problema que ya tenemos y
que costaría claramente más escribir a mano.

### Fase 0–1 (MVP)

| Paquete | Para qué | Alternativa descartada |
|---------|----------|------------------------|
| `next`, `react`, `react-dom` | Framework | — |
| `typescript`, `@types/*` | Tipos | — |
| `tailwindcss` v4 | Estilos | CSS Modules (más lento de iterar) |
| `@supabase/supabase-js`, `@supabase/ssr` | Datos + auth | Auth propia (semanas de trabajo + riesgo) |
| `zod` | Validación de formularios, acciones y salidas de IA | Validación manual (frágil) |
| `date-fns`, `@date-fns/tz` | Fechas y timezones | `Temporal` (aún no disponible en todos lados); Moment (obsoleto) |
| `lucide-react` | Iconos (viene con shadcn/ui) | — |
| `clsx`, `tailwind-merge`, `class-variance-authority` | Requeridos por shadcn/ui | — |
| `@radix-ui/*` | Primitivas accesibles que instala shadcn/ui bajo demanda | — |

### Fase 2–3 (según módulo)

| Paquete | Para qué | Nota |
|---------|----------|------|
| `@anthropic-ai/sdk` | IA: texto y visión | Solo servidor |
| `recharts` | Gráficos de finanzas | Solo si un SVG propio no basta; se evalúa al llegar |

### Fase 4+

| Paquete | Para qué |
|---------|----------|
| `@serwist/next` | Service worker, caché offline del shell |
| `rrule` | Solo si aparecen recurrencias que el modelo de ciclos no cubra |

### Explícitamente **no** instalamos ahora

`redux` · `zustand` · `@tanstack/react-query` · `prisma`/`drizzle` (Supabase ya tipa el
esquema; un ORM encima duplicaría la fuente de verdad y complicaría RLS) ·
`moment` · `lodash` · `framer-motion` · librerías de calendario pesadas (`FullCalendar`
pesa más que nuestra vista y es difícil de hacer nativa en móvil).

---

## 15. Qué revisar antes de programar

Puntos donde tu decisión cambia el código, en orden de impacto:

1. **Moneda única por perfil** (CLP por defecto) vs. multi-moneda — §2.9.
2. **Online-first** en el MVP, offline real en Fase 5 — §2.11.
3. **Tope de escalada por postergación a los 5 días** y sus tres salidas — §2.5.
4. **Pesos del motor de prioridad** (§10.2): ¿la urgencia debe pesar más que la
   importancia, o al revés en tu caso?
5. **Cinco destinos de navegación** con "Vida" como hub — §8.
6. **IA opt-in** y sin envío de montos crudos — §2.13 y [`AI.md`](./AI.md).
7. El **alcance del MVP** — ver [`ROADMAP.md`](./ROADMAP.md).
