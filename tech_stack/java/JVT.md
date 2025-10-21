# `JWT`

**JWT (JSON Web Token)** - это открытый стандарт (**RFC 7519**) для создания токенов доступа, основанных на **JSON
**.<br>
**JWT** позволяют безопасно передавать информацию между сторонами в виде компактного и самодостаточного объекта.

<details closed> 
    <summary>
        <b>Основные концепции JWT</b>
    </summary>

Ключевые характеристики JWT:

- Компактность: Может быть передан через URL, POST параметры или внутри HTTP заголовков
- Самодостаточность: Содержит всю необходимую информацию о пользователе
- Безопасность: Подписывается цифровой подписью для проверки целостности
- Статус: Не требует хранения состояния на сервере (**stateless**)

Основные сценарии использования:

- Аутентификация: После входа пользователя, каждый последующий запрос будет содержать JWT
- Обмен информацией: Безопасная передача данных между сторонами
- Авторизация: Определение прав доступа на основе claims в токене

Преимущества:

- Масштабируемость (не требует сессий на сервере)
- Кроссплатформенность
- Поддержка различных алгоритмов подписи
- Легкая интеграция с мобильными и SPA приложениями

</details>

## Структура JWT

<details closed> 
    <summary>
        <b>Компоненты JWT токена</b>
    </summary>

JWT состоит из трех частей, разделенных точками:

### `header.payload.signature`

Пример JWT токена:

```text
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.
eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

### Header (Заголовок)
Содержит метаданные о типе токена и алгоритме подписи.

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

`alg`: Алгоритм подписи (HS256, RS256, ES256, etc.)<br>
`typ`: Тип токена (обычно "JWT")

### Payload (Полезная нагрузка)
Содержит **claims** (утверждения) - утверждения о сущности и дополнительных данных.

```json
{
  "sub": "1234567890",
  "name": "John Doe",
  "admin": true,
  "iat": 1516239022,
  "exp": 1516242622
}
```

### Signature (Подпись)
Создается путем кодирования header и payload с использованием секретного ключа и указанного алгоритма.

```java
String signature = HMACSHA256(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  secret
);
```
</details>

## Claims (Утверждения) в JWT

<details closed> 
    <summary>
        <b>Типы claims и их назначение</b>
    </summary>

### Зарегистрированные claims (стандартные):

- `iss` (Issuer) - издатель токена
- `sub` (Subject) - субъект токена (обычно user ID)
- `aud` (Audience) - получатель токена
- `exp` (Expiration Time) - время истечения токена
- `nbf` (Not Before) - время, с которого токен становится валидным
- `iat` (Issued At) - время создания токена
- `jti` (JWT ID) - уникальный идентификатор токена

### Public claims: 
Могут быть определены по усмотрению, но должны быть зарегистрированы в **IANA JSON Web Token Registry**

### Private claims: 
Пользовательские claims для обмена информацией между сторонами

### Пример payload с различными claims:
```json
{
  // Зарегистрированные claims
  "iss": "my-auth-server",
  "sub": "user-12345",
  "aud": "my-api",
  "exp": 1716242622,
  "iat": 1716239022,
  "jti": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  
  // Public claims
  "name": "John Doe",
  "email": "john.doe@example.com",
  
  // Private claims
  "roles": ["ROLE_USER", "ROLE_ADMIN"],
  "permissions": ["read:data", "write:data"],
  "tenant_id": "company-abc"
}
```

### Рекомендации по использованию claims:

- Используйте `exp` для ограничения времени жизни токена
- Включайте `jti` для предотвращения replay атак
- Используйте `sub` для идентификации пользователя
- Ограничивайте количество custom claims для уменьшения размера токена
</details>

## Алгоритмы подписи JWT
<details closed> 
    <summary>
        <b>Типы алгоритмов и их безопасность</b>
    </summary>

### HMAC с SHA-2 (Симметричные алгоритмы)


- `HS256` - `HMAC` с `SHA-256`
- `HS384` - `HMAC` с `SHA-384`  
- `HS512` - `HMAC` с `SHA-512`

Использование: один секретный ключ для подписи и проверки

```java
String secret = "my-super-secret-key";
```

### RSA с SHA-2 (Асимметричные алгоритмы)

- `RS256` - `RSA` с `SHA-256`
- `RS384` - `RSA` с `SHA-384`
- `RS512` - `RSA` с `SHA-512`

Использование: приватный ключ для подписи, публичный для проверки

```java
PrivateKey privateKey = // загрузка приватного ключа
PublicKey publicKey = // загрузка публичного ключа
```

### ECDSA с SHA-2 (Эллиптические кривые)

- `ES256` - `ECDSA` с `P-256` и `SHA-256`
- `ES384` - `ECDSA` с `P-384` и `SHA-384`
- `ES512` - `ECDSA` с `P-521` и `SHA-512`

Использование: более эффективные асимметричные алгоритмы

### Рекомендации по выбору алгоритма:

- Для микросервисов: `RS256` или `ES256` (публичные ключи можно распространять)
- Для монолитных приложений: `HS256` (проще в настройке)
- Высокая безопасность: `ES512` или `RS512`
- Мобильные приложения: `ES256` (меньший размер ключей)

### Пример создания ключей:

```java
// Генерация RSA ключей
KeyPairGenerator keyPairGenerator = KeyPairGenerator.getInstance("RSA");
keyPairGenerator.initialize(2048);
KeyPair keyPair = keyPairGenerator.generateKeyPair();

PrivateKey privateKey = keyPair.getPrivate();
PublicKey publicKey = keyPair.getPublic();

// Сохранение ключей в формате PEM
String privateKeyPem = "-----BEGIN PRIVATE KEY-----\n" +
                       Base64.getEncoder().encodeToString(privateKey.getEncoded()) +
                       "\n-----END PRIVATE KEY-----";
```
</details>

### Работа с JWT в Java
<details closed> 
    <summary>
        <b>Библиотеки для работы с JWT</b>
    </summary>

### Популярные Java библиотеки:

- `jjwt` (Java JWT) - наиболее популярная
- `auth0` java-jwt - от Auth0
- `nimbus-jose-jwt` - полная реализация JOSE
- `java-jwt` - от jsonwebtoken.io

### Создание и валидация JWT токена с использованием JJWT:

```java
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.SignatureAlgorithm;
import io.jsonwebtoken.security.Keys;
import java.security.Key;
import java.util.Date;

public class JwtUtil {
    
    private final Key secretKey;
    private final long expirationMs;
    
    public JwtUtil(String secret, long expirationMs) {
        this.secretKey = Keys.hmacShaKeyFor(secret.getBytes());
        this.expirationMs = expirationMs;
    }
    
    public String generateToken(String username, List<String> roles) {
        Date now = new Date();
        Date expiryDate = new Date(now.getTime() + expirationMs);
        
        return Jwts.builder()
                .setSubject(username)
                .claim("roles", roles)
                .setIssuedAt(now)
                .setExpiration(expiryDate)
                .signWith(secretKey, SignatureAlgorithm.HS256)
                .compact();
    }
    
    public String generateToken(UserDetails userDetails) {
        Map<String, Object> claims = new HashMap<>();
        claims.put("roles", userDetails.getAuthorities().stream()
                .map(GrantedAuthority::getAuthority)
                .collect(Collectors.toList()));
        
        return Jwts.builder()
                .setClaims(claims)
                .setSubject(userDetails.getUsername())
                .setIssuedAt(new Date())
                .setExpiration(new Date(System.currentTimeMillis() + expirationMs))
                .signWith(secretKey, SignatureAlgorithm.HS256)
                .compact();
    }

    public String getUsernameFromToken(String token) {
        return Jwts.parserBuilder()
                .setSigningKey(secretKey)
                .build()
                .parseClaimsJws(token)
                .getBody()
                .getSubject();
    }

    public List<String> getRolesFromToken(String token) {
        Claims claims = Jwts.parserBuilder()
                .setSigningKey(secretKey)
                .build()
                .parseClaimsJws(token)
                .getBody();

        return claims.get("roles", List.class);
    }

    public boolean validateToken(String token) {
        try {
            Jwts.parserBuilder()
                    .setSigningKey(secretKey)
                    .build()
                    .parseClaimsJws(token);
            return true;
        } catch (Exception e) {
            // Логирование ошибки валидации
            return false;
        }
    }

    public boolean isTokenExpired(String token) {
        Date expiration = Jwts.parserBuilder()
                .setSigningKey(secretKey)
                .build()
                .parseClaimsJws(token)
                .getBody()
                .getExpiration();
        return expiration.before(new Date());
    }
}
```

### Расширенная валидация с кастомными проверками:

```java
public class AdvancedJwtValidator {
    
    private final JwtParser parser;
    
    public AdvancedJwtValidator(Key secretKey) {
        this.parser = Jwts.parserBuilder()
                .setSigningKey(secretKey)
                .requireIssuer("my-auth-server")
                .requireAudience("my-api")
                .build();
    }
    
    public JwtValidationResult validateToken(String token) {
        try {
            Claims claims = parser.parseClaimsJws(token).getBody();
            
            // Дополнительные проверки
            if (!hasRequiredRoles(claims)) {
                return JwtValidationResult.failure("Insufficient roles");
            }
            
            if (isTokenRevoked(claims.getId())) {
                return JwtValidationResult.failure("Token revoked");
            }
            
            return JwtValidationResult.success(claims);
            
        } catch (ExpiredJwtException e) {
            return JwtValidationResult.failure("Token expired");
        } catch (MalformedJwtException e) {
            return JwtValidationResult.failure("Invalid token format");
        } catch (SecurityException e) {
            return JwtValidationResult.failure("Invalid signature");
        } catch (Exception e) {
            return JwtValidationResult.failure("Token validation failed: " + e.getMessage());
        }
    }
    
    public static class JwtValidationResult {
        
        private final boolean valid;
        private final String error;
        private final Claims claims;
        
        private JwtValidationResult(boolean valid, String error, Claims claims) {
            this.valid = valid;
            this.error = error;
            this.claims = claims;
        }
        
        public static JwtValidationResult success(Claims claims) {
            return new JwtValidationResult(true, null, claims);
        }
        
        public static JwtValidationResult failure(String error) {
            return new JwtValidationResult(false, error, null);
        }
        
        // геттеры
    }
}
```