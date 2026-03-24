# Documentación Técnica — Grocery Guard v1.0

**Feature**: Gestión de Listas de Compra (001)
**Arquitectura**: Onion Architecture
**Stack**: Java Spring Boot · MongoDB · React + Vite
**Fecha**: 2026-03-24

---

## 1. Diagrama de Componentes

```mermaid
graph TB
  subgraph Frontend ["🖥️ Presentation Layer — React + Vite"]
    UI["Componentes React\n(cero lógica de negocio)"]
    RQ["TanStack Query\n(estado servidor)"]
    Loader["AsyncFeedback\n(loader > 3s)"]
    UI --> RQ
    UI --> Loader
  end

  subgraph Backend ["☕ Spring Boot Application"]
    subgraph Infrastructure ["🔌 Infrastructure Layer"]
      REST["REST Controllers\n@RestController"]
      EH["Global Exception Handler\n@ControllerAdvice"]
      LOG["Logging Interceptor\nSLF4J + MDC"]
      MR["Mongo Repositories\nSpring Data"]
    end

    subgraph Application ["⚙️ Application Layer"]
      UC_L["ListaUseCases\nCrear · Clonar · Completar · Reactivar"]
      UC_I["ItemUseCases\nAgregar · Editar · Eliminar"]
      PORTS["Ports (Interfaces)\nListaRepository\nItemRepository"]
    end

    subgraph Domain ["💎 Domain / Core Layer"]
      ENT["Entities\nLista · Item"]
      ENUM["Enums\nEstadoLista · TipoUnidad"]
      EXC["Domain Exceptions\nListaVaciaException\nNombreDuplicadoException\nTransicionInvalidaException"]
    end

    REST --> UC_L
    REST --> UC_I
    EH --> REST
    LOG --> REST
    UC_L --> PORTS
    UC_I --> PORTS
    UC_L --> ENT
    UC_I --> ENT
    PORTS --> MR
    ENT --> ENUM
    ENT --> EXC
  end

  subgraph DB ["🍃 MongoDB"]
    COL_L[("Colección: listas")]
    COL_I[("Colección: items")]
  end

  subgraph Docs ["📄 API Docs"]
    SWAGGER["Swagger UI\n/swagger-ui.html"]
  end

  RQ -->|"HTTP REST JSON"| REST
  MR --> COL_L
  MR --> COL_I
  REST --> SWAGGER

  style Domain fill:#fff0f0,stroke:#e53935
  style Application fill:#fff8e1,stroke:#f9a825
  style Infrastructure fill:#e8f5e9,stroke:#43a047
  style Frontend fill:#e3f2fd,stroke:#1e88e5
  style DB fill:#f3e5f5,stroke:#8e24aa
```

---

## 2. Diagramas de Secuencia

### 2.1 HU-01: Crear Lista

```mermaid
sequenceDiagram
  actor U as Usuario
  participant FE as Frontend (React)
  participant API as REST Controller
  participant UC as CrearListaUseCase
  participant DOM as Lista (Domain)
  participant REPO as ListaRepository
  participant DB as MongoDB

  U->>FE: Ingresa título + descripción opcional
  FE->>API: POST /api/v1/listas\n{ titulo, descripcion }
  API->>API: Valida request (titulo no vacío)
  alt titulo vacío
    API-->>FE: 400 { error: TITULO_REQUERIDO }
    FE-->>U: Muestra error de validación
  else titulo válido
    API->>UC: ejecutar(titulo, descripcion)
    UC->>DOM: new Lista(UUID, titulo, desc, EN_PREPARACION, NOW(), null, [])
    UC->>REPO: guardar(lista)
    REPO->>DB: insertOne(listas, documento)
    DB-->>REPO: OK
    REPO-->>UC: lista persistida
    UC-->>API: ListaDTO
    API-->>FE: 201 { data: ListaDTO }
    FE-->>U: Lista creada, redirige a vista de ítems
  end
```

---

### 2.2 HU-02: Clonar Lista

```mermaid
sequenceDiagram
  actor U as Usuario
  participant FE as Frontend
  participant API as REST Controller
  participant UC as ClonarListaUseCase
  participant REPO_L as ListaRepository
  participant REPO_I as ItemRepository
  participant DB as MongoDB

  U->>FE: Selecciona lista COMPLETADA → "Clonar"
  FE->>API: POST /api/v1/listas/{id}/clonar
  API->>UC: ejecutar(listaOrigenId)
  UC->>REPO_L: buscarPorId(listaOrigenId)
  REPO_L->>DB: findOne({ _id: id })
  DB-->>REPO_L: lista origen

  alt lista no encontrada
    UC-->>API: ListaNoEncontradaException
    API-->>FE: 404 { error: LISTA_NO_ENCONTRADA }
  else estado ≠ COMPLETADA
    UC-->>API: TransicionInvalidaException
    API-->>FE: 422 { error: ESTADO_INVALIDO_PARA_CLONAR }
  else OK
    UC->>REPO_I: buscarPorListaId(listaOrigenId)
    DB-->>REPO_I: [items originales]

    Note over UC: Inicia transacción MongoDB

    UC->>REPO_L: guardar(nuevaLista)
    loop Para cada ítem original
      UC->>REPO_I: guardar(itemCopiado con nuevo id + nuevo lista_id)
    end

    Note over UC: Commit transacción

    UC-->>API: nuevaListaConItems
    API-->>FE: 201 { data: nuevaLista }
    FE-->>U: Nueva lista en EN_PREPARACION lista para editar
  end
```

---

### 2.3 HU-03: Completar Lista

```mermaid
sequenceDiagram
  actor U as Usuario
  participant FE as Frontend
  participant API as REST Controller
  participant UC as CompletarListaUseCase
  participant REPO_L as ListaRepository
  participant REPO_I as ItemRepository
  participant DB as MongoDB

  U->>FE: Pulsa "Completar lista"
  FE->>API: PATCH /api/v1/listas/{id}/completar
  API->>UC: ejecutar(listaId)
  UC->>REPO_L: buscarPorId(listaId)
  DB-->>REPO_L: lista

  alt lista no encontrada
    API-->>FE: 404 LISTA_NO_ENCONTRADA
  else estado = COMPLETADA
    API-->>FE: 422 TRANSICION_INVALIDA
  else
    UC->>REPO_I: contarPorListaId(listaId)
    DB-->>REPO_I: count

    alt count = 0
      UC-->>API: ListaVaciaException
      API-->>FE: 400 { error: LISTA_VACIA, message: "No se puede completar una lista vacía" }
      FE-->>U: Toast de error
    else count >= 1
      UC->>REPO_L: actualizarEstado(listaId, COMPLETADA, fechaProcesado=NOW())
      DB-->>REPO_L: OK
      UC-->>API: ListaDTO actualizada
      API-->>FE: 200 { data: { estado: COMPLETADA, fecha_procesado: "..." } }
      FE-->>U: Lista marcada como completada
    end
  end
```

---

### 2.4 HU-04: Agregar Ítem

```mermaid
sequenceDiagram
  actor U as Usuario
  participant FE as Frontend
  participant API as REST Controller
  participant UC as AgregarItemUseCase
  participant REPO_L as ListaRepository
  participant REPO_I as ItemRepository
  participant DB as MongoDB

  U->>FE: Ingresa nombre + cantidad + tipo_unidad
  FE->>API: POST /api/v1/listas/{listaId}/items\n{ nombre, cantidad, tipo_unidad }

  API->>API: Valida campos (nombre, cantidad>0, tipo_unidad en catálogo)
  alt validación falla
    API-->>FE: 400 error de validación
  else
    API->>UC: ejecutar(listaId, nombre, cantidad, tipoUnidad)
    UC->>REPO_L: buscarPorId(listaId)

    alt lista no existe
      API-->>FE: 404 LISTA_NO_ENCONTRADA
    else estado ≠ EN_PREPARACION
      API-->>FE: 422 LISTA_NO_EDITABLE
    else
      UC->>REPO_I: existeNombreEnLista(listaId, nombre)
      DB-->>REPO_I: boolean

      alt nombre duplicado
        API-->>FE: 409 { error: NOMBRE_DUPLICADO, message: "Ya existe un ítem con ese nombre" }
        FE-->>U: Muestra error inline
      else
        UC->>REPO_I: guardar(nuevoItem)
        DB-->>REPO_I: OK
        UC-->>API: ItemDTO
        API-->>FE: 201 { data: ItemDTO }
        FE-->>U: Ítem aparece en la lista
      end
    end
  end
```

---

## 3. Diagrama de Despliegue

```mermaid
graph TB
  subgraph Cliente ["💻 Cliente (Browser / Mobile)"]
    BROWSER["Browser\nReact SPA\nVite Build"]
  end

  subgraph Servidor ["🖥️ Servidor de Aplicación"]
    subgraph JVM ["JVM — Java 21"]
      SPRING["Spring Boot App\n:8080"]
      SWAGGER_UI["Swagger UI\n/swagger-ui.html"]
    end
    STATIC["Static Files\n(React build)"]
  end

  subgraph BaseDatos ["🍃 Capa de Datos"]
    MONGO[("MongoDB\n:27017\nDB: grocery_guard")]
    IDX_L["Index: listas.estado"]
    IDX_I["Index: items.lista_id\nIndex: items.(lista_id,nombre) UNIQUE"]
    MONGO --> IDX_L
    MONGO --> IDX_I
  end

  subgraph Config ["⚙️ Configuración"]
    ENV["application.properties\n- MongoDB URI\n- Pool: min=5 max=20\n- Server port: 8080"]
  end

  BROWSER -->|"HTTP :8080\nREST + JSON"| SPRING
  BROWSER -->|"Carga estática"| STATIC
  SPRING -->|"Spring Data MongoDB\nConnection Pool"| MONGO
  SPRING --> SWAGGER_UI
  ENV --> SPRING

  style Cliente fill:#e3f2fd,stroke:#1e88e5
  style Servidor fill:#e8f5e9,stroke:#43a047
  style BaseDatos fill:#f3e5f5,stroke:#8e24aa
  style Config fill:#fff8e1,stroke:#f9a825
```

---

## 4. Diagrama de Flujo de la Capa de Logging

```mermaid
flowchart LR
  REQ["HTTP Request\nentra al Controller"]
  INT["Logging Interceptor\npreHandle()"]
  MDC["MDC.put(traceId, op, listaId)"]
  BIZ["Lógica de Negocio\n(UseCase)"]
  LOG_OK["LOG.info()\nop | listaId | duration_ms | OK"]
  LOG_WARN["LOG.warn()\nerror de negocio\n+ mensaje descriptivo"]
  LOG_ERR["LOG.error()\nerror de infra\n+ stack trace simplificado"]
  RESP["HTTP Response\nsale del Controller"]
  MDC_CLR["MDC.clear()\npostHandle()"]

  REQ --> INT
  INT --> MDC
  MDC --> BIZ
  BIZ -->|"Éxito"| LOG_OK
  BIZ -->|"Excepción dominio"| LOG_WARN
  BIZ -->|"Excepción infra"| LOG_ERR
  LOG_OK --> RESP
  LOG_WARN --> RESP
  LOG_ERR --> RESP
  RESP --> MDC_CLR
```

### Formato de Log Estructurado

```
[2026-03-24T10:05:32Z] INFO  traceId=abc123 op=CREAR_LISTA listaId=uuid-001 duration_ms=45 result=OK
[2026-03-24T10:05:35Z] WARN  traceId=def456 op=AGREGAR_ITEM listaId=uuid-002 error=NOMBRE_DUPLICADO nombre="Arroz"
[2026-03-24T10:05:40Z] ERROR traceId=ghi789 op=GUARDAR_LISTA error=MongoTimeoutException message="Connection timeout" stackTrace="..."
```

---

## 5. Estructura de Paquetes (Backend)

```
src/main/java/com/groceryguard/
│
├── domain/                          # 💎 CORE — sin dependencias externas
│   ├── model/
│   │   ├── Lista.java               # Record inmutable
│   │   ├── Item.java                # Record inmutable
│   │   ├── EstadoLista.java         # Enum: EN_PREPARACION, COMPLETADA
│   │   └── TipoUnidad.java          # Enum: 12 valores
│   ├── exception/
│   │   ├── ListaVaciaException.java
│   │   ├── NombreDuplicadoException.java
│   │   ├── TransicionInvalidaException.java
│   │   └── ListaNoEncontradaException.java
│   └── port/
│       ├── ListaRepository.java     # Interface (Port)
│       └── ItemRepository.java      # Interface (Port)
│
├── application/                     # ⚙️ CASOS DE USO
│   ├── usecase/
│   │   ├── lista/
│   │   │   ├── CrearListaUseCase.java
│   │   │   ├── ClonarListaUseCase.java
│   │   │   ├── CompletarListaUseCase.java
│   │   │   └── ReactivarListaUseCase.java
│   │   └── item/
│   │       ├── AgregarItemUseCase.java
│   │       ├── EditarItemUseCase.java
│   │       └── EliminarItemUseCase.java
│   └── dto/
│       ├── request/
│       │   ├── CrearListaRequest.java
│       │   ├── AgregarItemRequest.java
│       │   └── EditarItemRequest.java
│       └── response/
│           ├── ListaResponse.java
│           └── ItemResponse.java
│
├── infrastructure/                  # 🔌 ADAPTADORES
│   ├── persistence/
│   │   ├── MongoListaRepository.java   # Implementa domain.port.ListaRepository
│   │   ├── MongoItemRepository.java    # Implementa domain.port.ItemRepository
│   │   └── document/
│   │       ├── ListaDocument.java      # @Document("listas")
│   │       └── ItemDocument.java       # @Document("items")
│   ├── web/
│   │   ├── ListaController.java        # @RestController /api/v1/listas
│   │   ├── ItemController.java         # @RestController /api/v1/listas/{id}/items
│   │   └── GlobalExceptionHandler.java # @ControllerAdvice
│   └── logging/
│       └── RequestLoggingInterceptor.java
│
└── GroceryGuardApplication.java     # @SpringBootApplication
```
