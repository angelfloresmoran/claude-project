# Platziflix - Proyecto Multi-plataforma

## Reglas de Desarrollo (LEER PRIMERO)

1. **Docker obligatorio** para el backend. Antes de ejecutar cualquier comando de backend, verifica que el contenedor esté corriendo con `docker ps` y usa los comandos del Makefile (`make start`, `make migrate`, etc.) — nunca ejecutes Python directamente.
2. **API REST** es la única fuente de datos para Frontend y Mobile. No hay estado compartido entre clientes.
3. **Soft deletes globales**: nunca uses `DELETE` SQL directo. Todos los modelos tienen `deleted_at`; filtrar siempre con `.filter(Model.deleted_at.is_(None))`.
4. **Testing requerido** para nuevas funcionalidades en los tres proyectos.
5. **Migraciones Alembic** para cualquier cambio de schema de BD (`make create-migration` + `make migrate`).
6. **Convenciones de naming**: `snake_case` (Python), `camelCase` (JS/TS), `PascalCase` (Swift/Kotlin).
7. **TypeScript strict** en Frontend — no usar `any`.

---

## Arquitectura General

```
┌─────────────────────────────────────────────────────────────────┐
│  WEB (Next.js 15)    ANDROID (Kotlin)       iOS (Swift)         │
│  localhost:3000       Native App             Native App          │
└──────────┬───────────────────┬──────────────────────┬───────────┘
           │                   │                      │
           └───────────────────┴──────────────────────┘
                               │ HTTP REST JSON
                               ▼
           ┌──────────────────────────────────────┐
           │   BACKEND FastAPI — localhost:8000   │
           └───────────────────┬──────────────────┘
                               │ SQLAlchemy 2.0 ORM
                               ▼
           ┌──────────────────────────────────────┐
           │   PostgreSQL 15 (Docker) — port 5432 │
           └──────────────────────────────────────┘
```

**Principio clave**: el Backend es la única fuente de verdad. Frontend y Mobile consumen la misma API REST; no se comunican entre sí.

---

## Stack Tecnológico

### Backend
- **Framework**: FastAPI
- **ORM**: SQLAlchemy 2.0
- **Migraciones**: Alembic
- **BD**: PostgreSQL 15 (vía Docker)
- **Dependencias**: UV (pyproject.toml)
- **Puerto**: 8000

### Frontend
- **Framework**: Next.js 15 (App Router)
- **React**: 19.0 — TypeScript strict
- **Estilos**: SCSS + CSS Modules
- **Testing**: Vitest + React Testing Library
- **Fuentes**: Geist Sans & Geist Mono

### Mobile
- **Android**: Kotlin + Jetpack Compose + Retrofit + Hilt (DI)
- **iOS**: Swift + SwiftUI + URLSession

---

## Estructura de Archivos Clave

### Backend (`Backend/app/`)
```
main.py                     ← Entrypoint: rutas HTTP + inyección de dependencias
core/config.py              ← Settings (nombre, versión, DATABASE_URL)
db/base.py                  ← engine, SessionLocal, get_db()
db/seed.py                  ← Datos de prueba
models/base.py              ← BaseModel: id, created_at, updated_at, deleted_at
models/course.py            ← Course (name, slug, thumbnail, description, ratings)
models/teacher.py           ← Teacher
models/lesson.py            ← Lesson (video_url, slug, course_id FK)
models/class_.py            ← Class
models/course_teacher.py    ← Tabla pivot Many-to-Many Course ↔ Teacher
models/course_rating.py     ← CourseRating (rating 1-5, soft delete, user_id)
schemas/rating.py           ← Pydantic: RatingRequest, RatingResponse, RatingStatsResponse
services/course_service.py  ← Service Layer: toda la lógica de negocio y queries
alembic/versions/           ← Migraciones (d18a... schema inicial, 0e3a... ratings)
```

### Frontend (`Frontend/src/`)
```
app/page.tsx                        ← Home: grid Netflix de cursos (Server Component)
app/course/[slug]/page.tsx          ← Detalle de curso (Server Component)
app/classes/[class_id]/page.tsx     ← Reproductor de video
components/Course/                  ← Tarjeta de curso (thumbnail + estrella rating)
components/CourseDetail/            ← Vista detalle (info + lista de clases)
components/StarRating/              ← Componente interactivo 1-5 estrellas
components/VideoPlayer/             ← Reproductor embebido
services/ratingsApi.ts              ← Cliente HTTP completo para ratings (CRUD)
types/index.ts                      ← Course, Class, CourseDetail, Progress, Quiz, FavoriteToggle
types/rating.ts                     ← CourseRating, RatingStats, RatingRequest, ApiError
```

### Mobile Android (`Mobile/PlatziFlixAndroid/app/src/main/java/.../`)
```
presentation/courses/viewmodel/CourseListViewModel.kt   ← MVVM/MVI con StateFlow
presentation/courses/screen/CourseListScreen.kt         ← UI Compose
presentation/courses/state/CourseListUiState.kt         ← Estado UI
presentation/courses/components/                        ← CourseCard, Loading, Error
domain/models/Course.kt                                 ← Modelo de dominio
domain/repositories/CourseRepository.kt                 ← Interfaz del repositorio
data/entities/CourseDTO.kt                              ← Entity de red (Retrofit)
data/mappers/CourseMapper.kt                            ← DTO → Domain
data/repositories/RemoteCourseRepository.kt             ← Implementación real (API)
data/repositories/MockCourseRepository.kt               ← Implementación mock (dev offline)
data/network/ApiService.kt                              ← Interface Retrofit
data/network/NetworkModule.kt                           ← Config Retrofit
di/AppModule.kt                                         ← Hilt DI Module
```

### Mobile iOS (`Mobile/PlatziFlixiOS/PlatziFlixiOS/`)
```
Presentation/ViewModels/CourseListViewModel.swift       ← MVVM con @Published
Presentation/Views/CourseListView.swift                 ← UI SwiftUI
Presentation/Views/CourseCardView.swift                 ← Tarjeta de curso
Presentation/Views/DesignSystem.swift                   ← Tokens de diseño
Domain/Models/Course.swift, Teacher.swift, Class.swift  ← Modelos de dominio
Domain/Repositories/CourseRepositoryProtocol.swift      ← Protocolo repositorio
Data/Entities/CourseDTO.swift, TeacherDTO.swift, ClassDetailDTO.swift ← Network entities
Data/Mapper/CourseMapper.swift, TeacherMapper.swift, ClassMapper.swift ← DTO → Domain
Data/Repositories/RemoteCourseRepository.swift          ← Implementación API
Data/Repositories/CourseAPIEndpoints.swift              ← Enum de endpoints
Services/NetworkManager.swift                           ← URLSession wrapper
Services/NetworkService.swift                           ← Protocolo de red
Services/APIEndpoint.swift                              ← Protocolo de endpoint
```

---

## Modelo de Datos

```
BaseModel (abstracto)
  └── id (PK), created_at, updated_at, deleted_at (soft delete)

Course ──────────────────────────────── CourseRating
  id, name, slug (unique+index),           id, course_id (FK), user_id,
  description, thumbnail (URL)             rating (INT, CHECK 1-5), deleted_at
       │
       ├── M:M via course_teachers ──── Teacher
       │                                  id, name, ...
       │
       └── 1:N ───────────────────────── Lesson
                                          id, course_id (FK), name,
                                          slug, description, video_url
```

**Regla de ratings**: un usuario solo puede tener un rating activo por curso. El `POST /courses/{id}/ratings` implementa lógica upsert: actualiza si ya existe, crea si no existe.

---

## API Endpoints Completos

### Cursos
| Método | Ruta | Descripción |
|--------|------|-------------|
| GET | `/` | Bienvenida |
| GET | `/health` | Estado del servicio + conectividad DB |
| GET | `/courses` | Lista de cursos (incluye `average_rating`, `total_ratings`) |
| GET | `/courses/{slug}` | Detalle con teachers, lessons y rating stats |
| GET | `/classes/{class_id}` | Detalle de clase/lección con `video_url` |

### Ratings
| Método | Ruta | Descripción |
|--------|------|-------------|
| POST | `/courses/{course_id}/ratings` | Crear o actualizar rating (upsert) |
| GET | `/courses/{course_id}/ratings` | Listar todos los ratings activos |
| GET | `/courses/{course_id}/ratings/stats` | Stats: `average_rating`, `total_ratings`, `rating_distribution` |
| GET | `/courses/{course_id}/ratings/user/{user_id}` | Rating de un usuario (204 si no existe) |
| PUT | `/courses/{course_id}/ratings/{user_id}` | Actualizar rating existente (falla si no existe) |
| DELETE | `/courses/{course_id}/ratings/{user_id}` | Soft delete del rating |

---

## Patrones de Desarrollo

### Backend
- **Capas**: `main.py` (HTTP) → `CourseService` (lógica) → SQLAlchemy models (datos)
- **DI**: `Depends(get_course_service)` inyecta `CourseService(db)` en cada request
- **Agregaciones en SQL**: las stats de ratings se calculan con `func.avg` / `func.count` en la BD, no en Python
- **Soft deletes**: siempre filtrar `.filter(Model.deleted_at.is_(None))` en queries

### Frontend
- **Server Components** por defecto: el data fetching ocurre en el servidor con `fetch()` directo
- **Client Components** solo para interactividad: `StarRating` usa estado del cliente
- **ratingsApi.ts**: servicio con `fetchWithTimeout` (10s), manejo de errores tipados con `ApiError`
- **Tipos futuros en `types/index.ts`**: `Progress`, `Quiz`, `QuizOption`, `FavoriteToggle` están definidos pero **no tienen endpoints en el backend todavía**

### Mobile (ambas plataformas)
- **Arquitectura en capas**: Presentation → Domain → Data (Clean Architecture)
- **Mappers explícitos**: los DTOs de red nunca se exponen al dominio; siempre se mapean
- **Android diferencial**: tiene `MockCourseRepository` para desarrollo sin API; usa Hilt para DI
- **iOS diferencial**: mappers más granulares (Teacher, Class, Course separados); usa URLSession directamente sin Retrofit

---

## Infraestructura Docker

```yaml
# Backend/docker-compose.yml
services:
  db:    postgres:15 → port 5432, volumen persistente
  api:   FastAPI     → port 8000, volume mount ./app (hot-reload)
         env DATABASE_URL: postgresql://platziflix_user:platziflix_password@db:5432/platziflix_db
```

---

## Comandos de Desarrollo

### Backend
```bash
cd Backend
make start           # Levantar Docker Compose (db + api)
make stop            # Detener containers
make logs            # Ver logs de todos los servicios
make migrate         # Aplicar migraciones Alembic
make create-migration # Crear nueva migración
make seed            # Poblar datos de prueba
make seed-fresh      # Reset completo + seed
```

### Frontend
```bash
cd Frontend
yarn dev             # Servidor de desarrollo (localhost:3000)
yarn build           # Build de producción
yarn test            # Ejecutar tests (Vitest)
yarn lint            # ESLint
```

---

## URLs del Sistema

| Servicio | URL |
|---------|-----|
| Frontend Web | http://localhost:3000 |
| Backend API | http://localhost:8000 |
| API Docs (Swagger) | http://localhost:8000/docs |
| PostgreSQL | localhost:5432 |

---

## Funcionalidades Implementadas

- ✅ Catálogo de cursos con grid estilo Netflix
- ✅ Detalle de curso (profesores, lecciones, clases)
- ✅ Navegación por slug SEO-friendly
- ✅ Reproductor de video integrado
- ✅ Sistema de ratings completo (CRUD + stats + distribución)
- ✅ Health checks de API y DB
- ✅ Apps móviles nativas (Android + iOS)
- ✅ Testing en Frontend y Backend

## Funcionalidades Pendientes (tipos ya definidos en Frontend)

- ⬜ Progreso de usuario (`Progress` en `types/index.ts`)
- ⬜ Quizzes por clase (`Quiz`, `QuizOption` en `types/index.ts`)
- ⬜ Favoritos (`FavoriteToggle` en `types/index.ts`)
- ⬜ Rating interactivo en Mobile (actualmente solo Web tiene StarRating)
