# Predictify Backend - Sistema de Gestión de Eventos con IA

API REST para gestión de eventos con predicciones de asistencia basadas en Inteligencia Artificial.

## 🏗️ Arquitectura

- **Framework**: Spring Boot 3.4.0
- **Base de Datos**: PostgreSQL 17
- **Autenticación**: JWT (JSON Web Tokens)
- **ORM**: Hibernate/JPA con UUID como PKs
- **Documentación**: OpenAPI/Swagger UI

## 🚀 Levantar el Proyecto

### 1. Iniciar Base de Datos

```bash
docker-compose up -d
```

Esto iniciará:
- PostgreSQL 17 en puerto 5435
- pgAdmin en http://localhost:5050

### 2. Ejecutar Backend

```bash
mvn spring-boot:run
```

La API estará disponible en: **http://localhost:8081**

### 3. Documentación API

Swagger UI: **http://localhost:8081/swagger-ui.html**

## 📡 Endpoints Principales

### Autenticación

| Método | Endpoint | Descripción |
|--------|----------|-------------|
| POST | `/api/auth/register` | Registrar nuevo usuario |
| POST | `/api/auth/login` | Iniciar sesión |

### Eventos

| Método | Endpoint | Descripción | Rol Requerido |
|--------|----------|-------------|---------------|
| GET | `/api/events` | Listar eventos publicados | Público |
| GET | `/api/events/upcoming` | Eventos próximos | Público |
| GET | `/api/events/featured` | Eventos destacados | Público |
| GET | `/api/events/trending` | Eventos en tendencia | Público |
| GET | `/api/events/{slug}` | Detalle de evento | Público |
| POST | `/api/events` | Crear evento | ORGANIZER, ADMIN |
| PUT | `/api/events/{id}` | Actualizar evento | ORGANIZER (propio), ADMIN |
| DELETE | `/api/events/{id}` | Eliminar evento | ORGANIZER (propio), ADMIN |
| POST | `/api/events/{id}/publish` | Publicar evento | ORGANIZER (propio), ADMIN |

### Registros de Asistencia

| Método | Endpoint | Descripción | Rol Requerido |
|--------|----------|-------------|---------------|
| POST | `/api/events/{eventId}/register` | Registrarse a evento | ATTENDEE, ORGANIZER, ADMIN |
| DELETE | `/api/events/{eventId}/register` | Cancelar registro | ATTENDEE, ORGANIZER, ADMIN |
| GET | `/api/events/{eventId}/registrations` | Ver registros | ORGANIZER (propio), ADMIN |
| POST | `/api/events/{eventId}/interested` | Marcar interés | ATTENDEE, ORGANIZER, ADMIN |

### Predicciones

| Método | Endpoint | Descripción | Rol Requerido |
|--------|----------|-------------|---------------|
| GET | `/api/events/{id}/prediction` | Obtener predicción | ATTENDEE, ORGANIZER, ADMIN |
| POST | `/api/events/{id}/prediction/generate` | Generar predicción | ORGANIZER (propio), ADMIN |

### Usuarios

| Método | Endpoint | Descripción | Rol Requerido |
|--------|----------|-------------|---------------|
| GET | `/api/users/me` | Perfil del usuario actual | Autenticado |
| PUT | `/api/users/me` | Actualizar perfil | Autenticado |
| GET | `/api/users/me/events` | Mis eventos registrados | Autenticado |

### Organizadores

| Método | Endpoint | Descripción | Rol Requerido |
|--------|----------|-------------|---------------|
| POST | `/api/organizers/profile` | Crear perfil organizador | ORGANIZER, ADMIN |
| GET | `/api/organizers/{id}` | Ver perfil organizador | Público |

## 🔐 Autenticación

Todos los endpoints protegidos requieren el header:

```
Authorization: Bearer {token}
```

### Ejemplo de Login

```bash
curl -X POST http://localhost:8081/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "user@example.com",
    "password": "password123"
  }'
```

Respuesta:
```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiIs...",
  "refreshToken": "eyJhbGciOiJIUzI1NiIs..."
}
```

## 👥 Roles de Usuario

### ATTENDEE (Usuario Normal)
- Ver eventos publicados
- Registrarse a eventos
- Guardar eventos favoritos
- Ver predicciones

### ORGANIZER (Organizador)
- Todo lo de ATTENDEE
- Crear y gestionar eventos propios
- Ver analytics de eventos propios
- Generar predicciones

### ADMIN (Administrador)
- Acceso total al sistema
- Gestionar cualquier evento
- Verificar organizadores
- Ver analytics globales

## 🗄️ Base de Datos

El esquema completo se encuentra en `src/main/resources/schema.sql`

### Tablas Principales
- **users**: Gestión de usuarios
- **events**: Eventos del sistema
- **event_registrations**: Registros de asistencia
- **event_predictions**: Predicciones de IA
- **organizers**: Perfiles de organizadores
- **permissions**: Sistema de permisos
- **protected_routes**: Configuración de rutas frontend

Total: 33+ tablas con relaciones complejas

## 🧪 Datos de Prueba

El archivo `schema.sql` incluye:
- 14 fact ores de predicción predefinidos
- 40+ permisos del sistema
- 20+ rutas protegidas configuradas

## 🌐 Integración con Frontend Angular

### Headers Requeridos

```typescript
const headers = {
  'Content-Type': 'application/json',
  'Authorization': `Bearer ${token}`
};
```

### Servicios Recomendados

```typescript
// auth.service.ts
class AuthService {
  login(email: string, password: string): Observable<AuthResponse>
  register(data: RegisterRequest): Observable<AuthResponse>
  logout(): void
}

// events.service.ts
class EventsService {
  getUpcomingEvents(): Observable<Event[]>
  getEventBySlug(slug: string): Observable<Event>
  registerToEvent(eventId: string): Observable<void>
}

// permissions.service.ts
class PermissionsService {
  getUserPermissions(): Observable<string[]>
  hasPermission(permission: string): boolean
}
```

### Guards para Rutas

```typescript
@Injectable()
export class AuthGuard implements CanActivate {
  canActivate(): boolean {
    return !!this.authService.getToken();
  }
}

@Injectable()
export class RoleGuard implements CanActivate {
  canActivate(route: ActivatedRouteSnapshot): boolean {
    const requiredRoles = route.data['roles'];
    return this.authService.hasRole(requiredRoles);
  }
}
```

## 📊 Predicciones con IA

El sistema utiliza múltiples factores para predecir asistencia:

### Factores Positivos
- Alto engagement (interesados vs capacidad)
- Tema trending
- Organizador verificado  
- Registros anticipados
- Buen historial del organizador

### Factores Negativos
- Evento gratuito (mayor no-show)
- Eventos competidores en la misma fecha
- Mal clima previsto
- Temporada de vacaciones

## 🔧 Configuración

### Variables de Entorno

```properties
# Database
SPRING_DATASOURCE_URL=jdbc:postgresql://localhost:5435/predictify_db
SPRING_DATASOURCE_USERNAME=postgres
SPRING_DATASOURCE_PASSWORD=secret

# JWT
JWT_SECRET=your-secret-key
JWT_EXPIRATION=86400000

# Gemini AI
GEMINI_API_KEY=your-api-key
```

## 📝 Modelos de Datos (DTOs)

### RegisterRequest
```json
{
  "name": "John Doe",
  "email": "john@example.com",
  "password": "securePassword123",
  "role": "ATTENDEE"
}
```

### EventCreateRequest
```json
{
  "title": "Angular Workshop 2024",
  "description": "Aprende Angular desde cero",
  "category": "WORKSHOP",
  "type": "PRESENCIAL",
  "startDate": "2024-12-20",
  "startTime": "10:00:00",
  "capacity": 50,
  "price": 0,
  "isFree": true,
  "location": {
    "type": "PHYSICAL",
    "city": "Bogotá",
    "country": "Colombia",
    "address": "Calle 123 #45-67"
  }
}
```

## 🐛 Troubleshooting

### Base de Datos no Inicia
```bash
docker-compose down
docker volume rm predictify-backend_postgres_data_secure
docker-compose up -d
```

### Puerto 8081 en Uso
Cambiar en `application.yml`:
```yaml
server:
  port: 8082
```

### Compilación Falla
```bash
mvn clean install -DskipTests
```

## 📚 Stack Tecnológico

- Java 21
- Spring Boot 3.4.0
- Spring Security + JWT
- Spring Data JPA
- PostgreSQL 17
- Hibernate
- Lombok
- OpenAPI/Swagger
- Maven
- Docker & Docker Compose

## 🤝 Contribución

1. Fork el proyecto
2. Crea una rama para tu feature (`git checkout -b feature/AmazingFeature`)
3. Commit tus cambios (`git commit -m 'Add some AmazingFeature'`)
4. Push a la rama (`git push origin feature/AmazingFeature`)
5. Abre un Pull Request

## 📄 Licencia

Este proyecto está bajo la Licencia MIT.

## 👨‍💻 Autor

**PredictifyLabs Team**

---

**Nota**: Este backend está diseñado para trabajar con un frontend Angular que consume estos endpoints.
