# Overview: Gestión de Listas de Compra

**Feature ID**: 001
**Branch**: `feature/001-shopping-list-lifecycle`
**Versión**: 1.0
**Fecha**: 2026-03-24

---

## Descripción General

**Grocery Guard** resuelve el problema del olvido en el hogar. Esta funcionalidad cubre el ciclo de vida completo de una lista de compras: desde su creación, pasando por la gestión de su estado (en preparación / completada), hasta la reutilización de compras anteriores mediante clonación.

El sistema está diseñado para ser usado **en movimiento**, con claridad visual y operaciones rápidas optimizadas para el entorno del supermercado.

---

## Diagrama de Casos de Uso

```mermaid
%%{init: {"theme": "default", "themeVariables": {"fontSize": "14px"}} }%%
graph LR
  %% Actores
  UsuarioHogar(["👤 Usuario del Hogar"])
  Comprador(["🛒 Comprador"])
  CompradorRecurrente(["🔄 Comprador Recurrente"])

  %% Sistema
  subgraph Sistema ["🛡️ Grocery Guard — Gestión de Listas"]
    direction TB

    UC1("📝 Crear lista\ndesde cero")
    UC2("✅ Completar lista")
    UC3("🔄 Reactivar lista")
    UC4("📋 Clonar lista\ndesde historial")

    %% Relaciones internas entre casos de uso
    UC2 -. "<<include>>\nvalidar ítems ≥ 1" .-> UC2_VAL("⚠️ Validar lista\nno vacía")
    UC4 -. "<<include>>\nverificar estado\nCOMPLETADA" .-> UC4_VAL("⚠️ Verificar estado\nCOMPLETADA")
  end

  %% Relaciones actores → casos de uso
  UsuarioHogar --> UC1
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
  COMPLETADA_CLONE --> EN_PREPARACION : [nueva lista creada]

  note right of COMPLETADA
    fecha_procesado = timestamp
    Lista inmutable para historial
  end note

  note right of EN_PREPARACION
    fecha_procesado = null
    Lista editable
  end note
```

---

## Diagrama de Flujo de Operaciones

```mermaid
flowchart TD
  Start([Usuario inicia acción]) --> Decision{¿Qué quiere hacer?}

  Decision -->|"Crear nueva lista"| Input1["Ingresa título obligatorio\n+ descripción opcional"]
  Input1 --> Validate1{¿Título vacío?}
  Validate1 -->|Sí| Error1[/"❌ Error 400\nTítulo obligatorio"/]
  Validate1 -->|No| CreateList[("✅ Lista creada\nestado: EN_PREPARACION\nfecha_creacion: NOW()")]

  Decision -->|"Completar lista"| CheckItems{¿Lista tiene ítems?}
  CheckItems -->|No| Error2[/"❌ Error 400\nNo se puede completar\nuna lista vacía"/]
  CheckItems -->|Sí| CompleteList[("✅ Lista completada\nestado: COMPLETADA\nfecha_procesado: NOW()")]

  Decision -->|"Reactivar lista"| CheckStatus1{¿Estado = COMPLETADA?}
  CheckStatus1 -->|No| Error3[/"❌ Error de transición\nestado inválido"/]
  CheckStatus1 -->|Sí| ReactivateList[("✅ Lista reactivada\nestado: EN_PREPARACION\nfecha_procesado: null")]

  Decision -->|"Clonar lista"| CheckStatus2{¿Estado = COMPLETADA?}
  CheckStatus2 -->|No| Error4[/"❌ Error 422\nSolo se clonan\nlistas COMPLETADAS"/]
  CheckStatus2 -->|Sí| CloneList[("✅ Nueva lista creada\nestado: EN_PREPARACION\nid: nuevo UUID\nítems: copia profunda")]
```

---

## Historias de Usuario

| ID | Título | Actor | Documento |
|----|--------|-------|-----------|
| HU-01 | Creación de Lista desde Cero | Usuario del Hogar | [HU-01-crear-lista.md](./stories/HU-01-crear-lista.md) |
| HU-02 | Clonación Basada en Historial | Comprador Recurrente | [HU-02-clonar-historial.md](./stories/HU-02-clonar-historial.md) |
| HU-03 | Gestión del Ciclo de Vida | Comprador | [HU-03-ciclo-de-vida.md](./stories/HU-03-ciclo-de-vida.md) |

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
| `fecha_procesado` | DateTime? | No | Asignado por el sistema al completar; `null` al reactivar |
| `items` | List\<Item\> | No | Colección de productos |

### Enum: Estado

```
EN_PREPARACION  → Lista activa, editable
COMPLETADA      → Compra finalizada, inmutable para historial
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

---

## Criterios de Éxito

| ID | Criterio | Métrica |
|----|----------|---------|
| SC-01 | Crear lista en < 60 segundos | Tiempo de tarea (usabilidad) |
| SC-02 | Clonar reduce tiempo en ≥ 70% vs. creación manual | Comparación de tiempos |
| SC-03 | Historial 100% intacto tras clonar | 0 modificaciones en lista original |
| SC-04 | Todos los errores de negocio son comprensibles | 100% de errores con mensaje legible |
| SC-05 | Ciclo completo sin errores por usuario nuevo | ≥ 95% tasa de éxito |

---

## Dependencias entre Historias

```mermaid
graph LR
  HU01["HU-01\nCrear lista"] --> HU03["HU-03\nCiclo de vida"]
  HU03 --> HU02["HU-02\nClonar historial"]

  style HU01 fill:#d4edda,stroke:#28a745
  style HU03 fill:#fff3cd,stroke:#ffc107
  style HU02 fill:#cce5ff,stroke:#004085
```

> **Orden de implementación sugerido**: HU-01 → HU-03 → HU-02

---

## Fuera de Alcance (Esta Feature)

- Gestión interna de ítems (CRUD de productos dentro de la lista)
- Compartir listas entre múltiples usuarios
- Notificaciones o recordatorios
- Estadísticas de consumo o reportes históricos
- Autenticación y autorización de usuarios

---

## Documentos Relacionados

- [Especificación completa (spec.md)](./spec.md)
- [Constitución del Proyecto](.../../memory/constitution.md)
- [Checklist de Requisitos](./checklists/requirements.md)
