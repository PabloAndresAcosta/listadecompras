# HU-02: Clonación Basada en Historial

**Feature**: Gestión de Listas de Compra (001)
**Estado**: Listo para desarrollo
**Última actualización**: 2026-03-24

---

## Historia de Usuario

> **Como** comprador recurrente,
> **quiero** seleccionar una lista `COMPLETADA` y clonarla para una nueva compra,
> **para** no tener que reescribir los mismos productos cada semana.

---

## Contexto para el Desarrollador

Esta historia implementa el mecanismo central de reutilización del historial. El principio clave es la **inmutabilidad**: la clonación genera una entidad completamente nueva; la lista de origen jamás se toca. Este diseño es intencional para preservar la integridad del historial de consumo.

La operación de clonación es una **copia profunda (deep copy)** de la lista y sus ítems, no una referencia.

**Actor principal**: Comprador Recurrente

---

## Criterios de Aceptación

| # | Criterio | Resultado Esperado |
|---|----------|--------------------|
| CA-1 | El usuario clona una lista en estado `COMPLETADA` | Nueva lista creada con nuevo `id` y `fecha_creacion` actuales |
| CA-2 | El usuario inspecciona la lista clonada | `estado = EN_PREPARACION`, `fecha_procesado = null` |
| CA-3 | El usuario inspecciona los ítems de la lista clonada | Todos los ítems de la original están presentes como copias independientes |
| CA-4 | El usuario modifica un ítem de la lista clonada | La lista original permanece sin cambios |
| CA-5 | El usuario intenta clonar una lista en estado `EN_PREPARACION` | El sistema rechaza la operación con mensaje descriptivo |
| CA-6 | El usuario intenta clonar con un `id` inexistente | El sistema retorna error de recurso no encontrado |
| CA-7 | El usuario verifica la lista original tras la clonación | La lista original no ha sido modificada (mismo `id`, mismos ítems, mismo estado) |

---

## Requisitos Funcionales Relacionados

- **FR-02** — El sistema acepta el `id` de una lista existente en estado `COMPLETADA`.
- **FR-02** — El sistema genera una nueva lista con: nuevo `id`, `fecha_creacion` actual, `estado = EN_PREPARACION`, `fecha_procesado = null`.
- **FR-02** — Los ítems se copian como entidades independientes (no referencias) para no afectar la lista original.
- **FR-02** — Si la lista origen no existe o no está en estado `COMPLETADA`, el sistema rechaza la operación con error descriptivo.
- **FR-05** — El sistema asigna la `fecha_creacion` de la nueva lista; el cliente no la envía.

---

## Reglas de Negocio Aplicables

| ID | Regla |
|----|-------|
| BR-02 | Solo listas `COMPLETADAS` pueden ser clonadas. Violación → Error 422. |
| BR-04 | El sistema asigna `fecha_creacion` de la nueva lista. |
| BR-05 | La clonación no modifica la lista origen. Es inmutable post-clonación. |

---

## Modelo de Datos Involucrado

```
Operación: POST /listas/{id}/clonar

Input:
  id (path param) : UUID → ID de la lista COMPLETADA a clonar

Output (nueva lista creada):
  Lista {
    id             : UUID       → Nuevo UUID generado por el sistema
    titulo         : String     → Copiado de la lista original
    descripcion    : String?    → Copiado de la lista original
    estado         : Enum       → Siempre EN_PREPARACION
    fecha_creacion : DateTime   → Timestamp actual (del sistema)
    fecha_procesado: null       → Siempre null al clonar
    items          : [Item]     → Copia profunda de los ítems de la original
  }
```

---

## Diagrama de la Operación

```
Lista Original (COMPLETADA)
        │
        │  clonar(id)
        ▼
[Sistema crea nueva lista]──────────────────────────────────────────────────────►
        │                                                                          │
        │  deep copy items                                     Lista Nueva         │
        └──────────────────────────────────────────────────► (EN_PREPARACION)     │
                                                                                   │
Lista Original: INTACTA ◄──────────────────────────────────────────────────────┘
```

---

## Escenarios de Prueba

### ✅ Escenario 1 — Clonación exitosa
**Dado** que existe la lista `id=ABC` con estado `COMPLETADA` y 3 ítems
**Cuando** el usuario solicita clonar la lista `ABC`
**Entonces** el sistema crea una nueva lista con:
  - `id` diferente a `ABC` (nuevo UUID)
  - `estado = EN_PREPARACION`
  - `fecha_creacion` con timestamp actual
  - `fecha_procesado = null`
  - 3 ítems copiados (IDs propios, independientes)
**Y** la lista `ABC` permanece con `estado = COMPLETADA` y sus ítems originales intactos.

---

### ❌ Escenario 2 — Intento de clonar lista en preparación
**Dado** que existe la lista `id=XYZ` con estado `EN_PREPARACION`
**Cuando** el usuario solicita clonarla
**Entonces** el sistema responde con error descriptivo y **no** crea ninguna lista nueva.

---

### ❌ Escenario 3 — Intento de clonar lista inexistente
**Dado** que el `id=NOEXISTE` no corresponde a ninguna lista
**Cuando** el usuario solicita clonarla
**Entonces** el sistema responde con error de recurso no encontrado.

---

### ✅ Escenario 4 — Independencia de ítems tras clonación
**Dado** que se clonó la lista `ABC` y se creó la lista `DEF`
**Cuando** el usuario modifica el ítem `Arroz` en la lista `DEF`
**Entonces** el ítem `Arroz` en la lista `ABC` no se ve afectado.

---

## Criterio de Éxito Asociado

> **SC-02**: Los usuarios que reutilizan una lista previa reducen el tiempo de creación en al menos un **70%** vs. creación desde cero.
> *Métrica*: Comparación de tiempos de tarea en pruebas de usabilidad.

> **SC-03**: El historial de listas completadas permanece **100% intacto** tras operaciones de clonación.
> *Métrica*: 0 modificaciones en listas originales tras clonar.

---

## Notas de Implementación (Guía, no mandato)

- El endpoint sugerido es `POST /listas/{id}/clonar`.
- La operación debe ser transaccional: si falla la copia de ítems, no debe persistir la nueva lista (rollback).
- El error por estado incorrecto debe retornar `422 Unprocessable Entity` con mensaje claro.
- El error por lista no encontrada debe retornar `404 Not Found`.
- La copia de ítems debe generar nuevos IDs para cada ítem clonado.

---

## Dependencias

- **HU-01** — Debe existir al menos una lista creada (en estado `EN_PREPARACION`) para que eventualmente pase a `COMPLETADA` y pueda ser clonada.
- **HU-03** — La lista fuente debe estar en estado `COMPLETADA` (transición cubierta en HU-03).

## Historias Relacionadas

- [HU-01 — Creación de Lista desde Cero](./HU-01-crear-lista.md)
- [HU-03 — Gestión del Ciclo de Vida](./HU-03-ciclo-de-vida.md)
