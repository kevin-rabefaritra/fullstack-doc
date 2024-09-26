# OAuth2 Authentication Server with Spring Boot

## Key points
- It is in charge of authenticating users and delivers tokens that can be used to access various services.
- It acts as a centralized point of *authorization management*.
- It is typically used with one or more *resource servers* i.e. servers that holds protected resources.

*You can think of it as a **ticketing office** of an amusement park. A ticket has been bought on your name and lets you access various attractions for a certain period of time.*

## Setup
### Step 1: Installation
For Maven
```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-authorization-server</artifactId>
</dependency>
```
or using Gradle
```
implementation "org.springframework.boot:spring-boot-starter-oauth2-authorization-server"
```

### Step 2: Configuration
We need to configure the following:

1. **Login page location** for non-authenticated users to login.

2. **UserDetailsService** implementation that manages end-users credentials and roles.

3. **RegisteredClientRepository** to manage *OAuth 2.0 clients* i.e. web or mobile apps that can request tokens. For each client we need:
- **Client ID and secret** to ensure requesters are legit.
- **Authentication method(s)** how the client will authenticate with their ID & secret: *Basic* HTTP header, *JWT*, *POST* method, *TLS*, or none.

- **Grant type(s)** that are allowed for the client.
- **Scope(s)**

5. **AuthorizationServerSettings** bean to configure the Spring authorization server.

Optionally, we can include:

1. **JWKSource** to sign access tokens.

2. **KeyPair** required by the `JWKSource` implementation.

3. **JwtDecoder** to decode signed access tokens.
