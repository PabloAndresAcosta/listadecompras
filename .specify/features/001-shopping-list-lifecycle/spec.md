# Feature Specification: Gestión de Listas de Compra

**Feature ID**: 001
**Branch**: `feature/001-shopping-list-lifecycle`
**Status**: Draft
**Created**: 2026-03-24
**Last Updated**: 2026-03-24

---

## Clarifications

### Session 2026-03-24

- Q: ¿Se requieren archivos de historia de usuario independientes por HU para uso como referencia de desarrollo? → A: Sí. Se crean 3 archivos `.md` independientes en `stories/`, uno por HU, con criterios de aceptación, requisitos, reglas de negocio, escenarios de prueba y notas de implementación por historia.
- Q: ¿Se requiere un documento central con diagrama de casos de uso en formato Mermaid? → A: Sí. Se crea `overview.md` con diagramas Mermaid (casos de uso, estados, flujo de operaciones) y tabla de navegación a todas las historias.

---

## 1. Vision & Purpose

### Problem Statement
Los usuarios del hogar olvidan ítems en el supermercado o dedican tiempo innecesario recreando listas que ya usaron antes. No existe un mecanismo que les permita reutilizar compras históricas ni seguir el ciclo de vida de una lista desde su preparación hasta su finalización.

### User Value
Optimizar la experiencia de compra mediante la creación ágil de listas y la reutilización de compras históricas, ahorrando tiempo y reduciendo olvidos.

### Business Value
- Reducción del tiempo promedio para crear una lista de compra recurrente.
- Fidelización del usuario al retener su historial de consumo.
- Base de datos de patrones de compra que habilita futuras funcionalidades (sugerencias, estadísticas).

---

## 2. Actors & Stakeholders

| Actor | Rol | Interacción Principal |
|-------|-----|-----------------------|
| Usuario del Hogar | Primario | Crea y gestiona listas desde casa |
| Comprador | Primario | Usa la lista activa en el supermercado |
| Comprador Recurrente | Primario | Clona listas anteriores para nuevas compras |

---

## 3. User Stories

> 📂 **Referencia rápida para desarrolladores**: Cada historia tiene su propio archivo detallado en la carpeta `stories/`:
> - [HU-01 — Creación de Lista desde Cero](./stories/HU-01-crear-lista.md)
> - [HU-02 — Clonación Basada en Historial](./stories/HU-02-clonar-historial.md)
> - [HU-03 — Gestión del Ciclo de Vida](./stories/HU-03-ciclo-de-vida.md)
>
> 🗺️ **Vista global con diagramas Mermaid**: [overview.md](./overview.md)

### HU-01: Creación de Lista desde Cero
**Como** usuario del hogar
**Quiero** crear una nueva lista de compras con nombre y descripción
**Para** organizar los productos que necesito comprar

**Criterios de Aceptación**:
- El sistema solicita: `título` (obligatorio) y `descripción` (opcional).
- La fecha de creación (`fecha_creacion`) se asigna automáticamente por el sistema al momento del registro.
- El estado inicial de toda lista recién creada es `EN_PREPARACION`.
- No es posible crear una lista sin título.

---

### HU-02: Clonación Basada en Historial
**Como** comprador recurrente
**Quiero** seleccionar una lista `COMPLETADA` y clonarla para una nueva compra
**Para** no tener que reescribir los mismos productos cada semana

**Criterios de Aceptación**:
- Solo se pueden clonar listas en estado `COMPLETADA`.
- Al clonar se genera una nueva entidad con un `id` único y una `fecha_creacion` con el timestamp actual.
- Todos los ítems de la lista original se copian a la nueva lista.
- El estado de la nueva lista clonada es `EN_PREPARACION`.
- La lista original permanece intacta e inmutable; no se modifica ni elimina.
- La descripción de la nueva lista es editable por el usuario antes de confirmar la clonación.

---

### HU-03: Gestión del Ciclo de Vida (Estados)
**Como** usuario en el supermercado
**Quiero** marcar una lista como terminada y poder reabrirla si es necesario
**Para** mantener el historial limpio y poder corregir errores

**Criterios de Aceptación**:

**Completar lista (`EN_PREPARACION` → `COMPLETADA`)**:
- El sistema registra automáticamente `fecha_procesado` con el timestamp del momento de la transición.
- No es posible completar una lista que no tenga al menos un ítem. El sistema rechaza la acción con un mensaje descriptivo.

**Reactivar lista (`COMPLETADA` → `EN_PREPARACION`)**:
- La lista vuelve a estado `EN_PREPARACION` y queda disponible para edición.
- El campo `fecha_procesado` se limpia (valor `null`) al reactivar, ya que la compra no se considera finalizada.

---

## 4. Domain Model

### Entidad: Lista de Compra

| Campo | Tipo | Obligatorio | Descripción |
|-------|------|-------------|-------------|
| `id` | UUID | Sí | Identificador único de la lista. Generado por el sistema. |
| `titulo` | String | Sí | Nombre descriptivo de la lista (ej: "Súper Mensual"). |
| `descripcion` | String | No | Notas adicionales (ej: "Comprar en el mercado de la esquina"). |
| `estado` | Enum | Sí | Estado actual del ciclo de vida: `EN_PREPARACION` o `COMPLETADA`. |
| `fecha_creacion` | DateTime | Sí | Timestamp de creación del registro. Asignado automáticamente por el sistema. |
| `fecha_procesado` | DateTime | No | Timestamp de cuando el estado cambió a `COMPLETADA`. `null` si no ha sido completada o fue reactivada. |
| `items` | List\<Item\> | No | Colección de productos asociados a la lista. |

### Estados del Ciclo de Vida

```
[NUEVA]
   │
   ▼
EN_PREPARACION ──── (completar, requiere ≥1 ítem) ───► COMPLETADA
      ▲                                                      │
      └──────────────── (reactivar) ───────────────────────┘
```

| Estado | Descripción | `fecha_procesado` |
|--------|-------------|-------------------|
| `EN_PREPARACION` | Lista en edición activa (Draft) | `null` |
| `COMPLETADA` | Compra realizada con éxito (Archived) | timestamp de completado |

> **Nota de diseño**: El estado "Reactivada" es un estado transitorio; una lista reactivada queda en `EN_PREPARACION`. No existe un tercer valor en el enum.

---

## 5. Functional Requirements

### FR-01: Creación de Lista
- El sistema acepta `titulo` (requerido) y `descripcion` (opcional) como datos de entrada.
- El sistema asigna automáticamente `id` (UUID), `fecha_creacion` (timestamp actual) y `estado = EN_PREPARACION`.
- Si el `titulo` está ausente o vacío, el sistema rechaza la operación con un error de validación descriptivo.

### FR-02: Clonación de Lista
- El sistema acepta el `id` de una lista existente en estado `COMPLETADA`.
- El sistema genera una nueva lista con: nuevo `id`, `fecha_creacion` actual, `estado = EN_PREPARACION`, `fecha_procesado = null`.
- Los ítems se copian como entidades independientes (no referencias) para no afectar la lista original.
- Si la lista origen no existe o no está en estado `COMPLETADA`, el sistema rechaza la operación con un error descriptivo.

### FR-03: Transición a COMPLETADA
- El sistema verifica que la lista tenga al menos un ítem antes de permitir la transición.
- Si la lista está vacía, el sistema responde con error: *"No se puede completar una lista vacía"*.
- Si la validación pasa, el sistema actualiza `estado = COMPLETADA` y `fecha_procesado` al timestamp actual.

### FR-04: Reactivación de Lista
- El sistema acepta el `id` de una lista en estado `COMPLETADA`.
- El sistema actualiza `estado = EN_PREPARACION` y `fecha_procesado = null`.
- Si la lista no está en estado `COMPLETADA`, el sistema rechaza la operación con un mensaje apropiado.

### FR-05: Integridad de Fechas
- Ningún cliente externo puede enviar ni modificar `fecha_creacion` o `fecha_procesado`; estos campos son de solo escritura interna del sistema.

---

## 6. Business Rules

| ID | Regla | Consecuencia si se viola |
|----|-------|--------------------------|
| BR-01 | El título es obligatorio en la creación. | Error de validación 400. |
| BR-02 | Solo listas `COMPLETADAS` pueden ser clonadas. | Error 422 con mensaje descriptivo. |
| BR-03 | No se puede completar una lista sin ítems. | Error 400: "No se puede completar una lista vacía". |
| BR-04 | El sistema (backend) asigna todas las fechas. | El frontend nunca envía campos de fecha. |
| BR-05 | La clonación no modifica la lista origen. | La lista original es inmutable post-clonación. |
| BR-06 | Al reactivar, `fecha_procesado` se limpia a `null`. | La transición es reversible sin rastro de fecha. |

---

## 7. User Scenarios & Testing

### Escenario 1: Creación exitosa de lista
1. El usuario ingresa el título "Compras de Marzo" y una descripción opcional.
2. El sistema crea la lista con estado `EN_PREPARACION` y fecha de creación automática.
3. **Resultado esperado**: Lista disponible para agregar ítems.

### Escenario 2: Intento de crear lista sin título
1. El usuario envía la solicitud sin proporcionar título.
2. El sistema rechaza la creación.
3. **Resultado esperado**: Mensaje de error claro indicando que el título es obligatorio.

### Escenario 3: Clonación de lista completada
1. El usuario selecciona "Compras de Febrero" (estado `COMPLETADA`) y elige clonarla.
2. El sistema genera "Compras de Febrero (copia)" con nuevo ID, nueva fecha y mismos ítems.
3. **Resultado esperado**: Nueva lista en `EN_PREPARACION`; lista original sin cambios.

### Escenario 4: Intento de clonar lista en preparación
1. El usuario intenta clonar "Lista Actual" (estado `EN_PREPARACION`).
2. **Resultado esperado**: El sistema rechaza la acción con mensaje descriptivo.

### Escenario 5: Completar lista con ítems
1. El usuario tiene una lista con 5 ítems en `EN_PREPARACION` y la marca como completada.
2. **Resultado esperado**: Estado pasa a `COMPLETADA`, `fecha_procesado` registra el momento exacto.

### Escenario 6: Intentar completar lista vacía
1. El usuario intenta marcar como completada una lista sin ítems.
2. **Resultado esperado**: Error "No se puede completar una lista vacía".

### Escenario 7: Reactivar lista completada
1. El usuario reabre "Compras de Febrero" (estado `COMPLETADA`).
2. **Resultado esperado**: Estado vuelve a `EN_PREPARACION`, `fecha_procesado` queda en `null`.

---

## 8. Success Criteria

| ID | Criterio | Métrica |
|----|----------|---------|
| SC-01 | Los usuarios crean una nueva lista en menos de 60 segundos. | Tiempo de tarea medido en pruebas de usabilidad. |
| SC-02 | Los usuarios que reutilizan una lista previa reducen el tiempo de creación en al menos un 70% vs. creación desde cero. | Comparación de tiempos de tarea. |
| SC-03 | El historial de listas completadas permanece 100% intacto tras operaciones de clonación. | 0 modificaciones en listas originales tras clonar. |
| SC-04 | El sistema informa correctamente todos los errores de negocio (lista vacía, estado incorrecto). | 100% de los errores definidos retornan mensajes comprensibles al usuario. |
| SC-05 | El ciclo de vida completo (crear → completar → clonar → completar) puede ejecutarse sin errores por un usuario nuevo. | Tasa de éxito ≥ 95% en pruebas con usuarios. |

---

## 9. Assumptions

- Un usuario puede tener múltiples listas activas simultáneamente (sin restricción de una sola lista activa).
- Los ítems clonados se duplican como entidades independientes; los cambios en la nueva lista no afectan la original.
- El campo `descripcion` acepta texto libre sin longitud máxima definida (se aplicará límite razonable en implementación).
- No se requiere autenticación multi-usuario en esta fase; el alcance es de un único usuario por sesión/dispositivo.
- La lógica de gestión de `items` (agregar, quitar, marcar ítem) es parte de una especificación separada.

---

## 10. Out of Scope

- Gestión interna de ítems (CRUD de productos dentro de la lista).
- Compartir listas entre múltiples usuarios.
- Notificaciones o recordatorios.
- Estadísticas de consumo o reportes históricos.
- Autenticación y autorización de usuarios.
