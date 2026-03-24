# Research & Technical Decisions

**Feature**: Gestión de Listas de Compra (001)
**Fase**: Plan — Phase 0
**Fecha**: 2026-03-24

---

## 1. Arquitectura

### Decisión: Onion Architecture (Arquitectura Cebolla)

| Atributo | Detalle |
|----------|---------|
| **Elegida** | Onion Architecture (Hexagonal / Ports & Adapters) |
| **Justificación** | Alinea con el mandato constitucional de "Lógica Centralizada" y "Simplicidad Radical". El núcleo de negocio no depende de frameworks ni de la base de datos, lo que facilita onboarding de nuevos devs y cambios de infraestructura futuros. |
| **Alternativas descartadas** | MVC plano (mezcla lógica con infraestructura), Microservicios (over-engineering para este scope) |

#### Capas y Responsabilidades

```
┌─────────────────────────────────────────────┐
│           Presentation (Frontend)           │  ← Consume API; cero lógica de negocio
├─────────────────────────────────────────────┤
│      Infrastructure (Adaptadores)           │  ← MongoDB, REST controllers, Logging
├─────────────────────────────────────────────┤
│      Application (Casos de Uso)             │  ← Orquesta; define Ports (interfaces)
├─────────────────────────────────────────────┤
│         Domain / Core (Núcleo)              │  ← Entidades, reglas, sin deps externas
└─────────────────────────────────────────────┘
         Dependencias: solo hacia adentro →
```

---

## 2. Backend

### Decisión: Java con Spring Boot

| Atributo | Detalle |
|----------|---------|
| **Elegido** | Java 21 + Spring Boot 3.x |
| **Justificación** | Ecosistema maduro, amplia documentación, soporte nativo para Onion con inyección de dependencias. Spring Data MongoDB simplifica la capa de persistencia. Mayor disponibilidad de talento en la región. |
| **Alternativa evaluada** | Golang — excelente rendimiento pero menor disponibilidad de librerías para Spring-style DI y mayor curva para devs Java. Se mantiene como opción futura si el SLA de 1000ms no se cumple. |

#### Estructura de Paquetes (Java)

```
com.groceryguard/
├── domain/
│   ├── model/          # Lista, Item, EstadoLista, TipoUnidad
│   ├── exception/      # Excepciones de dominio tipadas
│   └── port/           # Interfaces: ListaRepository, ItemRepository
├── application/
│   ├── usecase/        # CrearLista, ClonarLista, CompletarLista, ReactivarLista
│   │                   # AgregarItem, EditarItem, EliminarItem
│   └── dto/            # Request/Response DTOs
├── infrastructure/
│   ├── persistence/    # MongoListaRepository, MongoItemRepository
│   ├── web/            # REST Controllers, ExceptionHandler global
│   └── logging/        # Interceptor de logging estructurado
└── GroceryGuardApp.java
```

---

## 3. Base de Datos

### Decisión: MongoDB

| Atributo | Detalle |
|----------|---------|
| **Elegida** | MongoDB (documento embebido + colección separada de ítems) |
| **Justificación** | Flexibilidad de esquema para el modelo Lista+Items. Consultas de clonación eficientes mediante proyección de documentos. |
| **Modelo de almacenamiento** | Items como colección independiente (no embebidos) para facilitar operaciones CRUD individuales sin reescribir el documento completo de la Lista. |
| **Alternativas descartadas** | Items embebidos (dificulta edición/eliminación individual), SQL relacional (overhead innecesario para este modelo) |

#### Índices requeridos

| Colección | Campo | Tipo | Razón |
|-----------|-------|------|-------|
| `listas` | `_id` | Default | Lookups por ID |
| `listas` | `estado` | Simple | Filtro por estado en clonación |
| `items` | `lista_id` | Simple | Lookup de ítems por lista |
| `items` | `lista_id + nombre` | Compuesto único | Garantiza BR-07 (unicidad nombre/lista) en DB |

---

## 4. SLA y Concurrencia

### Decisión: Pool de Conexiones + Índices

| Requerimiento | Decisión |
|---------------|----------|
| Latencia < 1000ms (p100) | Pool MongoDB mínimo 5 conexiones, máximo 20. Índices en campos de consulta críticos. |
| 10 usuarios concurrentes | Spring Boot Tomcat con thread pool de 10-20 workers. Suficiente para el scope actual. |
| Operación más costosa | Clonación (copia de ítems): operación atómica en transacción MongoDB; índice en `lista_id` garantiza lectura < 100ms para listas con ≤ 500 ítems. |

---

## 5. Logging

### Decisión: SLF4J + Logback con patrón estructurado en consola

| Atributo | Detalle |
|----------|---------|
| **Framework** | SLF4J (abstracción) + Logback (implementación) — incluido en Spring Boot |
| **Formato** | JSON estructurado en consola: `timestamp | level | traceId | operation | listaId | duration_ms | result` |
| **Puntos de log obligatorios** | Entrada de cada endpoint (request), salida exitosa (response + duration), excepciones de negocio (nivel WARN), errores de infraestructura (nivel ERROR + stack trace simplificado) |
| **Herramienta adicional** | MDC (Mapped Diagnostic Context) para propagar `traceId` a través del hilo de ejecución |

---

## 6. Testing

### Decisión: JUnit 5 + Mockito

| Capa | Estrategia | Cobertura objetivo |
|------|------------|--------------------|
| Domain | Unit tests puros (sin mocks) | 90%+ |
| Application | Unit tests con Mockito (mock de ports) | 80%+ |
| Infrastructure | Integration tests con `@DataMongoTest` y Testcontainers (MongoDB) | Endpoints críticos |
| Presentation | E2E manual en fase inicial; Cypress futuro | — |

---

## 7. Frontend

### Decisión: React + Vite

| Atributo | Detalle |
|----------|---------|
| **Elegido** | React 18 + Vite 5 |
| **Justificación** | Virtual DOM nativo cumple el requerimiento de minimal re-renders. Ecosistema estable, ampliamente conocido. |
| **Estado** | TanStack Query (React Query) para manejo de estado servidor; elimina la necesidad de Redux para este scope. |
| **Regla de oro** | Cero lógica de negocio en componentes: validaciones, cálculos y transformaciones viven exclusivamente en el API. |
| **Loader/Progress** | Componente global `<AsyncFeedback>` se activa automáticamente cuando cualquier request supera 3000ms. |

---

## 8. API Documentation

### Decisión: SpringDoc OpenAPI 3 (Swagger UI)

- Generación automática desde anotaciones en Controllers.
- Disponible en `/swagger-ui.html` en entorno de desarrollo.
- Permite al equipo de Frontend maquetar vistas en paralelo desde Fase 1.

---

## 9. Seguridad

**Fuera de alcance para v1.0.** No se implementa autenticación ni autorización en esta fase. Alineado con la constitución del proyecto y el spec (sección Out of Scope).
