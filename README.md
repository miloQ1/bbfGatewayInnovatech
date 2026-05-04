# BFF Gateway — Innovatech Solutions

Punto de entrada único para el frontend React. Valida JWT y enruta los requests a los microservicios correspondientes.

---

## ¿Qué es un API Gateway?

Un **API Gateway** es un proxy inteligente que se ubica entre el cliente y los microservicios. Su trabajo es recibir todos los requests del frontend y redirigirlos al microservicio correcto.

```
Sin Gateway:
Browser → authService      :8080
Browser → proyectsService  :8081
Browser → resourcesService :8083
Browser → ...              :808X

Con Gateway:
Browser → Gateway :8090 → authService      :8080
                        → proyectsService  :8081
                        → resourcesService :8083
```

**Ventajas:**
- El frontend solo conoce una URL — el Gateway
- Los microservicios quedan en red interna, no expuestos al exterior
- La autenticación se centraliza en un solo punto
- Fácil agregar nuevos microservicios sin tocar el frontend

---

## ¿Qué es un BFF?

**BFF (Backend For Frontend)** es un patrón que va un paso más allá del Gateway. Además de enrutar, el BFF está diseñado específicamente para las necesidades del frontend que lo consume.

```
API Gateway puro:
Browser → Gateway → microservicio A
                  → microservicio B

BFF:
Browser → BFF → llama A + B + C en paralelo → combina → devuelve al browser
```

Un BFF puede agregar datos de múltiples servicios en una sola respuesta, transformar el formato de los datos, o adaptar la API exactamente a lo que el frontend necesita.

---

## Este proyecto: híbrido BFF + API Gateway

Este servicio implementa ambos patrones:

- **Como API Gateway**: valida JWT y enruta cada request al microservicio correcto
- **Como BFF**: está diseñado específicamente para el frontend React de Innovatech e inyecta contexto del usuario (`X-User-Id`, `X-User-Name`) en cada request

En el futuro puede extenderse para agregar datos de múltiples servicios en una sola respuesta (comportamiento BFF completo).

---

## Arquitectura

```
React Frontend (Vite :5173)
        │
        ▼
  BFF Gateway :8090
        │
        ├── /api/auth/**    ──► authService      :8080  (libre, sin JWT)
        ├── /api/users/**   ──► authService      :8080  (requiere JWT)
        ├── /api/projects/**──► proyectsService  :8081  (requiere JWT)
        ├── /api/clients/** ──► proyectsService  :8081  (requiere JWT)
        ├── /api/phases/**  ──► proyectsService  :8081  (requiere JWT)
        └── /api/tasks/**   ──► proyectsService  :8081  (requiere JWT)
```

---

## Flujo de autenticación

```
1. Browser envía:  Authorization: Bearer <JWT>
2. Gateway extrae el token del header
3. JwtValidationGatewayFilterFactory verifica la firma con el secret compartido
4. Si es válido → extrae userId y userName del payload
5. Inyecta headers X-User-Id y X-User-Name en el request
6. Reenvía el request al microservicio correspondiente
7. Si es inválido → retorna 401 Unauthorized
```

---

## Stack

| Tecnología | Versión | Uso |
|---|---|---|
| Spring Boot | 3.4.5 | Framework base |
| Spring Cloud Gateway | 4.2.1 | Enrutamiento reactivo |
| Spring WebFlux | 3.4.5 | Motor reactivo (no bloqueante) |
| Spring Security | 6.x | Seguridad WebFlux |
| JJWT | 0.12.6 | Validación de tokens JWT |

---

## Estructura del proyecto

```
src/main/java/cl/duoc/bffGateway/
├── BffGatewayApplication.java
├── config/
│   └── SecurityConfig.java                      ← Seguridad WebFlux (permitAll)
├── filter/
│   └── JwtValidationGatewayFilterFactory.java   ← Filtro JWT personalizado
└── service/
    └── JwtService.java                          ← Validación y extracción de claims

src/main/resources/
└── application.yml                              ← Rutas y configuración CORS
```

---

## Componentes clave

### `JwtValidationGatewayFilterFactory`

Filtro personalizado que se aplica a las rutas protegidas. Extiende `AbstractGatewayFilterFactory` — Spring Cloud Gateway lo registra automáticamente con el nombre `JwtValidation` (quitando el sufijo `GatewayFilterFactory`).

```java
@Component
public class JwtValidationGatewayFilterFactory
    extends AbstractGatewayFilterFactory<JwtValidationGatewayFilterFactory.Config> {

    @Override
    public GatewayFilter apply(Config config) {
        return (exchange, chain) -> {
            // 1. Leer Authorization header
            // 2. Validar firma JWT
            // 3. Inyectar X-User-Id y X-User-Name
            // 4. Reenviar al microservicio
        };
    }
}
```

### `JwtService`

Valida tokens y extrae claims. Usa el mismo `jwt.secret` que el `authService` — así puede verificar tokens sin necesitar comunicarse con el servicio de autenticación.

```java
private SecretKey getSigningKey() {
    byte[] keyBytes = secret.getBytes(StandardCharsets.UTF_8);
    return Keys.hmacShaKeyFor(keyBytes);
}
```

### `SecurityConfig`

Configuración reactiva (`@EnableWebFluxSecurity`). El Gateway no gestiona autenticación propia — delega esa responsabilidad al filtro JWT personalizado.

```java
.authorizeExchange(exchanges -> exchanges
    .anyExchange().permitAll()  // el filtro JWT maneja la auth
)
```

### `application.yml`

Define las rutas. Las rutas con `- JwtValidation` requieren token válido; las que no tienen el filtro son públicas.

```yaml
routes:
  - id: auth-service          # público
    uri: http://localhost:8080
    predicates:
      - Path=/api/auth/**

  - id: projects-service      # protegido
    uri: http://localhost:8081
    predicates:
      - Path=/api/projects/**
    filters:
      - JwtValidation
```

---

## Configuración

### `application.yml`

```yaml
server:
  port: 8090

jwt:
  secret: <mismo-secret-que-authService>
```

> **Importante**: el `jwt.secret` debe ser idéntico al configurado en el `authService`. Si difieren, el Gateway no podrá verificar los tokens generados por el servicio de autenticación.

### CORS

El Gateway centraliza la configuración CORS para todos los microservicios:

```yaml
globalcors:
  cors-configurations:
    '[/**]':
      allowedOriginPatterns:
        - "http://localhost:5173"
      allowedMethods: [GET, POST, PUT, PATCH, DELETE, OPTIONS]
      allowedHeaders: ["*"]
      allowCredentials: true
```

---

## Agregar un nuevo microservicio

1. Levantar el nuevo servicio en un puerto libre (ej: `8083`)
2. Agregar la ruta en `application.yml`:

```yaml
- id: resources-service
  uri: http://localhost:8083
  predicates:
    - Path=/api/resources/**
  filters:
    - JwtValidation
```

3. Reiniciar el Gateway — no hay cambios en el frontend

---

## Ejecutar

```bash
./mvnw spring-boot:run
```

El Gateway levanta en `http://localhost:8090`.

El frontend (Vite) debe apuntar todo su tráfico al Gateway:

```ts
// vite.config.ts
proxy: {
  '/api': {
    target: 'http://localhost:8090',
    changeOrigin: true,
  }
}
```
