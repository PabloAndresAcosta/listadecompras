# HANDOFF — Grocery Guard (Lista de Compras)

> **Instrucción para nueva sesión**: Lee este archivo primero. Luego lee los artefactos
> referenciados antes de ejecutar cualquier comando.

---

## Repositorio

**URL**: https://github.com/PabloAndresAcosta/listadecompras
**Directorio local**: `C:\Users\pablo\Documents\proyectos\listadecompras`

---

## Estado Actual del Proyecto

**Fecha de corte**: 2026-03-24
**Fase completada**: Plan ✅
**Próxima fase**: Tasks

### Fases completadas

| Fase | Comando | Branch | PR | Estado |
|------|---------|--------|----|--------|
| Constitución | `/speckit.constitution` | `feature/constitution` | #1 | ✅ Mergeado |
| Especificación | `/speckit.specify` | `feature/specify` | #3 | ✅ Mergeado |
| Clarificaciones | `/speckit.clarify` (x2) | `feature/specify` | #3 | ✅ Mergeado |
| Plan técnico | `/speckit.plan` | `feature/plan` | #5 | ✅ Mergeado |

---

## Próximo Comando

```
/speckit.tasks
```

**Instrucciones**:
- Crear branch `feature/tasks` desde `develop` antes de ejecutar
- El comando debe leer `plan/plan.md` como fuente principal
- Generar `tasks.md` con tareas atómicas ordenadas por dependencia

---

## Artefactos Clave (leer en este orden)

```
1. .specify/memory/constitution.md
   → Principios del proyecto: KISS, Contrato Sagrado, Diagnóstico sin Fricción

2. .specify/features/001-shopping-list-lifecycle/spec.md
   → Qué construimos: 4 HUs, modelo de dominio, reglas de negocio, criterios de éxito

3. .specify/features/001-shopping-list-lifecycle/overview.md
   → Vista rápida con diagramas Mermaid: casos de uso, estados, flujo, dependencias HUs

4. .specify/features/001-shopping-list-lifecycle/plan/plan.md
   → Cómo lo construimos: 4 fases, roadmap Gantt, gates de calidad

5. .specify/features/001-shopping-list-lifecycle/plan/research.md
   → Decisiones técnicas: Java Spring Boot, MongoDB, React + Vite, Onion Architecture

6. .specify/features/001-shopping-list-lifecycle/plan/data-model.md
   → Entidades MongoDB, índices, estrategia de clonación

7. .specify/features/001-shopping-list-lifecycle/plan/contracts/api-contracts.md
   → 9 endpoints REST con request/response/error codes
```

---

## Resumen Técnico (para no leer todo de nuevo)

| Decisión | Elección |
|----------|----------|
| Arquitectura | Onion (Domain → Application → Infrastructure → Presentation) |
| Backend | Java 21 + Spring Boot 3.x |
| Base de datos | MongoDB — colecciones `listas` e `items` separadas |
| Frontend | React 18 + Vite + TanStack Query |
| Logging | SLF4J + Logback, JSON estructurado con MDC traceId |
| Testing | JUnit 5 + Mockito · Domain ≥90% · Application ≥80% |
| SLA | 100% endpoints < 1000ms · 10 usuarios concurrentes |
| Seguridad | Fuera de alcance en v1.0 |

---

## Historias de Usuario (Feature 001)

| ID | Título | Archivo |
|----|--------|---------|
| HU-01 | Creación de Lista desde Cero | `stories/HU-01-crear-lista.md` |
| HU-02 | Clonación Basada en Historial | `stories/HU-02-clonar-historial.md` |
| HU-03 | Gestión del Ciclo de Vida (Estados) | `stories/HU-03-ciclo-de-vida.md` |
| HU-04 | Gestión de Ítems (Productos) | `stories/HU-04-gestionar-items.md` |

**Orden de implementación sugerido**: HU-01 → HU-04 → HU-03 → HU-02

---

## Reglas de Negocio Críticas (recordatorio rápido)

| ID | Regla |
|----|-------|
| BR-03 | No se puede completar lista sin ítems → Error 400 |
| BR-07 | Nombre de ítem único por lista → Error 409 |
| BR-08 | Ítems solo gestionables en `EN_PREPARACION` → Error 422 |
| BR-04 | El backend asigna todas las fechas; el frontend nunca las envía |

---

## Convención de Branches

```
main        ← producción estable
develop     ← integración
feature/*   ← una rama por fase del workflow speckit
```

**Próximo branch a crear**: `feature/tasks` desde `develop`
