# HU-03: Gestión del Ciclo de Vida (Estados)

**Feature**: Gestión de Listas de Compra (001)
**Estado**: Listo para desarrollo
**Última actualización**: 2026-03-24

---

## Historia de Usuario

> **Como** usuario en el supermercado,
> **quiero** marcar una lista como terminada y poder reabrirla si es necesario,
> **para** mantener el historial limpio y poder corregir errores.

---

## Contexto para el Desarrollador

Esta historia gestiona las **transiciones de estado** del ciclo de vida de una lista. Existen dos operaciones:

1. **Completar** (`EN_PREPARACION` → `COMPLETADA`): el usuario terminó de comprar.
2. **Reactivar** (`COMPLETADA` → `EN_PREPARACION`): el usuario necesita editar o retomar una compra ya cerrada.

Ambas transiciones son controladas por el sistema con validaciones de estado previas. El campo `fecha_procesado` actúa como testigo de la operación: se puebla al completar y se borra al reactivar.

> ⚠️ **"Reactivada" no es un estado del enum** — es el resultado de la transición `COMPLETADA → EN_PREPARACION`. El enum tiene exactamente dos valores: `EN_PREPARACION` y `COMPLETADA`.

**Actor principal**: Comprador (uso en supermercado)

---

## Criterios de Aceptación

### Completar Lista

| # | Criterio | Resultado Esperado |
|---|----------|--------------------|
| CA-1 | El usuario completa una lista `EN_PREPARACION` con ≥1 ítem | `estado = COMPLETADA`, `fecha_procesado` con timestamp actual |
| CA-2 | El usuario intenta completar una lista sin ítems | Error 400: "No se puede completar una lista vacía" |
| CA-3 | El usuario intenta completar una lista ya `COMPLETADA` | El sistema rechaza con error de transición inválida |
| CA-4 | El usuario inspecciona una lista completada | `fecha_procesado` refleja el momento exacto de la transición |

### Reactivar Lista

| # | Criterio | Resultado Esperado |
|---|----------|--------------------|
| CA-5 | El usuario reactiva una lista `COMPLETADA` | `estado = EN_PREPARACION`, `fecha_procesado = null` |
| CA-6 | El usuario intenta reactivar una lista ya `EN_PREPARACION` | El sistema rechaza con error de transición inválida |
| CA-7 | El usuario inspecciona la lista reactivada | `fecha_procesado` es `null` (no el valor anterior) |

---

## Requisitos Funcionales Relacionados

- **FR-03** — El sistema verifica que la lista tenga al menos un ítem antes de permitir la transición a `COMPLETADA`.
- **FR-03** — Si la lista está vacía, el sistema responde: *"No se puede completar una lista vacía"*.
- **FR-03** — Si la validación pasa, el sistema actualiza `estado = COMPLETADA` y `fecha_procesado` al timestamp actual.
- **FR-04** — El sistema acepta el `id` de una lista en estado `COMPLETADA` para reactivar.
- **FR-04** — El sistema actualiza `estado = EN_PREPARACION` y `fecha_procesado = null`.
- **FR-04** — Si la lista no está en estado `COMPLETADA`, el sistema rechaza la operación.
- **FR-05** — `fecha_procesado` es asignado exclusivamente por el sistema; el cliente no lo envía.

---

## Reglas de Negocio Aplicables

| ID | Regla |
|----|-------|
| BR-03 | No se puede completar una lista sin ítems. Violación → Error 400. |
| BR-04 | El sistema asigna `fecha_procesado`. El frontend no envía fechas. |
| BR-06 | Al reactivar, `fecha_procesado` se limpia a `null`. |

---

## Máquina de Estados

```
                    ┌──────────────────────────────────────┐
                    │                                      │
                    ▼                                      │
            EN_PREPARACION                             COMPLETADA
                    │                                      │
                    │  [completar]                         │
                    │  Guard: items.length >= 1            │
                    │  Effect: fecha_procesado = NOW()     │
                    └──────────────────────────────────────►
                                                           │
                                                           │  [reactivar]
                                                           │  Effect: fecha_procesado = null
                                                           │
                    ◄──────────────────────────────────────┘
```

### Tabla de Transiciones Válidas

| Estado Actual | Operación | Estado Resultante | `fecha_procesado` |
|---------------|-----------|-------------------|-------------------|
| `EN_PREPARACION` | completar (con ítems) | `COMPLETADA` | timestamp actual |
| `EN_PREPARACION` | completar (sin ítems) | ❌ Error 400 | sin cambio |
| `COMPLETADA` | reactivar | `EN_PREPARACION` | `null` |
| `COMPLETADA` | completar | ❌ Error de transición | sin cambio |
| `EN_PREPARACION` | reactivar | ❌ Error de transición | sin cambio |

---

## Modelo de Datos Involucrado

```
Operación Completar: PATCH /listas/{id}/completar
Operación Reactivar: PATCH /listas/{id}/reactivar

Estado antes de completar:
  Lista { estado: EN_PREPARACION, items: [≥1 item], fecha_procesado: null }

Estado después de completar:
  Lista { estado: COMPLETADA, fecha_procesado: "2026-03-24T15:30:00Z" }

Estado después de reactivar:
  Lista { estado: EN_PREPARACION, fecha_procesado: null }
```

---

## Escenarios de Prueba

### ✅ Escenario 1 — Completar lista con ítems
**Dado** que existe la lista `id=ABC` con `estado = EN_PREPARACION` y 5 ítems
**Cuando** el usuario solicita completarla
**Entonces**:
  - `estado = COMPLETADA`
  - `fecha_procesado` tiene el timestamp de la operación (±2 segundos)

---

### ❌ Escenario 2 — Completar lista vacía
**Dado** que existe la lista `id=ABC` con `estado = EN_PREPARACION` y 0 ítems
**Cuando** el usuario solicita completarla
**Entonces** el sistema responde con error `400 Bad Request` y el mensaje *"No se puede completar una lista vacía"*.
**Y** el estado de la lista permanece `EN_PREPARACION`.

---

### ✅ Escenario 3 — Reactivar lista completada
**Dado** que existe la lista `id=ABC` con `estado = COMPLETADA` y `fecha_procesado` poblada
**Cuando** el usuario solicita reactivarla
**Entonces**:
  - `estado = EN_PREPARACION`
  - `fecha_procesado = null`

---

### ❌ Escenario 4 — Reactivar lista ya en preparación
**Dado** que existe la lista `id=ABC` con `estado = EN_PREPARACION`
**Cuando** el usuario solicita reactivarla
**Entonces** el sistema responde con error de transición inválida.

---

### ✅ Escenario 5 — Ciclo completo de vida
**Dado** que existe la lista `id=ABC` con `estado = EN_PREPARACION` y 3 ítems
**Cuando** el usuario la completa → luego la reactiva → luego la completa de nuevo
**Entonces** cada transición es válida, `fecha_procesado` refleja la última completación y se limpia correctamente en la reactivación.

---

## Criterio de Éxito Asociado

> **SC-04**: El sistema informa correctamente todos los errores de negocio (lista vacía, estado incorrecto).
> *Métrica*: 100% de los errores definidos retornan mensajes comprensibles al usuario.

> **SC-05**: El ciclo de vida completo (crear → completar → clonar → completar) puede ejecutarse sin errores por un usuario nuevo.
> *Métrica*: Tasa de éxito ≥ 95% en pruebas con usuarios.

---

## Notas de Implementación (Guía, no mandato)

- Se recomiendan endpoints dedicados por transición (`PATCH /listas/{id}/completar`, `PATCH /listas/{id}/reactivar`) para mantener la semántica clara vs. un genérico `PATCH /listas/{id}` con campo `estado`.
- Las transiciones inválidas (ej. completar una lista ya completada) deben retornar `409 Conflict` o `422 Unprocessable Entity` con mensaje descriptivo.
- `fecha_procesado` nunca debe ser enviada por el cliente; debe ser rechazada si viene en el payload.
- El log del sistema debe registrar cada transición de estado con el `id` de la lista y el timestamp.

---

## Dependencias

- **HU-01** — La lista debe existir (creada via HU-01) para poder transicionar su estado.
- **HU-02** — Las listas `COMPLETADAS` son las candidatas para clonación (HU-02 depende de que HU-03 pueda completar listas).

## Historias Relacionadas

- [HU-01 — Creación de Lista desde Cero](./HU-01-crear-lista.md)
- [HU-02 — Clonación Basada en Historial](./HU-02-clonar-historial.md)
