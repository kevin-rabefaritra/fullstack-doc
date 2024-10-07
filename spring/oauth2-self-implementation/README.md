# OAuth 2.0 - Self implementation with resource server

Implement a resource server in your app, it covers:
- authentication mechanism for users.
- web **access** and **refresh tokens** generation.
- token validity verification.

We will create 3 endpoints:
1. User authentication
2. Renew access token using refresh token
4. Access protected resources
3. Get user info using their access token

## User management
Create a `UserDetailsService` bean. For simplicity, we will use `In-Memory Authentication`(more details [here](https://docs.spring.io/spring-security/reference/servlet/authentication/passwords/in-memory.html)).

```java
import org.springframework.security.core.userdetails.User;
...
@Bean
public UserDetailsService userDetailsService() {
    UserDetails user1 = User.builder()
        .username("user1")
        .password("{noop}password1")
        .build();
	UserDetails user2 = User.builder()
        .username("admin")
        .password("{noop}password2")
        .build();
	return new InMemoryUserDetailsManager(user1, user2);
}
```

In a real-case scenario, user credentials should be **encrypted** and **stored in a persistent storage**.

## Security configuration
**Step 1**: Create a `SecurityFilterChain` bean to protect endpoints and register our OAuth 2.0 resource server.

```java
@Bean
SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
	return httpSecurity
		.csrf(AbstractHttpConfigurer::disable) // CSRF not required in our scenario
		.sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
		.authorizeHttpRequests(requests -> {
			authorizeRequests.anyRequest().authenticated();
		})
		.oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults())) // used for token processing
		.httpBasic(Customizer.withDefaults()) // Used for user basic authentication
		.build();
}
```

**Step 2**: Create a RSA public and private key.

We will generate a PKCS8 key pair `public.pem` and `private.pem` and put them under the `main/resources` folder. You can use online tools to generate them.

We can reference the keys in our `application.properties` file.

```properties
rsa.public-key=classpath:public.pem
rsa.private-key=classpath:private.pem
```

**Step 3**: Configure the JWT encoder / decoder.

We create two beans `JwtEncoder` and `JwtDecoder` then use our keys to encode and decode.

```java
public WebConfig(
	@Value("${rsa.public-key}") RSAPublicKey publicKey,
	@Value("${rsa.private-key}") RSAPrivateKey privateKey
) {
	this.rsaPublicKey = publicKey;
	this.rsaPrivateKey = privateKey;
}

@Bean
JwtEncoder jwtEncoder() {
	JWK jwk = new RSAKey.Builder(this.rsaPublicKey).privateKey(this.rsaPrivateKey).build();
	JWKSource<SecurityContext> jwkSource = new ImmutableJWKSet<>(new JWKSet(jwk));
	return new NimbusJwtEncoder(jwkSource);
}

@Bean
JwtDecoder jwtDecoder() {
	return NimbusJwtDecoder.withPublicKey(this.rsaPublicKey).build();
}
```

## Generating a token
**Step 1**: We need to create a service that generates tokens from an `Authentication`.

```java
@Autowired
private JwtEncoder jwtEncoder;
...
public String generateToken(Authentication authentication) {
	Instant now = Instant.now();
	String scope = authentication.getAuthorities().stream()
			.map(GrantedAuthority::getAuthority)
			.collect(Collectors.joining(" "));
	JwtClaimsSet claims = JwtClaimsSet.builder()
			.issuer("self")
			.issuedAt(now)
			.expiresAt(now.plus(2, ChronoUnit.HOURS))
			.subject(authentication.getName())
			.build();
	return this.jwtEncoder.encode(JwtEncoderParameters.from(claims)).getTokenValue();
}
```

**Step 2**: the `Authentication` object can be obtained from a controller endpoint.

```java
@GetMapping("/token")
String authenticate(Authentication authentication) {
	tokenService.generateToken(authentication);
}
```

That is, if the user authenticates successfully using the *basic authentication* `httpBasic` registered in our `SecurityFilterChain` bean, the request will reach our controller with the user info in the `Authentication` parameter.

We then pass this `Authentication` parameter to our `generateToken` method and return the token.