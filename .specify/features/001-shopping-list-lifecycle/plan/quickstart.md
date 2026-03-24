# Quickstart — Grocery Guard v1.0

**Para**: Desarrolladores nuevos que se suman al proyecto
**Objetivo**: Entender y correr el sistema en menos de 30 minutos

---

## Prerequisitos

| Herramienta | Versión mínima | Verificar con |
|-------------|---------------|---------------|
| Java JDK | 21 | `java -version` |
| Maven | 3.9+ | `mvn -version` |
| Docker | 24+ | `docker -v` |
| Node.js | 20+ | `node -v` |
| npm | 10+ | `npm -v` |

---

## 1. Levantar MongoDB con Docker

```bash
docker run -d \
  --name grocery-mongo \
  -p 27017:27017 \
  -e MONGO_INITDB_DATABASE=grocery_guard \
  mongo:7
```

Verificar que corre:
```bash
docker ps | grep grocery-mongo
```

---

## 2. Backend (Spring Boot)

```bash
# Desde la raíz del proyecto backend
cd backend/

# Compilar y ejecutar
mvn spring-boot:run

# La app estará disponible en:
# API:     http://localhost:8080/api/v1
# Swagger: http://localhost:8080/swagger-ui.html
```

**Configuración** (`src/main/resources/application.properties`):
```properties
spring.data.mongodb.uri=mongodb://localhost:27017/grocery_guard
spring.data.mongodb.auto-index-creation=true
server.port=8080
```

---

## 3. Frontend (React + Vite)

```bash
# Desde la raíz del proyecto frontend
cd frontend/

# Instalar dependencias
npm install

# Levantar en modo desarrollo
npm run dev

# Disponible en: http://localhost:5173
```

---

## 4. Verificar que todo funciona

1. Abre **Swagger UI**: http://localhost:8080/swagger-ui.html
2. Crea una lista: `POST /api/v1/listas` con `{ "titulo": "Test" }`
3. Agrega un ítem: `POST /api/v1/listas/{id}/items` con `{ "nombre": "Leche", "cantidad": 2, "tipo_unidad": "LITRO" }`
4. Completa la lista: `PATCH /api/v1/listas/{id}/completar`
5. Abre el Frontend en http://localhost:5173 y navega el flujo

---

## 5. Correr Tests

```bash
# Backend — unit tests + coverage report
cd backend/
mvn test jacoco:report

# El reporte de cobertura estará en:
# target/site/jacoco/index.html

# Objetivo: Domain ≥ 90% | Application ≥ 80%
```

---

## Flujo de datos en 5 pasos

```
Usuario → React (presenta) → REST API (controla) → UseCase (decide) → MongoDB (persiste)
                                         ↑
                              Toda la lógica vive aquí
```

> La única fuente de verdad es el **API**. El Frontend solo pinta lo que el Backend autoriza.

---

## Estructura del Repositorio (esperada)

```
listadecompras/
├── backend/          ← Java Spring Boot (Onion Architecture)
├── frontend/         ← React + Vite
├── .specify/         ← Artefactos de especificación y plan
│   ├── memory/       ← Constitución del proyecto
│   └── features/001-shopping-list-lifecycle/
│       ├── spec.md
│       ├── overview.md
│       ├── stories/
│       └── plan/     ← Este directorio
└── README.md
```

---

## Dónde leer más

| Documento | Para qué |
|-----------|---------|
| `spec.md` | Entender QUÉ construimos y por qué |
| `overview.md` | Vista rápida con diagramas Mermaid |
| `plan/research.md` | Por qué elegimos cada tecnología |
| `plan/data-model.md` | Modelo de datos MongoDB + índices |
| `plan/contracts/api-contracts.md` | Todos los endpoints con ejemplos |
| `plan/docs/technical-architecture.md` | Diagramas de componentes, secuencia, despliegue |
| `Swagger UI` | Explorar y probar el API en vivo |
