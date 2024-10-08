# OAuth 2.0 - Self implementation with resource server

Implement a resource server in your app, it covers:
- authentication mechanism for users.
- web **access tokens** generation.
- token validity verification.

## Implementation

### 1. Add Resource server dependency
Add the OAuth 2.0 resource server dependency, which includes **Spring Security**.
```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>
```

### 2. Implement UserDetailsService
Create a `UserDetailsService` bean. For simplicity, we will use `In-Memory Authentication`(more details [here](https://docs.spring.io/spring-security/reference/servlet/authentication/passwords/in-memory.html)).

```java
import org.springframework.security.core.userdetails.User;
...
@Bean
public UserDetailsService userDetailsService() {
    UserDetails user = User.builder()
        .username("user1")
        .password("{noop}password1")
        .build();
	return new InMemoryUserDetailsManager(user);
}
```

In a real-case scenario, user credentials should be **encrypted** and **stored in a persistent storage**.

### 3. Security configuration
**Step 1**: Create a `SecurityFilterChain` bean to protect endpoints and register our OAuth 2.0 resource server.

```java
@Bean
SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
	return httpSecurity
	.csrf(AbstractHttpConfigurer::disable) // CSRF not required since we don't use sessions
	.sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
	.authorizeHttpRequests(requests -> {
		authorizeRequests.anyRequest().authenticated();
	}) // require all requests to be authenticated
	.oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults())) // used for JWT processing
	.httpBasic(Customizer.withDefaults()) // Used for user basic authentication
	.build();
}
```

**Step 2**: Create a RSA public and private key.

We will generate a PKCS8 key pair `public.pem` and `private.pem` and put them under the `main/resources` folder. You can use an online tool to generate them.

Then we can reference the keys in our `application.properties` file.

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

### 4. Generating a token

Through basic authentication, we can retrieve the authenticated user information from the `Authentication` object provided in our endpoints.

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
		.issuer("self") // we can put any issuer here
		.issuedAt(now)
		.expiresAt(now.plus(2, ChronoUnit.HOURS)) // define the expiration date of the token
		.subject(authentication.getName())
		.build();
	return this.jwtEncoder.encode(JwtEncoderParameters.from(claims)).getTokenValue();
}
```

**Step 2**: the `Authentication` object can be obtained from a controller endpoint.

```java
@GetMapping("/auth")
String authenticate(Authentication authentication) {
	tokenService.generateToken(authentication);
}
```

**Step 3**: we can now authenticate to obtain a token

```bash
curl --request GET \
  --url http://localhost:8080/api/oauth2/auth \
  --header 'Authorization: Basic <USER BASE64 USERNAME:PASSWORD>'
```

**Step 4**: a token can be used to access a protected resource by passing it to the request header

```bash
curl --request GET \
  --url http://localhost:8080/api/protectedresource \
  --header 'Authorization: Bearer <OUR TOKEN>'
```

## Improvements
### Restricting basic auth
You may notice that the basic authentication can also be used to access the protected resource.

```bash
curl --request GET \
  --url http://localhost:8080/api/protectedresource \
  --header 'Authorization: Basic <USER BASE64 USERNAME:PASSWORD>'
```

Ideally, we would like to only allow basic auth for the `/api/oauth2/auth` endpoint.

#### Option 1: Add a separate configuration
Add a separate configuration for the authentication endpoint so that basic auth can be used. We create another `SecurityFilterChain` bean with a higher precedence.

```java
@Bean
@Order(1)
public SecurityFilterChain securityFilterChainBasic(HttpSecurity http) throws Exception {
	return http
		.securityMatcher("api/oauth2/auth") // only applies to this endpoint
		.authorizeHttpRequests(authorizeRequests -> {
			authorizeRequests.anyRequest().authenticated();
		})
		.httpBasic(Customizer.withDefaults())
		.build();
}
```

then we remove basic auth from the initial `SecurityFilterChain` and set it to a lower precedence.

```java
@Bean
@Order(2)
SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
	return httpSecurity
		.csrf(AbstractHttpConfigurer::disable) // CSRF not required since we don't use sessions
		.sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
		.authorizeHttpRequests(requests -> {
			authorizeRequests.anyRequest().authenticated();
		}) // require all requests to be authenticated
		.oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults())) // used for JWT processingauthentication
		.build();
}
```

#### Option 2: Implement AuthenticationManager
With this approach, we get rid of basic authentication, use only one `SecurityFilterChain` and authenticate using a classic `POST` request.

**Step 1**: Register a password encoder.

```java
@Bean
PasswordEncoder passwordEncoder() {
	return PasswordEncoderFactories.createDelegatingPasswordEncoder();
}
```

and update the `UserDetailsService` bean to use it.

```java
import org.springframework.security.core.userdetails.User;
...
@Bean
public UserDetailsService userDetailsService() {
    UserDetails user = User.builder()
        .username("user1")
        .password(passwordEncoder().encode("password1"))
        .build();
	return new InMemoryUserDetailsManager(user);
}
```

**Step 2**: Create an authentication provider

```java
AuthenticationProvider authenticationProvider() {
	DaoAuthenticationProvider authProvider = new DaoAuthenticationProvider();
	// our UserDetailsService implementation
	authProvider.setUserDetailsService(userDetailsService());

	// our PasswordEncoder (cannot be null)
	authProvider.setPasswordEncoder(passwordEncoder());
	return authProvider;
}
```

**Step 3**: Register the authentication provider in the `SecurityFilterChain`

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
	return http
		.csrf(AbstractHttpConfigurer::disable)
		.sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
		.authorizeHttpRequests(authorizeRequests -> {
			// expose the auth endpoint
			authorizeRequests.requestMatchers("api/oauth2/auth").permitAll();
			authorizeRequests.anyRequest().authenticated();
		})
		.oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()))
		.authenticationProvider(authenticationProvider()) // our authentication provider
		.build();
}
```

**Step 4**: Create an authentication manager bean
```java
@Bean
AuthenticationManager authenticationManager(AuthenticationConfiguration configuration) throws Exception {
	return configuration.getAuthenticationManager();
}
```

**Step 5**: Update the authentication controller method

```java
@PostMapping("/auth")
String authenticate(@RequestBody SignInRequest request) {
	return tokenService.authenticate(request);
}
```

`SignInRequest` just wraps the username and password.

```java
public record SignInRequest(String username, String password) {}
```

**Step 6**: Authenticate the user

We inject the `AuthenticationManager` bean to do the authentication for us.

```java
class TokenService {

	@Autowired
	private AuthenticationManager authenticationManager;

	...
	String authenticate(SignInRequest request) throws AuthenticationException {
		UsernamePasswordAuthenticationToken authenticationToken = new UsernamePasswordAuthenticationToken(username, password);
		Authentication authentication = this.authenticationManager.authenticate(authenticationToken);
		return generateToken(authentication); // method implemented above
	}
}
```

Now we can authenticate using a POST request.

```bash
curl --request POST \
  --url http://localhost:8080/api/oauth2/auth \
  --header 'content-type: application/json' \
  --data '{
  "username": "user1",
  "password": "password1"
}'
```