# Copilot Instructions for Car Rental Backend

## Project Overview
Spring Boot 3.4.10 REST API for car rental system with JWT-based authentication. Java 21, Maven-built, MySQL backend, containerized with Docker.

**Key Services:**
- `UserService`: Handles registration and login with password encryption and JWT token generation
- `AuthController`: Public endpoints for `/auth/signup` and `/auth/login`
- `SecurityConfig`: Stateless session management, CORS enabled, JWT filter placeholder

## Architecture & Data Flow

### Authentication Flow
1. User submits credentials to `/auth/login` → `AuthController`
2. `UserService.loginUser()` validates password using `BCryptPasswordEncoder` against `User` entity
3. `JwtUtil.generateToken()` creates HS256-signed JWT (24hr expiration)
4. Token returned to client for subsequent API calls

### Security Configuration
- **SessionCreationPolicy.STATELESS**: No session state; JWT in Authorization header
- **Password**: BCrypt hashed, validated via `PasswordEncoder.matches()`
- **JWT Secret**: Hardcoded in `JwtUtil` (line 11) - **move to environment variable in production**
- **CORS**: Wildcard enabled (`*`) - restrict to specific origins in production

### Package Structure
```
com.klu.carrental/
├── controller/       # REST endpoints (@RestController)
├── entity/          # JPA entities (User model)
├── repository/      # Spring Data JPA interfaces
├── security/        # JWT, config, authentication details
└── service/         # Business logic (UserService)
```

## Critical Implementation Patterns

### Adding New Endpoints
1. Create controller method in `controller/` package with `@PostMapping` or `@GetMapping`
2. If protected: endpoint requires JWT validation (SecurityConfig line 38-44 allows public paths)
3. Add new public paths to `SecurityConfig.securityFilterChain()` `.requestMatchers()` call if needed
4. Service layer handles business logic; controller calls service methods

**Example:** New endpoint in `AuthController`:
```java
@GetMapping("/profile")
public ResponseEntity<?> getUserProfile(@RequestHeader("Authorization") String token) {
    String username = jwtUtil.extractUsername(token.substring(7)); // Remove "Bearer "
    return ResponseEntity.ok(userService.getUserProfile(username));
}
```

### Database Integration
- **ORM**: Spring Data JPA with MySQL connector
- **Auto-schema**: `spring.jpa.hibernate.ddl-auto=update` (auto-creates/updates tables)
- **Repositories**: Extend `JpaRepository<Entity, ID>` in `repository/` package
- **Example:** `UserRepository.findByUsername()` - query method auto-implemented by Spring

### Error Handling
- Current: Service throws `RuntimeException` (e.g., `"User already exists!"`)
- Pattern: Catch exceptions in controller, return `ResponseEntity.badRequest()` or `.status(HttpStatus.CONFLICT)`
- Recommendation: Create custom exception classes in `exception/` package for better API responses

## Build & Deployment

### Maven Commands
```bash
./mvnw clean install      # Full build (runs tests)
./mvnw -DskipTests clean install  # Skip tests
./mvnw spring-boot:run    # Run locally (http://localhost:8081)
```

### Docker Build & Run
```bash
# Build
docker build -t carrental:latest .

# Run with MySQL
docker run -p 8081:8081 \
  -e SPRING_DATASOURCE_URL=jdbc:mysql://host.docker.internal:3306/carrental \
  -e SPRING_DATASOURCE_USERNAME=root \
  -e SPRING_DATASOURCE_PASSWORD=root \
  carrental:latest
```

### Database Setup
```sql
CREATE DATABASE carrental;
-- Tables auto-created by Hibernate (ddl-auto=update)
```

## Configuration

### application.properties
- **Port**: 8081 (server.port)
- **MySQL**: localhost:3306/carrental (credentials: root/root)
- **JPA**: `show-sql=true` logs SQL (disable in production)

### Environment Variables (Production)
```properties
SPRING_DATASOURCE_URL=jdbc:mysql://prod-host:3306/carrental
SPRING_DATASOURCE_USERNAME=prod_user
SPRING_DATASOURCE_PASSWORD=<secure_password>
JWT_SECRET=<your-production-secret-key-min-32-chars>
```

## Project-Specific Conventions

1. **Package Naming**: `com.klu.carrental.*` - maintain this structure
2. **Dependency Injection**: Use constructor injection (all services use `@Autowired`-style constructors)
3. **HTTP Status Codes**: 
   - 200: Success
   - 401: Invalid credentials
   - 409: User already exists
   - 400: Invalid input
4. **Request/Response**: Simple `Map<String, String>` or typed DTOs (not yet implemented, create in `dto/` package for complex objects)
5. **CORS**: Currently `@CrossOrigin(origins = "*")` on `AuthController` - use Spring Security CORS config for global control

## Common Workflows for Agents

### Adding a New Feature
1. Create entity in `entity/` package if new data model
2. Create repository interface in `repository/` (extend `JpaRepository`)
3. Add service methods in `service/` for business logic
4. Create controller endpoint in `controller/`
5. Update `SecurityConfig` if endpoint is public or has special permissions
6. Test with `./mvnw spring-boot:run` locally

### Debugging
- Logs: Check console output from `spring-boot:run` or Docker logs
- Database: Use MySQL CLI to verify data: `SELECT * FROM user;`
- JWT Issues: Decode token at jwt.io or check `JwtUtil.validateToken()` logic
- Security: Review `SecurityConfig.securityFilterChain()` for endpoint permissions

### Testing
- Run: `./mvnw test`
- Location: `src/test/java/` (currently has placeholder `EcommerceApplicationTests`)
- Framework: JUnit 5 (included in `spring-boot-starter-test`)

## Known Issues & TODO

1. **JWT Secret**: Hardcoded in `JwtUtil.java` line 11 - externalize to config/environment variable
2. **JWT Filter**: `SecurityConfig` references JWT filter in comment (line 49) but not implemented - add `JwtAuthenticationFilter`
3. **Exception Handling**: No global exception handler - implement `@RestControllerAdvice` in `exception/` package
4. **DTO Pattern**: Using `Map<String, String>` instead of typed request/response objects
5. **SQL Injection Risk**: User input not validated before DB queries in `UserRepository` query methods

## Dependencies
- Spring Boot 3.4.10 (Spring 6.x)
- Spring Security 6.x (SecurityConfig syntax is 6.x compatible)
- JJWT 0.11.5 (JWT generation/parsing)
- MySQL Connector/J 8.x
- JPA/Hibernate (auto-schema management)
