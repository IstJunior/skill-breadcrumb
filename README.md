# breadcrumb — Skill de compresión de contexto para Claude Code

> **Carga el entendimiento completo de tu proyecto en ~300 tokens en lugar de ~40,000.**

Un skill de Claude Code que reemplaza la lectura del codebase completo al inicio de cada sesión por una instantánea estructurada y siempre actualizada de la arquitectura, decisiones técnicas y tareas pendientes de tu proyecto.

---

## El problema que resuelve

En cada sesión nueva, Claude re-lee el codebase desde cero.

En un proyecto real, eso cuesta **40,000–60,000 tokens antes de escribir una sola línea de código**. El 95% de esas lecturas son redundantes — el código no cambió.

La causa raíz: Claude no tiene "memoria" de lo que es tu proyecto. Solo tiene lo que lee en la ventana de contexto actual.

**breadcrumb** resuelve esto sin requerir infraestructura especial — solo dos archivos markdown y un skill.

---

## Cómo funciona

El sistema tiene tres componentes:

| Componente | Ubicación | Propósito |
|-----------|----------|---------|
| `system-map.md` | `.claude/context/system-map.md` | Mapa de arquitectura comprimido (~400–600 tokens) |
| `session-log.md` | `.claude/context/session-log.md` | Últimas 5 sesiones: qué cambió y **por qué** |
| `breadcrumb.md` | `~/.claude/skills/breadcrumb.md` | Instrucciones que le dicen a Claude cómo usar lo anterior |

En lugar de leer 50 archivos, Claude lee 2 archivos y un git diff. Entiende el mismo codebase con 98% menos tokens.

---

## Flujo técnico

### Inicio de sesión — `/breadcrumb load`

```
USUARIO EJECUTA: /breadcrumb
                 │
                 ▼
┌─────────────────────────────────────────────────────────┐
│  Claude lee .claude/context/system-map.md               │
│  (~400–600 tokens)                                      │
│  → Arquitectura completa del proyecto                   │
│  → Índice de archivos con descripciones en una línea    │
│  → Features activas y su estado                         │
│  → Bugs conocidos y decisiones no obvias                │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│  Claude lee .claude/context/session-log.md              │
│  (~100–200 tokens)                                      │
│  → Qué pasó en la última sesión                         │
│  → Qué está pendiente ahora mismo                       │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│  Compara el hash git del system-map con el HEAD actual  │
│  git diff <hash_guardado> HEAD --name-only              │
│                                                         │
│  Sin cambios? ────────────────────────────────────────▶ LISTO
│                                                         │
│  ¿Archivos cambiados?                                   │
│    └─▶ Lee SOLO los archivos modificados                │
│         (omite: lockfiles, assets, auto-generados)      │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│  ✅ Contexto cargado. Claude reporta:                   │
│                                                         │
│  📁 Proyecto: MiApp                                     │
│  🔧 Stack: Next.js 14 | Supabase | Vercel | Tailwind    │
│  📋 Última sesión: 2026-03-25 — auditoría SEO           │
│  🔄 Archivos cambiados: 2 → app/layout.tsx, lib/i18n.ts │
│  📌 Pendiente: fix /contact, crear llms.txt             │
└─────────────────────────────────────────────────────────┘

Tokens totales usados: 300–800 (vs. 40,000+ sin breadcrumb)
```

---

### Fin de sesión — `/breadcrumb update`

```
USUARIO EJECUTA: /breadcrumb update
                 │
                 ▼
┌─────────────────────────────────────────────────────────┐
│  Claude identifica los cambios de la sesión:            │
│  • Archivos tocados esta sesión                         │
│  • Decisiones tomadas (y POR QUÉ)                       │
│  • Tareas completadas                                   │
│  • Qué queda pendiente                                  │
└────────────────────┬────────────────────────────────────┘
                     │
           ┌─────────┴──────────┐
           ▼                    ▼
┌──────────────────┐  ┌──────────────────────────────────┐
│  system-map.md   │  │  session-log.md                  │
│                  │  │                                  │
│  Edita SOLO las  │  │  Agrega nueva entrada al inicio: │
│  líneas que      │  │                                  │
│  cambiaron:      │  │  ## 2026-03-27 — [título]        │
│  • FILE INDEX    │  │  Done: [tareas]                  │
│  • FEATURES      │  │  Files: [lista]                  │
│  • BUGS          │  │  Decisions: [POR QUÉ] ← más val. │
│  • PENDING       │  │  Next: [próximos pasos]          │
│  • hash header   │  │                                  │
│                  │  │  Conserva solo las últimas 5     │
│  Máx 800 tokens  │  │                                  │
└──────────────────┘  └──────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────┐
│  ✅ Breadcrumb actualizado                              │
│  📝 system-map.md — 3 líneas modificadas                │
│  📓 session-log.md — nueva entrada agregada             │
└─────────────────────────────────────────────────────────┘

Costo: ~200 tokens
```

---

### Configuración inicial — `/breadcrumb init`

```
USUARIO EJECUTA: /breadcrumb init  (solo una vez por proyecto)
                 │
                 ▼
┌─────────────────────────────────────────────────────────┐
│  Explora la estructura del codebase                     │
│  • ls, find (filtrado), package.json / go.mod           │
│  • git log, hash HEAD actual                            │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│  Lee archivos clave de arquitectura (selectivamente)    │
│  ✅ Lee: entry points, config, auth, tipos, schema      │
│  ❌ Omite: tests, lockfiles, assets, archivos generados │
└────────────────────┬────────────────────────────────────┘
                     │
           ┌─────────┴──────────┐
           ▼                    ▼
┌──────────────────────┐  ┌──────────────────────────────┐
│  Genera              │  │  Genera                      │
│  system-map.md       │  │  session-log.md              │
│                      │  │                              │
│  STACK               │  │  Entrada inicial con el      │
│  ARCHITECTURE        │  │  estado actual del proyecto  │
│  FILE INDEX          │  │  y lo que fue explorado      │
│  ACTIVE FEATURES     │  │                              │
│  KNOWN BUGS          │  │                              │
│  PENDING             │  │                              │
└──────────┬───────────┘  └──────────────┬───────────────┘
           │                             │
           └─────────────┬───────────────┘
                         ▼
              Escribe en .claude/context/
              (crea el directorio si no existe)
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│  ✅ Breadcrumb inicializado                             │
│  📁 .claude/context/system-map.md                       │
│  📓 .claude/context/session-log.md                      │
│                                                         │
│  A partir de ahora:                                     │
│    /breadcrumb        → inicio de sesión                │
│    /breadcrumb update → fin de sesión                   │
└─────────────────────────────────────────────────────────┘
```

---

## Ahorro de tokens

| Escenario | Sin breadcrumb | Con breadcrumb | Ahorro |
|----------|--------------------|-----------------|---------|
| Sesión típica (0–2 archivos cambiaron) | ~40,000 | 300–800 | **98%** |
| Feature nueva (10 archivos) | ~40,000 | 5,000–12,000 | **75%** |
| Solo preguntas / consultas | ~40,000 | 300 | **99%** |
| Refactor masivo (todo cambia) | ~40,000 | ~40,000 | 0% *(caso raro)* |

> El ahorro del 50% es el escenario **conservador**. La mayoría de sesiones ahorra 90–98%.

---

## Instalación

### 1. Copiar el archivo del skill

```bash
mkdir -p ~/.claude/skills
cp breadcrumb.md ~/.claude/skills/breadcrumb.md
```

### 2. Inicializar en tu proyecto

Navega a la raíz de tu proyecto y ejecuta:

```
/breadcrumb init
```

Claude explorará tu codebase y generará:
```
tu-proyecto/
└── .claude/
    └── context/
        ├── system-map.md   ← instantánea de arquitectura
        └── session-log.md  ← historial de sesiones
```

### 3. Úsalo

```
/breadcrumb          → carga contexto (ejecutar al inicio de cada sesión)
/breadcrumb update   → guarda el progreso (ejecutar al final de la sesión)
```

---

## Formato del `system-map.md`

```markdown
# System Map — MiApp
> Last updated: 2026-03-27 | git: a3f8c12

## STACK
Next.js 14 App Router | Supabase (PostgreSQL + Auth + Realtime) | Vercel | Tailwind + shadcn

## ARCHITECTURE
PUBLIC (SSR/indexed): /, /about, /pricing, /blog/*
AUTH: /login, /register, /forgot-password
PRIVATE (CSR/auth-gated): /dashboard/*, /settings
MIDDLEWARE: middleware.ts → redirige /dashboard/* no autenticado a /login

## FILE INDEX
app/dashboard/layout.tsx  → layout raíz de todas las páginas del dashboard; carga profile + period context
lib/supabase/client.ts    → cliente Supabase para el browser; singleton via useRef para evitar re-init
hooks/useProfile.ts       → obtiene perfil de usuario + preferencia de moneda; usado en cada página del dashboard
lib/utils/amortization.ts → cálculo de amortización francesa; estimatePaidInstallments() estima cuotas pagadas desde el saldo

## ACTIVE FEATURES
Módulo deudas: ✅ listo | columna insurance_rate_mv agregada (migración 074)
Cuotas tarjeta crédito: 🚧 en progreso | feature de 8 fases, fases 1–2 completas
AI Insights: ✅ listo | por módulo, lazy-loaded, Supabase edge function

## KNOWN BUGS / DECISIONS
- debt.total_paid no es confiable: usar estimatePaidInstallments() × cuota_base (puede desfasarse si pagos se registran sin reducir saldo)
- Simulador crédito restringido a super_admin: aún en pruebas, no listo para usuarios generales
- CLI de Supabase no configurado: migraciones se aplican manualmente en el dashboard

## PENDING
- Aplicar migración 075 (schema cuotas tarjeta crédito)
- Actualizar tasa seguro del crédito de Dairon a 0.12417
```

**Reglas del system-map:**
- **Máx 800 tokens** — comprimir sin piedad
- **Una línea por archivo** en FILE INDEX
- **Cero prosa** — solo hechos
- **La sección KNOWN BUGS / DECISIONS** es la más valiosa: captura el POR QUÉ de decisiones que no son visibles en el código

---

## Formato del `session-log.md`

```markdown
# Session Log

## 2026-03-27 — seguro de deudores en créditos
Done: columna insurance_rate_mv, checkbox en DebtDialog, total_paid derivado de tabla de amortización
Files: types/database.ts, components/modules/debts/DebtDialog.tsx, app/dashboard/debts/page.tsx
Decisions: se deriva total_paid de estimatePaidInstallments() porque debt.total_paid se desfasa cuando los pagos se registran manualmente sin actualizar el saldo
Next: actualizar tasa seguro del crédito Dairon a 0.12417, aplicar migración manualmente en Supabase
---

## 2026-03-25 — auditoría SEO completa
Done: auditoría completa con 7 agentes en paralelo, informe guardado en docs/seo-audit-2026-03-25.md
Files: docs/seo-audit-2026-03-25.md
Decisions: robots.ts no está bloqueando crawlers de IA — el bloqueo es a nivel Cloudflare, no en el código. Fix es en el dashboard de CF, no en el repo.
Next: fix /contact (server component), crear llms.txt, agregar userScalable=false a layout.tsx
---
```

---

## Por qué funciona mejor que tomar notas normales

| Propiedad | breadcrumb | Notas sueltas |
|----------|-----------|-------------|
| Estructurado y parseable | ✅ Claude sabe exactamente dónde buscar cada cosa | ❌ Requiere interpretación |
| Captura el POR QUÉ | ✅ Sección Decisions explícita | ❌ Generalmente solo describe el qué |
| Actualizaciones incrementales | ✅ Edita solo las líneas que cambiaron | ❌ Generalmente se reescribe todo |
| Usa git como fuente de verdad | ✅ El diff detecta exactamente qué cambió | ❌ Seguimiento manual |
| Vive con el código | ✅ `.claude/context/` está en el repo | ❌ Externo o separado |
| Sobrevive handoffs de equipo | ✅ Se versiona con git | ❌ Personal y efímero |

---

## Complementario al sistema de memoria de Claude

breadcrumb y el sistema de memoria integrado de Claude tienen propósitos distintos:

| Sistema | Qué guarda | Cuándo se carga |
|--------|--------|--------|
| `~/.claude/projects/.../memory/` | Preferencias del usuario, feedback, estilo de colaboración | Automáticamente, cada sesión |
| `.claude/context/` (breadcrumb) | Estado del codebase, índice de archivos, decisiones técnicas | A demanda via `/breadcrumb` |

Son **complementarios, no competidores**. Memory sabe *cómo trabajar contigo*. breadcrumb sabe *qué es el código*.

---

## `.claude/context/` — Decisión de control de versiones

| Opción | Cuándo usarla |
|--------|-------------|
| **Versionar con git** | Proyectos en equipo — breadcrumb se convierte en conocimiento compartido |
| **Agregar a `.gitignore`** | Proyectos personales o sensibles — se mantiene local |

Si lo versionas, cada miembro del equipo obtiene el contexto del proyecto gratis. Los nuevos integrantes pueden ejecutar `/breadcrumb load` el primer día y entender la arquitectura de inmediato.

---

## Licencia

MIT — úsalo libremente en proyectos personales o comerciales.

---

*Construido para [Claude Code](https://claude.ai/code). Diseñado para que cada sesión se sienta como si nunca te hubieras ido.*
