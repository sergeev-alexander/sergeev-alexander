## 🛡️ Spring Security

🔐 Что такое Spring Security?

Spring Security — это мощный и гибкий фреймворк для обеспечения аутентификации (authentication) и авторизации (authorization) в приложениях на Spring. 
Он позволяет защищать веб-приложения, REST API, методы сервисов и даже отдельные данные.

Spring Security построен на концепции цепочки фильтров (Filter Chain).

Когда HTTP-запрос приходит в приложение, он проходит через серию фильтров, и один из них - SecurityFilterChain, который отвечает за безопасность.

### ⚙️ Базовая настройка
<details> 
    <summary> 
        <b>Зависимости и конфигурация</b> 
    </summary>

```xml
<!-- Maven -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```
```groovy
// Gradle
implementation 'org.springframework.boot:spring-boot-starter-security'
```
```java
@Configuration
// Включает конфигурацию Spring Security. Импортирует WebSecurityConfiguration и другие необходимые классы связанные с безопасностью.
// в Spring Boot 3+ (с Spring Security 6), если есть хотя бы один @Bean SecurityFilterChain, Spring Boot автоматически включает WebSecurity, и @EnableWebSecurity не обязателен.
@EnableWebSecurity
public class SecurityConfig {

    // UserDetailsService — это интерфейс, который загружает пользователя по имени (loadUserByUsername)
    // Spring Security использует его для аутентификации.
    @Autowired
    private UserDetailsServiceImpl userDetailsService;

    // jwtAuthFilter - фильтр (обычно наследник OncePerRequestFilter)
    // Извлекает JWT из заголовка, проверяет его и устанавливает аутентификацию в SecurityContext.
    @Autowired
    private JwtAuthFilter jwtAuthFilter;

    // SecurityFilterChain - главный бин, который определяет цепочку фильтров безопасности для HTTP-запросов.
    // Spring Security может иметь несколько SecurityFilterChain бинов, если нужно разное поведение для разных путей (например, /api/** и /admin/**).
    @Bean 
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {  
        
        // HttpSecurity — это DSL (Domain Specific Language), с помощью которого настраивается
        // - какие URL защищены
        // - как происходит аутентификация (форма, JWT, OAuth2 и т.д.)
        // - CSRF-защита
        // - CORS
        // - сессии и пр.

        http
                // Отключает CORS-настройки по умолчанию. Обычно CORS настраивают отдельно, если нужно.
                .cors(AbstractHttpConfigurer::disable)

                // Отключает защиту от CSRF-атак.
                .csrf(AbstractHttpConfigurer::disable) 
                
                // CSRF можно отключать только в stateless API (например, с JWT), потому что CSRF защищает от атак, использующих куки и сессии. 
                // В JWT-приложениях аутентификация передаётся в заголовке (Authorization: Bearer ...), поэтому CSRF не применим. 

                // Указывает Spring Security не создавать HTTP-сессии.
                // Это стандарт для RESTful API с JWT: каждый запрос должен содержать токен, и сервер ничего не хранит о состоянии пользователя.
                .sessionManagement(session ->  
                        session.sessionCreationPolicy(SessionCreationPolicy.STATELESS)) // STATELESS = без сессий = масштабируемость + простота.
                
                // .authorizeHttpRequests(...) — новая (и рекомендуемая) замена устаревшему .authorizeRequests()
                .authorizeHttpRequests(authz -> authz

                        // /api/auth/** — например, /api/auth/login, /api/auth/register — доступны всем (без аутентификации).
                        .requestMatchers("/api/auth/**").permitAll()
                        
                        // Swagger UI и OpenAPI-документация — тоже открыты (для удобства разработки).
                        .requestMatchers("/swagger-ui/**", "/v3/api-docs/**").permitAll()
                        
                        // Все остальные запросы (anyRequest()) требуют аутентификации.
                        .anyRequest().authenticated()
                )

                // Явно указываем, что нужно использовать наш DaoAuthenticationProvider.
                // Хотя Spring Security часто подключает его автоматически, явное указание делает конфигурацию прозрачной.
                .authenticationProvider(authenticationProvider())
                
                // Добавление кастомного JWT-фильтра.
                // UsernamePasswordAuthenticationFilter обрабатывает формы входа.
                // Это нужно, чтобы JWT проверялся до любых других фильтров аутентификации.
                // Этот фильтр должен:
                // - Извлечь токен из Authorization заголовка.
                // - Проверить его подпись и срок действия.
                // - Если валиден — создать Authentication объект и поместить его в SecurityContextHolder.
                .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);

        // Собирает и возвращает сконфигурированную цепочку фильтров.
        return http.build();
    }

    // PasswordEncoder - хеширование и проверка паролей
    // Spring Security автоматически использует бин PasswordEncoder, если он объявлен.
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(); // BCryptPasswordEncoder — рекомендуемый способ хранения паролей (устойчив к атакам, с солью).
    }

    // DaoAuthenticationProvider - стандартный провайдер аутентификации в Spring Security.
    // Получает имя пользователя из запроса.
    // Загружает UserDetails через userDetailsService.
    // Сравнивает введённый пароль с хешем, используя passwordEncoder.
    // Spring Security может использовать несколько AuthenticationProvider
    @Bean
    public DaoAuthenticationProvider authenticationProvider() {
        DaoAuthenticationProvider authProvider = new DaoAuthenticationProvider();
        authProvider.setUserDetailsService(userDetailsService);
        authProvider.setPasswordEncoder(passwordEncoder());
        return authProvider;
    }

    // AuthenticationManager — основной интерфейс для выполнения аутентификации.
    // В Spring Boot он не создаётся автоматически как бин, поэтому его нужно объявить вручную, если нужно внедрять его в другие компоненты (например, в контроллер для ручной аутентификации).
    // Здесь используется AuthenticationConfiguration, предоставляемый Spring Security, чтобы получить настроенный менеджер, который уже знает о AuthenticationProvider.
    @Bean
    public AuthenticationManager authenticationManager(AuthenticationConfiguration authConfig) throws Exception {
        return authConfig.getAuthenticationManager();
    }
}
```
```java
// Реализация Spring Security интерфейса UserDetailsService 
// Основная задача: загрузить данные пользователя по "имени пользователя" (username), которое в контексте аутентификации может быть email, телефоном или логином.
// В Spring Security этот сервис вызывается автоматически при аутентификации (например, при логине через форму, JWT, OAuth2 и т.д.).
// Метод loadUserByUsername(String) ДОЛЖЕН выбрасывать UsernameNotFoundException, если пользователь не найден.
// Иначе Spring Security не сможет корректно обработать ошибку аутентификации.
@Service
public class UserDetailsServiceImpl implements UserDetailsService {
    
    private final UserRepository userRepository;

    @Autowired
    public UserDetailsServiceImpl(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
    
    // Основной метод интерфейса UserDetailsService.
    // Вызывается Spring Security при попытке аутентификации.
    // Хотя метод называется loadUserByUsername, в REST API с email-аутентификацией под "username" часто подразумевается email. Это нормально и широко используется.
    
    // 1. Spring Security получает учетные данные (например, email и пароль из /login).
    // 2. Вызывает этот метод с email в качестве username.
    // 3. Мы ищем пользователя по email в БД.
    // 4. Если не найден — кидаем UsernameNotFoundException → Spring Security вернёт 401.
    // 5. Если найден — возвращаем UserDetails с email, хешем пароля и ролями.
     
    // Здесь возвращается ХЕШ пароля (не plaintext!). Spring Security сам сравнит его с введённым паролем через PasswordEncoder (бин, объявленный в SecurityConfig).
    // @return объект UserDetails, содержащий данные для аутентификации
    @Override
    public UserDetails loadUserByUsername(String email) throws UsernameNotFoundException {
        // Ищем пользователя по email. UserRepository должен возвращать Optional<User>.
        // Если не найден — выбрасываем UsernameNotFoundException!
        User user = userRepository.findByEmail(email)
                .orElseThrow(() -> new UsernameNotFoundException("User not found with email: " + email));

        // Создаём экземпляр стандартного класса UserDetails от Spring Security.
        // Параметры:
        // - username: email (будет доступен как principal.getName())
        // - password: хеш пароля из БД (Spring сам сравнит его через PasswordEncoder)
        // - authorities: коллекция ролей/прав (например, ROLE_USER, ROLE_ADMIN)
        return new org.springframework.security.core.userdetails.User(
                user.getEmail(),          // username
                user.getPassword(),       // password (hashed!)
                getAuthorities(user)      // authorities (roles)
        );
    }

    // Вспомогательный метод для преобразования роли пользователя в Spring Security authorities.
      
    // Стиль ролей:
    // - Spring Security различает "authorities" и "roles".
    // - hasRole('ADMIN') ищет authority с именем "ROLE_ADMIN".
    // - hasAuthority('ADMIN') ищет authority с именем "ADMIN".
      
    // У нас enum Role определён как:
    // public enum Role { ROLE_USER, ROLE_ADMIN }
    // Поэтому user.getRole().name() даёт "ROLE_USER" — что идеально подходит для использования с hasRole('USER') или hasAuthority('ROLE_USER').
      
    
    // Придерживайтесь одного стиля в аннотациях:
    // @PreAuthorize("hasRole('ADMIN')")   → ожидает ROLE_ADMIN
    // @PreAuthorize("hasAuthority('ROLE_ADMIN')") → то же самое, но явно
      
    // @param user — сущность пользователя из БД
    // @return коллекция GrantedAuthority (обычно одна роль)
    private Collection<? extends GrantedAuthority> getAuthorities(User user) {
        // SimpleGrantedAuthority — стандартная реализация GrantedAuthority.
        // Передаём имя роли как есть (уже с префиксом ROLE_). Если роли без префикса, то добавляем ("ROLE_" + user.getRole()) 
        return Collections.singletonList(
                new SimpleGrantedAuthority(user.getRole().name())
        );
    }

    /*
     * ДОПОЛНИТЕЛЬНЫЕ ВОЗМОЖНОСТИ:
     * 
     * 1. ПОДДЕРЖКА НЕСКОЛЬКИХ РОЛЕЙ:
     *    Если у пользователя может быть несколько ролей, храни их в отдельной таблице
     *    и возвращай список:
     *    
     *    return user.getRoles().stream()
     *        .map(role -> new SimpleGrantedAuthority("ROLE_" + role.name()))
     *        .collect(Collectors.toList());
     * 
     * 2. УЧЁТ СТАТУСА ПОЛЬЗОВАТЕЛЯ:
     *    Spring UserDetails поддерживает флаги:
     *      - isEnabled() → для блокировки аккаунта
     *      - isAccountNonExpired() → срок действия аккаунта
     *      - isCredentialsNonExpired() → срок действия пароля
     *      - isAccountNonLocked() → заблокирован ли
     *    
     *    Пример:
     *    return new org.springframework.security.core.userdetails.User(
     *        user.getEmail(),
     *        user.getPassword(),
     *        user.isEnabled(),
     *        user.isAccountNonExpired(),
     *        user.isCredentialsNonExpired(),
     *        user.isAccountNonLocked(),
     *        getAuthorities(user)
     *    );
     * 
     * 3. КЕШИРОВАНИЕ:
     *    Для высоконагруженных систем можно добавить @Cacheable:
     *    
     *    @Cacheable("users")
     *    @Override
     *    public UserDetails loadUserByUsername(String email) { ... }
     */
}
```

