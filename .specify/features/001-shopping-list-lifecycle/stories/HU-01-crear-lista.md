# HU-01: Creación de Lista desde Cero

**Feature**: Gestión de Listas de Compra (001)
**Estado**: Listo para desarrollo
**Última actualización**: 2026-03-24

---

## Historia de Usuario

> **Como** usuario del hogar,
> **quiero** crear una nueva lista de compras con nombre y descripción,
> **para** organizar los productos que necesito comprar.

---

## Contexto para el Desarrollador

Esta historia cubre el punto de entrada del sistema. Es la operación más básica y fundamental: la creación de una entidad `Lista` en su estado inicial. Cualquier otra historia del sistema parte de una lista ya existente, por lo que esta HU es un prerrequisito funcional de todo el ciclo de vida.

**Actor principal**: Usuario del Hogar

---

## Criterios de Aceptación

| # | Criterio | Resultado Esperado |
|---|----------|--------------------|
| CA-1 | El usuario envía título y descripción válidos | Lista creada con `estado = EN_PREPARACION` |
| CA-2 | El usuario omite el título | El sistema rechaza la operación con error de validación |
| CA-3 | El usuario omite la descripción (campo opcional) | La lista se crea correctamente con `descripcion = null` |
| CA-4 | El usuario inspecciona la lista creada | `fecha_creacion` está poblada con el timestamp del sistema |
| CA-5 | El usuario envía un campo `fecha_creacion` en el payload | El sistema ignora el valor enviado y asigna el suyo propio |

---

## Requisitos Funcionales Relacionados

- **FR-01** — El sistema acepta `titulo` (requerido) y `descripcion` (opcional).
- **FR-01** — El sistema asigna automáticamente `id` (UUID), `fecha_creacion` (timestamp actual) y `estado = EN_PREPARACION`.
- **FR-01** — Si el `titulo` está ausente o vacío, el sistema rechaza la operación con error de validación descriptivo.
- **FR-05** — Ningún cliente externo puede enviar ni modificar `fecha_creacion`; es de escritura interna del sistema.

---

## Reglas de Negocio Aplicables

| ID | Regla |
|----|-------|
| BR-01 | El título es obligatorio. Violación → Error 400. |
| BR-04 | El sistema (backend) asigna `fecha_creacion`. El frontend no envía fechas. |

---

## Modelo de Datos Involucrado

```
Lista {
  id            : UUID       → Generado por el sistema (no aceptar del cliente)
  titulo        : String     → Obligatorio, no vacío
  descripcion   : String?    → Opcional, puede ser null
  estado        : Enum       → Siempre EN_PREPARACION al crear
  fecha_creacion: DateTime   → Asignado por el sistema (no aceptar del cliente)
  fecha_procesado: null      → Siempre null al crear
  items         : []         → Lista vacía al crear
}
```

---

## Escenarios de Prueba

### ✅ Escenario 1 — Creación exitosa con todos los campos
**Dado** que el usuario proporciona `titulo = "Compras de Marzo"` y `descripcion = "Mercado esquina"`
**Cuando** envía la solicitud de creación
**Entonces** el sistema retorna la lista con:
  - `id` generado (UUID válido)
  - `titulo = "Compras de Marzo"`
  - `descripcion = "Mercado esquina"`
  - `estado = EN_PREPARACION`
  - `fecha_creacion` con timestamp actual (±2 segundos)
  - `fecha_procesado = null`
  - `items = []`

---

### ✅ Escenario 2 — Creación exitosa sin descripción
**Dado** que el usuario proporciona solo `titulo = "Lista Express"`
**Cuando** envía la solicitud
**Entonces** la lista se crea con `descripcion = null` y sin errores.

---

### ❌ Escenario 3 — Fallo por título vacío
**Dado** que el usuario envía `titulo = ""`  (cadena vacía)
**Cuando** envía la solicitud
**Entonces** el sistema responde con error de validación y **no** crea la lista.

---

### ❌ Escenario 4 — Fallo por título ausente
**Dado** que el payload no contiene el campo `titulo`
**Cuando** envía la solicitud
**Entonces** el sistema responde con error de validación y **no** crea la lista.

---

## Criterio de Éxito Asociado

> **SC-01**: Los usuarios crean una nueva lista en menos de **60 segundos**.
> *Métrica*: Tiempo de tarea medido en pruebas de usabilidad.

---

## Notas de Implementación (Guía, no mandato)

- El `id` debe ser un UUID v4 generado en el servidor.
- El endpoint sugerido es `POST /listas`.
- El error de validación debe retornar un código `400 Bad Request` con un mensaje legible por el usuario (no solo un código de error técnico).
- La respuesta exitosa debe retornar la entidad completa creada (no solo el ID).

---

## Dependencias

- **Ninguna** — Esta HU no depende de ninguna otra historia; es el punto de entrada del sistema.

## Historias Relacionadas

- [HU-02 — Clonación Basada en Historial](./HU-02-clonar-historial.md)
- [HU-03 — Gestión del Ciclo de Vida](./HU-03-ciclo-de-vida.md)
