# HU-04: Gestión de Ítems (Productos)

**Feature**: Gestión de Listas de Compra (001)
**Estado**: Listo para desarrollo
**Última actualización**: 2026-03-24

---

## Historia de Usuario

> **Como** usuario del hogar,
> **quiero** agregar, editar y eliminar productos dentro de una lista en borrador,
> **para** construir la lista exacta de lo que necesito comprar con la cantidad y unidad correcta.

---

## Contexto para el Desarrollador

Los ítems son el componente central de una lista de compras. Un ítem representa un producto con su nombre, cantidad y unidad de medida. Las operaciones sobre ítems solo están permitidas cuando la lista está en estado `EN_PREPARACION` — una lista `COMPLETADA` es de solo lectura.

**Regla clave de unicidad**: el `nombre` del producto es único dentro de una misma lista. Si el usuario quiere 2 tipos de arroz, debe nombrarlos diferente (ej. "Arroz blanco" y "Arroz integral").

**Actor principal**: Usuario del Hogar

---

## Criterios de Aceptación

### Agregar Ítem

| # | Criterio | Resultado Esperado |
|---|----------|--------------------|
| CA-1 | Usuario agrega un ítem válido (`nombre`, `cantidad`, `tipo_unidad`) | Ítem creado con UUID propio; aparece en la lista |
| CA-2 | Usuario omite el `nombre` | Error 400 de validación |
| CA-3 | Usuario omite la `cantidad` | Error 400 de validación |
| CA-4 | Usuario envía `cantidad = 0` o negativa | Error 400: cantidad debe ser mayor a 0 |
| CA-5 | Usuario envía `tipo_unidad` fuera del catálogo | Error 400 con lista de valores válidos |
| CA-6 | Usuario agrega un ítem con nombre ya existente en la lista | Error 409: "Ya existe un ítem con ese nombre en la lista" |
| CA-7 | Usuario intenta agregar ítem a lista `COMPLETADA` | Error 422: operación no permitida en estado actual |

### Editar Ítem

| # | Criterio | Resultado Esperado |
|---|----------|--------------------|
| CA-8 | Usuario edita `cantidad` de un ítem existente | Ítem actualizado; `nombre` e `id` sin cambios |
| CA-9 | Usuario edita `tipo_unidad` de un ítem existente | Ítem actualizado con nueva unidad |
| CA-10 | Usuario edita `cantidad` a valor ≤ 0 | Error 400 de validación |
| CA-11 | Usuario intenta editar ítem de lista `COMPLETADA` | Error 422: operación no permitida |
| CA-12 | Usuario intenta editar el `nombre` del ítem | El sistema no acepta cambio de nombre (campo inmutable post-creación) |

### Eliminar Ítem

| # | Criterio | Resultado Esperado |
|---|----------|--------------------|
| CA-13 | Usuario elimina un ítem existente | Ítem removido de la lista; ya no aparece en la colección |
| CA-14 | Usuario intenta eliminar ítem con `id` inexistente | Error 404: recurso no encontrado |
| CA-15 | Usuario intenta eliminar ítem de lista `COMPLETADA` | Error 422: operación no permitida |

---

## Requisitos Funcionales Relacionados

- **FR-06** — Agregar ítem: validaciones de nombre único, cantidad positiva y tipo_unidad del catálogo.
- **FR-07** — Editar ítem: permite modificar cantidad y/o tipo_unidad; nombre es inmutable.
- **FR-08** — Eliminar ítem: elimina por `id`; solo en listas `EN_PREPARACION`.
- **FR-03** — La lista debe tener ≥1 ítem para poder completarse (esta HU habilita ese prerequisito).

---

## Reglas de Negocio Aplicables

| ID | Regla |
|----|-------|
| BR-07 | El `nombre` del producto es único por lista. Violación → Error 409. |
| BR-08 | Los ítems solo se gestionan en listas `EN_PREPARACION`. Violación → Error 422. |
| BR-09 | `cantidad` debe ser un número positivo mayor a 0. Violación → Error 400. |
| BR-10 | `tipo_unidad` debe pertenecer al catálogo de 12 valores definidos. Violación → Error 400. |

---

## Catálogo de Tipos de Unidad

| Valor enum | Descripción |
|------------|-------------|
| `bolsa` | Para productos vendidos en bolsa |
| `caja` | Para productos en caja |
| `paquete` | Para paquetes individuales |
| `cartón` | Para cartones (ej. huevos) |
| `litro` | Para líquidos medidos en litros |
| `docena` | Para productos por docena |
| `libra` | Para productos pesados en libras |
| `kilogramo` | Para productos pesados en kilogramos |
| `canasta` | Para productos vendidos en canasta |
| `lata` | Para productos en lata |
| `botella` | Para productos en botella |
| `unidades` | Para productos contados individualmente |

---

## Modelo de Datos

```
Ítem {
  id          : UUID       → Generado por el sistema (no aceptar del cliente)
  lista_id    : UUID       → ID de la lista contenedora
  nombre      : String     → Obligatorio; único por lista; inmutable post-creación
  cantidad    : Number     → Obligatorio; > 0; acepta decimales (ej: 1.5)
  tipo_unidad : Enum       → Obligatorio; uno de los 12 valores del catálogo
}
```

---

## Escenarios de Prueba

### ✅ Escenario 1 — Agregar ítem exitosamente
**Dado** que existe la lista `id=ABC` con `estado = EN_PREPARACION`
**Cuando** el usuario agrega `{ nombre: "Arroz", cantidad: 2, tipo_unidad: "kilogramo" }`
**Entonces** el sistema crea el ítem con nuevo UUID y lo retorna dentro de la lista.

---

### ❌ Escenario 2 — Agregar ítem duplicado
**Dado** que la lista `id=ABC` ya tiene el ítem "Arroz"
**Cuando** el usuario intenta agregar "Arroz" nuevamente
**Entonces** el sistema responde `409 Conflict` con mensaje: *"Ya existe un ítem con ese nombre en la lista"*.

---

### ❌ Escenario 3 — Agregar ítem con unidad inválida
**Dado** que el usuario envía `tipo_unidad: "tonelada"` (no está en el catálogo)
**Entonces** el sistema responde `400 Bad Request` con la lista de valores válidos.

---

### ✅ Escenario 4 — Editar cantidad de un ítem
**Dado** que la lista tiene el ítem `{ id: "I1", nombre: "Arroz", cantidad: 2, tipo_unidad: "kilogramo" }`
**Cuando** el usuario envía `{ cantidad: 3 }` al endpoint de edición
**Entonces** el sistema actualiza la cantidad a 3; nombre e `id` permanecen sin cambio.

---

### ✅ Escenario 5 — Editar tipo de unidad
**Dado** que la lista tiene el ítem `{ id: "I1", nombre: "Arroz", cantidad: 2, tipo_unidad: "kilogramo" }`
**Cuando** el usuario envía `{ tipo_unidad: "bolsa" }`
**Entonces** el sistema actualiza `tipo_unidad = bolsa`.

---

### ❌ Escenario 6 — Gestionar ítems en lista COMPLETADA
**Dado** que la lista `id=ABC` tiene `estado = COMPLETADA`
**Cuando** el usuario intenta agregar, editar o eliminar un ítem
**Entonces** el sistema responde `422 Unprocessable Entity` indicando que la operación no es permitida en el estado actual.

---

### ✅ Escenario 7 — Eliminar ítem
**Dado** que la lista tiene el ítem `{ id: "I1", nombre: "Arroz" }`
**Cuando** el usuario solicita eliminar el ítem `I1`
**Entonces** el ítem ya no aparece en la colección de la lista.

---

### ✅ Escenario 8 — Ciclo completo: agregar → editar → eliminar
**Dado** una lista vacía en `EN_PREPARACION`
1. Agregar "Leche, 2, litro" → ✅ creado
2. Editar cantidad a 3 → ✅ actualizado
3. Eliminar "Leche" → ✅ removido
4. Lista queda vacía → puede agregar nuevamente

---

## Criterio de Éxito Asociado

> **SC-06**: El usuario puede agregar un producto a la lista en menos de **15 segundos** (nombre + cantidad + unidad).
> *Métrica*: Tiempo de tarea medido en pruebas de usabilidad.

---

## Notas de Implementación (Guía, no mandato)

- Endpoints sugeridos:
  - `POST   /listas/{listaId}/items`         → Agregar ítem
  - `PATCH  /listas/{listaId}/items/{itemId}` → Editar cantidad y/o tipo_unidad
  - `DELETE /listas/{listaId}/items/{itemId}` → Eliminar ítem
- La validación de estado (`EN_PREPARACION`) debe ocurrir **antes** de cualquier validación de datos del ítem.
- El sistema debe validar `tipo_unidad` contra el catálogo enum; no aceptar strings arbitrarios.
- El `nombre` debe normalizarse (ej. trim de espacios) antes de verificar unicidad para evitar duplicados por espacios accidentales.
- Error por nombre duplicado: `409 Conflict`.
- Error por operación en lista incorrecta: `422 Unprocessable Entity`.
- Error por ítem no encontrado: `404 Not Found`.

---

## Dependencias

- **HU-01** — La lista debe existir en estado `EN_PREPARACION` (creada via HU-01).
- **HU-03** — La gestión de ítems es prerequisito de completar una lista (BR-03: ≥1 ítem para completar).

## Historias Relacionadas

- [HU-01 — Creación de Lista desde Cero](./HU-01-crear-lista.md)
- [HU-02 — Clonación Basada en Historial](./HU-02-clonar-historial.md)
- [HU-03 — Gestión del Ciclo de Vida](./HU-03-ciclo-de-vida.md)
