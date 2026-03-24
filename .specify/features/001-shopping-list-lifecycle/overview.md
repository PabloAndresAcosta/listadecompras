# Overview: Gestión de Listas de Compra

**Feature ID**: 001
**Branch**: `feature/001-shopping-list-lifecycle`
**Versión**: 1.1
**Fecha**: 2026-03-24

---

## Descripción General

**Grocery Guard** resuelve el problema del olvido en el hogar. Esta funcionalidad cubre el ciclo de vida completo de una lista de compras: desde su creación, la gestión de productos (ítems) con nombre, cantidad y unidad, pasando por el control de estados (en preparación / completada), hasta la reutilización de compras anteriores mediante clonación.

El sistema está diseñado para ser usado **en movimiento**, con claridad visual y operaciones rápidas optimizadas para el entorno del supermercado.

---

## Diagrama de Casos de Uso

```mermaid
%%{init: {"theme": "default", "themeVariables": {"fontSize": "14px"}} }%%
graph LR
  UsuarioHogar(["👤 Usuario del Hogar"])
  Comprador(["🛒 Comprador"])
  CompradorRecurrente(["🔄 Comprador Recurrente"])

  subgraph Sistema ["🛡️ Grocery Guard — Gestión de Listas"]
    direction TB

    UC1("📝 Crear lista\ndesde cero")
    UC2("✅ Completar lista")
    UC3("🔄 Reactivar lista")
    UC4("📋 Clonar lista\ndesde historial")
    UC5("🧺 Gestionar ítems\nagregar · editar · eliminar")

    UC2 -. "<<include>>\nvalidar ítems ≥ 1" .-> UC2_VAL("⚠️ Validar lista\nno vacía")
    UC4 -. "<<include>>\nverificar estado\nCOMPLETADA" .-> UC4_VAL("⚠️ Verificar estado\nCOMPLETADA")
    UC5 -. "<<include>>\nvalidar estado\nEN_PREPARACION" .-> UC5_VAL("⚠️ Verificar lista\nen borrador")
    UC5 -. "<<include>>\nvalidar nombre\núnico en lista" .-> UC5_VAL2("⚠️ Verificar nombre\nno duplicado")
  end

  UsuarioHogar --> UC1
  UsuarioHogar --> UC5
  UsuarioHogar --> UC3
  Comprador --> UC2
  Comprador --> UC3
  CompradorRecurrente --> UC4
  CompradorRecurrente --> UC1
```

---

## Diagrama de Estados del Ciclo de Vida

```mermaid
stateDiagram-v2
  direction LR

  [*] --> EN_PREPARACION : crear lista

  EN_PREPARACION --> COMPLETADA : completar()\n[items ≥ 1]\nfecha_procesado = NOW()

  EN_PREPARACION --> EN_PREPARACION : completar()\n[items = 0]\n❌ Error 400

  COMPLETADA --> EN_PREPARACION : reactivar()\nfecha_procesado = null

  COMPLETADA --> COMPLETADA_CLONE : clonar()
  COMPLETADA_CLONE --> EN_PREPARACION : [nueva lista creada\ncon ítems copiados]

  note right of COMPLETADA
    fecha_procesado = timestamp
    Ítems: solo lectura
  end note

  note right of EN_PREPARACION
    fecha_procesado = null
    Ítems: agregar/editar/eliminar
  end note
```

---

## Diagrama de Flujo de Operaciones

```mermaid
flowchart TD
  Start([Usuario inicia acción]) --> Decision{¿Qué quiere hacer?}

  Decision -->|"Crear lista"| Input1["Ingresa título obligatorio\n+ descripción opcional"]
  Input1 --> Validate1{¿Título vacío?}
  Validate1 -->|Sí| Error1[/"❌ Error 400\nTítulo obligatorio"/]
  Validate1 -->|No| CreateList[("✅ Lista creada\nestado: EN_PREPARACION")]

  Decision -->|"Gestionar ítems"| CheckDraft{¿Estado = EN_PREPARACION?}
  CheckDraft -->|No| Error5[/"❌ Error 422\nLista no editable"/]
  CheckDraft -->|Sí| ItemAction{¿Qué acción?}
  ItemAction -->|Agregar| CheckDup{¿Nombre duplicado?}
  CheckDup -->|Sí| Error6[/"❌ Error 409\nNombre ya existe\nen la lista"/]
  CheckDup -->|No| AddItem[("✅ Ítem creado\nnombre · cantidad · tipo_unidad")]
  ItemAction -->|Editar| EditItem[("✅ Ítem actualizado\ncantidad y/o tipo_unidad")]
  ItemAction -->|Eliminar| DeleteItem[("✅ Ítem eliminado")]

  Decision -->|"Completar lista"| CheckItems{¿Lista tiene ítems?}
  CheckItems -->|No| Error2[/"❌ Error 400\nNo se puede completar\nuna lista vacía"/]
  CheckItems -->|Sí| CompleteList[("✅ Lista completada\nestado: COMPLETADA\nfecha_procesado: NOW()")]

  Decision -->|"Reactivar lista"| CheckStatus1{¿Estado = COMPLETADA?}
  CheckStatus1 -->|No| Error3[/"❌ Error de transición\nestado inválido"/]
  CheckStatus1 -->|Sí| ReactivateList[("✅ Lista reactivada\nestado: EN_PREPARACION\nfecha_procesado: null")]

  Decision -->|"Clonar lista"| CheckStatus2{¿Estado = COMPLETADA?}
  CheckStatus2 -->|No| Error4[/"❌ Error 422\nSolo se clonan\nlistas COMPLETADAS"/]
  CheckStatus2 -->|Sí| CloneList[("✅ Nueva lista creada\nestado: EN_PREPARACION\nítems: copia profunda")]
```

---

## Historias de Usuario

| ID | Título | Actor | Documento |
|----|--------|-------|-----------|
| HU-01 | Creación de Lista desde Cero | Usuario del Hogar | [HU-01-crear-lista.md](./stories/HU-01-crear-lista.md) |
| HU-02 | Clonación Basada en Historial | Comprador Recurrente | [HU-02-clonar-historial.md](./stories/HU-02-clonar-historial.md) |
| HU-03 | Gestión del Ciclo de Vida | Comprador | [HU-03-ciclo-de-vida.md](./stories/HU-03-ciclo-de-vida.md) |
| HU-04 | Gestión de Ítems (Productos) | Usuario del Hogar | [HU-04-gestionar-items.md](./stories/HU-04-gestionar-items.md) |

---

## Modelo de Dominio

### Entidad: Lista de Compra

| Campo | Tipo | Oblig. | Descripción |
|-------|------|--------|-------------|
| `id` | UUID | Sí | Generado por el sistema |
| `titulo` | String | Sí | Nombre de la lista |
| `descripcion` | String? | No | Notas adicionales |
| `estado` | Enum | Sí | `EN_PREPARACION` \| `COMPLETADA` |
| `fecha_creacion` | DateTime | Sí | Asignado por el sistema al crear |
| `fecha_procesado` | DateTime? | No | Asignado al completar; `null` al reactivar |
| `items` | List\<Item\> | No | Colección de productos |

### Entidad: Ítem (Producto)

| Campo | Tipo | Oblig. | Descripción |
|-------|------|--------|-------------|
| `id` | UUID | Sí | Generado por el sistema |
| `lista_id` | UUID | Sí | Referencia a la lista contenedora |
| `nombre` | String | Sí | **Único por lista**; inmutable post-creación |
| `cantidad` | Number | Sí | Número positivo > 0 |
| `tipo_unidad` | Enum | Sí | Uno de 12 valores del catálogo |

### Catálogo TipoUnidad

`bolsa` · `caja` · `paquete` · `cartón` · `litro` · `docena` · `libra` · `kilogramo` · `canasta` · `lata` · `botella` · `unidades`

### Enum: Estado de Lista

```
EN_PREPARACION  → Lista activa; ítems editables
COMPLETADA      → Compra finalizada; ítems solo lectura
```

---

## Reglas de Negocio (Resumen)

| ID | Regla | Error si viola |
|----|-------|----------------|
| BR-01 | Título obligatorio al crear | 400 |
| BR-02 | Solo listas `COMPLETADAS` se pueden clonar | 422 |
| BR-03 | No se puede completar lista vacía | 400: "No se puede completar una lista vacía" |
| BR-04 | El sistema asigna todas las fechas | El frontend no envía fechas |
| BR-05 | Clonación no modifica la lista origen | — |
| BR-06 | Reactivar limpia `fecha_procesado` a `null` | — |
| BR-07 | Nombre de producto único por lista | 409: "Ya existe un ítem con ese nombre en la lista" |
| BR-08 | Ítems solo gestionables en `EN_PREPARACION` | 422 |
| BR-09 | `cantidad` debe ser > 0 | 400 |
| BR-10 | `tipo_unidad` debe pertenecer al catálogo | 400 con lista de valores válidos |

---

## Criterios de Éxito

| ID | Criterio | Métrica |
|----|----------|---------|
| SC-01 | Crear lista en < 60 segundos | Tiempo de tarea (usabilidad) |
| SC-02 | Clonar reduce tiempo en ≥ 70% vs. creación manual | Comparación de tiempos |
| SC-03 | Historial 100% intacto tras clonar | 0 modificaciones en lista original |
| SC-04 | Todos los errores de negocio son comprensibles | 100% de errores con mensaje legible |
| SC-05 | Ciclo completo sin errores por usuario nuevo | ≥ 95% tasa de éxito |
| SC-06 | Agregar un producto en < 15 segundos | Tiempo de tarea (usabilidad) |

---

## Dependencias entre Historias

```mermaid
graph LR
  HU01["HU-01\nCrear lista"] --> HU04["HU-04\nGestionar ítems"]
  HU01 --> HU03["HU-03\nCiclo de vida"]
  HU04 --> HU03
  HU03 --> HU02["HU-02\nClonar historial"]

  style HU01 fill:#d4edda,stroke:#28a745
  style HU04 fill:#e2d9f3,stroke:#6f42c1
  style HU03 fill:#fff3cd,stroke:#ffc107
  style HU02 fill:#cce5ff,stroke:#004085
```

> **Orden de implementación sugerido**: HU-01 → HU-04 → HU-03 → HU-02

---

## Fuera de Alcance (Esta Feature)

- Compartir listas entre múltiples usuarios
- Notificaciones o recordatorios
- Estadísticas de consumo o reportes históricos
- Autenticación y autorización de usuarios
- Marcar ítems como "comprado" dentro del supermercado

---

## Documentos Relacionados

- [Especificación completa (spec.md)](./spec.md)
- [Constitución del Proyecto](../../memory/constitution.md)
- [Checklist de Requisitos](./checklists/requirements.md)
