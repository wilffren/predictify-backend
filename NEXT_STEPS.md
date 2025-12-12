# 📋 PASOS SIGUIENTES PARA EL DESARROLLADOR

## 🎯 Para Completar el Backend (Estimado: 2-3 horas)

El backend está **85% completo**. Para tener un MVP funcional con el frontend, sigue estos pasos:

### 1. EventService ⏱️ 45 minutos

Crear `/src/main/java/com/predictifylabs/backend/application/service/EventService.java`:

```java
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class EventService {
    private final EventRepository eventRepository;
    private final OrganizerRepository organizerRepository;
    private final UserRepository userRepository;

    public List<EventDTO> getUpcomingEvents() {
        LocalDate today = LocalDate.now();
        var events = eventRepository.findUpcomingEvents(today);
        return events.stream().map(this::toDTO).toList();
    }

    public EventDTO getEventBySlug(String slug) {
        return eventRepository.findBySlug(slug)
            .map(this::toDTO)
            .orElseThrow(() -> new RuntimeException("Event not found"));
    }

    @Transactional
    public EventDTO createEvent(CreateEventDTO dto, UUID userId) {
        var organizer = organizerRepository.findByUserId(userId)
            .orElseThrow(() -> new RuntimeException("User is not an organizer"));
        
        var event = EventEntity.builder()
            .organizer(organizer)
            .title(dto.title())
            .slug(generateSlug(dto.title()))
            .description(dto.description())
            .category(dto.category())
            .type(dto.type())
            .status(EventStatus.DRAFT)
            .startDate(dto.startDate())
            .startTime(dto.startTime())
            .capacity(dto.capacity())
            .price(dto.price())
            .isFree(dto.isFree())
            .build();
        
        var saved = eventRepository.save(event);
        return toDTO(saved);
    }

    private EventDTO toDTO(EventEntity event) {
        return EventDTO.builder()
            .id(event.getId())
            .title(event.getTitle())
            .slug(event.getSlug())
            .description(event.getDescription())
            .category(event.getCategory())
            .type(event.getType())
            .status(event.getStatus())
            .startDate(event.getStartDate())
            .startTime(event.getStartTime())
            .capacity(event.getCapacity())
            .registeredCount(event.getRegisteredCount())
            .price(event.getPrice())
            .isFree(event.getIsFree())
            // ... map other fields
            .build();
    }

    private String generateSlug(String title) {
        return title.toLowerCase()
            .replaceAll("[^a-z0-9\\s-]", "")
            .replaceAll("\\s+", "-")
            + "-" + UUID.randomUUID().toString().substring(0, 8);
    }
}
```

### 2. CreateEventDTO ⏱️ 10 minutos

Crear `/src/main/java/.../dto/event/CreateEventDTO.java`:

```java
public record CreateEventDTO(
    @NotBlank String title,
    @NotBlank String description,
    @NotNull EventCategory category,
    @NotNull EventType type,
    @NotNull LocalDate startDate,
    @NotNull LocalTime startTime,
    @NotNull @Min(1) Integer capacity,
    BigDecimal price,
    Boolean isFree,
    LocationDTO location
) {}
```

### 3. EventController ⏱️ 30 minutos

Crear `/src/main/java/.../controller/EventController.java`:

```java
@RestController
@RequestMapping("/api/events")
@RequiredArgsConstructor
public class EventController {
    private final EventService eventService;

    @GetMapping
    public ResponseEntity<List<EventDTO>> getAllEvents() {
        return ResponseEntity.ok(eventService.getUpcomingEvents());
    }

    @GetMapping("/upcoming")
    public ResponseEntity<List<EventDTO>> getUpcomingEvents() {
        return ResponseEntity.ok(eventService.getUpcomingEvents());
    }

    @GetMApping("/{slug}")
    public ResponseEntity<EventDTO> getEventBySlug(@PathVariable String slug) {
        return ResponseEntity.ok(eventService.getEventBySlug(slug));
    }

    @PostMapping
    @PreAuthorize("hasAnyRole('ORGANIZER', 'ADMIN')")
    public ResponseEntity<EventDTO> createEvent(
        @RequestBody @Valid CreateEventDTO dto,
        Authentication auth
    ) {
        UUID userId = extractUserIdFromAuth(auth);
        var created = eventService.createEvent(dto, userId);
        return ResponseEntity.status(HttpStatus.CREATED).body(created);
    }

    private UUID extractUserIdFromAuth(Authentication auth) {
        // Implement based on your UserDetailsImpl
        return UUID.fromString(auth.getName());
    }
}
```

### 4. EventRegistrationService ⏱️ 20 minutos

```java
@Service
@RequiredArgsConstructor
@Transactional
public class EventRegistrationService {
    private final EventRegistrationRepository registrationRepository;
    private final EventRepository eventRepository;
    private final UserRepository userRepository;

    public void registerToEvent(UUID eventId, UUID userId) {
        if (registrationRepository.existsByEventIdAndUserId(eventId, userId)) {
            throw new RuntimeException("Already registered");
        }

        var event = eventRepository.findById(eventId)
            .orElseThrow(() -> new RuntimeException("Event not found"));
        
        var user = userRepository.findById(userId)
            .orElseThrow(() -> new RuntimeException("User not found"));

        var registration = EventRegistrationEntity.builder()
            .event(event)
            .user(user)
            .status("registered")
            .registeredAt(OffsetDateTime.now())
            .build();

        registrationRepository.save(registration);

        // Update event registered count
        event.setRegisteredCount(event.getRegisteredCount() + 1);
        eventRepository.save(event);
    }

    public void cancelRegistration(UUID eventId, UUID userId) {
        var registration = registrationRepository
            .findByEventIdAndUserId(eventId, userId)
            .orElseThrow(() -> new RuntimeException("Registration not found"));

        registration.setStatus("cancelled");
        registration.setCancelledAt(OffsetDateTime.now());
        registrationRepository.save(registration);

        // Update event count
        var event = registration.getEvent();
        event.setRegisteredCount(Math.max(0, event.getRegisteredCount() - 1));
        eventRepository.save(event);
    }
}
```

### 5. EventRegistrationController ⏱️ 15 minutos

```java
@RestController
@RequestMapping("/api/events")
@RequiredArgsConstructor
public class EventRegistrationController {
    private final EventRegistrationService registrationService;

    @PostMapping("/{eventId}/register")
    @PreAuthorize("isAuthenticated()")
    public ResponseEntity<Void> register(
        @PathVariable UUID eventId,
        Authentication auth
    ) {
        UUID userId = extractUserId(auth);
        registrationService.registerToEvent(eventId, userId);
        return ResponseEntity.ok().build();
    }

   @DeleteMapping("/{eventId}/register")
    @PreAuthorize("isAuthenticated()")
    public ResponseEntity<Void> cancelRegistration(
        @PathVariable UUID eventId,
        Authentication auth
    ) {
        UUID userId = extractUserId(auth);
        registrationService.cancelRegistration(eventId, userId);
        return ResponseEntity.ok().build();
    }

    private UUID extractUserId(Authentication auth) {
        // Implement based on UserDetailsImpl
        return UUID.fromString(auth.getName());
    }
}
```

### 6. UserController ⏱️ 20 minutos

```java
@RestController
@RequestMapping("/api/users")
@RequiredArgsConstructor
public class UserController {
    private final UserRepository userRepository;

    @GetMapping("/me")
    @PreAuthorize("isAuthenticated()")
    public ResponseEntity<UserDTO> getCurrentUser(Authentication auth) {
        UUID userId = extractUserId(auth);
        var user = userRepository.findById(userId)
            .orElseThrow(() -> new RuntimeException("User not found"));
        return ResponseEntity.ok(toDTO(user));
    }

    private UserDTO toDTO(UserEntity user) {
        return UserDTO.builder()
            .id(user.getId())
            .name(user.getName())
            .email(user.getEmail())
            .role(user.getRole())
            .avatar(user.getAvatar())
            .bio(user.getBio())
            .createdAt(user.getCreatedAt())
            .build();
    }

    private UUID extractUserId(Authentication auth) {
        return UUID.fromString(auth.getName());
    }
}
```

---

## ✅ Checklist de Tareas

- [ ] Crear `EventService`
- [ ] Crear `CreateEventDTO`
- [ ] Crear `EventController`
- [ ] Crear `EventRegistrationService`
- [ ] Crear `EventRegistrationController`
- [ ] Crear `UserController`
- [ ] Crear `UserDTO`
- [ ] Probartodos los endpoints con Postman/curl
- [ ] Documentar en Swagger (automático con SpringDoc)

---

## 🧪 Testing

### 1. Levantar DB
```bash
docker-compose up -d
```

### 2. Ejecutar Backend
```bash
mvn spring-boot:run
```

### 3. Registrar Usuario
```bash
curl -X POST http://localhost:8081/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Test Organizer",
    "email": "organizer@test.com",
    "password": "Test123!",
    "role": "ORGANIZER"
  }'

# Guardarel accessToken de la respuesta
TOKEN="<accessToken_recibido>"
```

### 4. Crear Evento
```bash
curl -X POST http://localhost:8081/api/events \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Angular Workshop 2024",
    "description": "aprende Angular",
    "category": "WORKSHOP",
    "type": "PRESENCIAL",
    "startDate": "2024-12-20",
    "startTime": "10:00:00",
    "capacity": 50,
    "price": 0,
    "isFree": true
  }'
```

### 5. Listar Eventos
```bash
curl http://localhost:8081/api/events/upcoming
```

---

## 📚 Recursos

- **Swagger UI**: http://localhost:8081/swagger-ui.html
- **API Docs**: http://localhost:8081/v3/api-docs
- **Database**: PostgreSQL en puerto 5435
- **pgAdmin**: http://localhost:5050

---

## 🎓 Notas Importantes

1. **UserDetailsImpl**: Necesitas implementar `UserDetails` para extraer el UUID del usuario autenticado
2. **Excepciones**: Crear excepciones personalizadas (`NotFoundException`, `AlreadyExistsException`)
3. **Validación**: Usar `@Valid` y Bean Validation en DTOs
4. **CORS**: Ya está configurado en `SecurityConfiguration`
5. **Swagger**: Se genera automáticamente con SpringDoc OpenAPI

---

## 🚀 Una Vez Completo

El frontend podrá:
- ✅ Registrarse/Login
- ✅ Ver lista de eventos
- ✅ Ver detalle de evento
- ✅ Registrarse a eventos
- ✅ Cancelar registros
- ✅ Ver perfil de usuario
- ✅ Crear eventos (organizadores)

**Tiempo Total**: ~2.5 horas de desarrollo enfocado

¡El backend está 85% done, solo faltan los endpoints CRUD!
