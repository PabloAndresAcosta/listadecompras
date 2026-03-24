# API Contracts — Grocery Guard v1.0

**Estándar**: REST / JSON
**Base URL**: `/api/v1`
**Documentación interactiva**: `/swagger-ui.html` (entorno dev)
**Fecha**: 2026-03-24

---

## Convenciones Generales

### Respuesta Exitosa

```json
{
  "data": { ... },
  "timestamp": "2026-03-24T10:00:00.000Z"
}
```

### Respuesta de Error

```json
{
  "error": {
    "code": "LISTA_VACIA",
    "message": "No se puede completar una lista vacía",
    "timestamp": "2026-03-24T10:00:00.000Z"
  }
}
```

### Códigos HTTP Utilizados

| Código | Significado |
|--------|-------------|
| `200` | Operación exitosa (GET, PATCH) |
| `201` | Recurso creado (POST) |
| `204` | Eliminación exitosa (DELETE) |
| `400` | Error de validación de entrada |
| `404` | Recurso no encontrado |
| `409` | Conflicto (nombre duplicado) |
| `422` | Operación no permitida en estado actual |

---

## Endpoints: Listas

### POST /api/v1/listas
**Crear lista desde cero (HU-01)**

**Request Body**:
```json
{
  "titulo": "Compras de Marzo",
  "descripcion": "Mercado de la esquina"
}
```

**Response 201**:
```json
{
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "titulo": "Compras de Marzo",
    "descripcion": "Mercado de la esquina",
    "estado": "EN_PREPARACION",
    "fecha_creacion": "2026-03-24T10:00:00.000Z",
    "fecha_procesado": null,
    "items": []
  }
}
```

**Errores**:

| Código | Condición | Error Code |
|--------|-----------|------------|
| `400` | `titulo` ausente o vacío | `TITULO_REQUERIDO` |

---

### GET /api/v1/listas
**Listar todas las listas**

**Response 200**:
```json
{
  "data": [
    {
      "id": "...",
      "titulo": "Compras de Marzo",
      "estado": "EN_PREPARACION",
      "fecha_creacion": "...",
      "fecha_procesado": null,
      "total_items": 5
    }
  ]
}
```

---

### GET /api/v1/listas/{id}
**Obtener lista con sus ítems**

**Response 200**:
```json
{
  "data": {
    "id": "...",
    "titulo": "Compras de Marzo",
    "descripcion": "...",
    "estado": "EN_PREPARACION",
    "fecha_creacion": "...",
    "fecha_procesado": null,
    "items": [
      {
        "id": "...",
        "nombre": "Arroz",
        "cantidad": 2.0,
        "tipo_unidad": "KILOGRAMO"
      }
    ]
  }
}
```

**Errores**:

| Código | Condición | Error Code |
|--------|-----------|------------|
| `404` | Lista no encontrada | `LISTA_NO_ENCONTRADA` |

---

### POST /api/v1/listas/{id}/clonar
**Clonar lista desde historial (HU-02)**

**Request Body**: *(vacío o con descripción opcional para la nueva lista)*
```json
{
  "descripcion": "Copia para Abril"
}
```

**Response 201**: *(nueva lista completa con ítems copiados)*

**Errores**:

| Código | Condición | Error Code |
|--------|-----------|------------|
| `404` | Lista origen no encontrada | `LISTA_NO_ENCONTRADA` |
| `422` | Lista origen no está `COMPLETADA` | `ESTADO_INVALIDO_PARA_CLONAR` |

---

### PATCH /api/v1/listas/{id}/completar
**Completar lista (HU-03)**

**Request Body**: *(vacío)*

**Response 200**:
```json
{
  "data": {
    "id": "...",
    "estado": "COMPLETADA",
    "fecha_procesado": "2026-03-24T15:30:00.000Z"
  }
}
```

**Errores**:

| Código | Condición | Error Code |
|--------|-----------|------------|
| `404` | Lista no encontrada | `LISTA_NO_ENCONTRADA` |
| `400` | Lista sin ítems | `LISTA_VACIA` |
| `422` | Lista ya está `COMPLETADA` | `TRANSICION_INVALIDA` |

---

### PATCH /api/v1/listas/{id}/reactivar
**Reactivar lista (HU-03)**

**Request Body**: *(vacío)*

**Response 200**:
```json
{
  "data": {
    "id": "...",
    "estado": "EN_PREPARACION",
    "fecha_procesado": null
  }
}
```

**Errores**:

| Código | Condición | Error Code |
|--------|-----------|------------|
| `404` | Lista no encontrada | `LISTA_NO_ENCONTRADA` |
| `422` | Lista ya está `EN_PREPARACION` | `TRANSICION_INVALIDA` |

---

## Endpoints: Ítems

### POST /api/v1/listas/{listaId}/items
**Agregar ítem a lista (HU-04)**

**Request Body**:
```json
{
  "nombre": "Arroz",
  "cantidad": 2.0,
  "tipo_unidad": "KILOGRAMO"
}
```

**Response 201**:
```json
{
  "data": {
    "id": "...",
    "lista_id": "...",
    "nombre": "Arroz",
    "cantidad": 2.0,
    "tipo_unidad": "KILOGRAMO"
  }
}
```

**Errores**:

| Código | Condición | Error Code |
|--------|-----------|------------|
| `400` | `nombre` vacío | `NOMBRE_REQUERIDO` |
| `400` | `cantidad` ≤ 0 | `CANTIDAD_INVALIDA` |
| `400` | `tipo_unidad` no válido | `TIPO_UNIDAD_INVALIDO` |
| `404` | Lista no encontrada | `LISTA_NO_ENCONTRADA` |
| `409` | Nombre ya existe en la lista | `NOMBRE_DUPLICADO` |
| `422` | Lista no está `EN_PREPARACION` | `LISTA_NO_EDITABLE` |

---

### PATCH /api/v1/listas/{listaId}/items/{itemId}
**Editar ítem (HU-04)**

**Request Body** *(enviar solo los campos a modificar)*:
```json
{
  "cantidad": 3.0,
  "tipo_unidad": "BOLSA"
}
```

**Response 200**:
```json
{
  "data": {
    "id": "...",
    "nombre": "Arroz",
    "cantidad": 3.0,
    "tipo_unidad": "BOLSA"
  }
}
```

**Errores**:

| Código | Condición | Error Code |
|--------|-----------|------------|
| `400` | `cantidad` ≤ 0 | `CANTIDAD_INVALIDA` |
| `400` | `tipo_unidad` no válido | `TIPO_UNIDAD_INVALIDO` |
| `404` | Ítem no encontrado | `ITEM_NO_ENCONTRADO` |
| `422` | Lista no está `EN_PREPARACION` | `LISTA_NO_EDITABLE` |

---

### DELETE /api/v1/listas/{listaId}/items/{itemId}
**Eliminar ítem (HU-04)**

**Response 204**: *(sin cuerpo)*

**Errores**:

| Código | Condición | Error Code |
|--------|-----------|------------|
| `404` | Ítem no encontrado | `ITEM_NO_ENCONTRADO` |
| `422` | Lista no está `EN_PREPARACION` | `LISTA_NO_EDITABLE` |

---

## Catálogo TipoUnidad (valores válidos)

```
BOLSA | CAJA | PAQUETE | CARTON | LITRO | DOCENA
LIBRA | KILOGRAMO | CANASTA | LATA | BOTELLA | UNIDADES
```
