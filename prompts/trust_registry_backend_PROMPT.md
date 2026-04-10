# 🤖 AI SYSTEM PROMPT: REPO GENERATION INSTRUCTIONS

## 1. MACRO CONTEXT: THE BIG PICTURE
**Project:** TrustVault (Makeathon: Portable Proof with VC and VP)
**Goal:** This repository implements the **Trust Registry** backend. In a decentralized identity ecosystem, Verifiers (Employers/Banks) receive digitally signed data from a mobile phone. Because there is no central authority verifying the data in real-time, the Verifier must consult a Trust Registry—a decentralized "Phonebook" or "Whitelist"—to answer two questions:
1. Is this Public Key/DID actually associated with a legitimate organization?
2. Is that organization currently authorized (ACTIVE), or has it been SUSPENDED/REVOKED?

## 2. REPOSITORY ROLE: TRUST REGISTRY BACKEND
**Function:** A lightweight Java Spring Boot microservice. It provides a REST API to allow "Global Admins" to onboard new Issuers (Universities) and allows "Verifiers" (Employers) to dynamically resolve a DID (Decentralized Identifier) to fetch the Issuer's cryptographic Public Key and current trust status.

## 3. TECH STACK & EXACT DEPENDENCIES
*   **Language:** Java 21
*   **Framework:** Spring Boot 3.5.0 (Maven)
*   **Dependencies:** `spring-boot-starter-web`, `spring-boot-starter-data-jpa`, `h2` (In-memory DB), `spring-boot-starter-validation`, `lombok`.

## 4. DATABASE & INITIALIZATION (`src/main/resources/`)
*   **`application.yml`:**
    ```yaml
    spring:
      application:
        name: trust-registry
      datasource:
        url: jdbc:h2:mem:registrydb
        driverClassName: org.h2.Driver
        username: sa
        password: password
      jpa:
        database-platform: org.hibernate.dialect.H2Dialect
        hibernate:
          ddl-auto: update
        defer-datasource-initialization: true
    server:
      port: 8080
    ```
*   **`data.sql` (Crucial for MVP Demo):**
    You MUST seed the database with the University of Texas so the Issuer backend has a valid record to test against.
    `INSERT INTO issuer (did, name, public_key, status) VALUES ('did:web:university.edu', 'University of Texas', '01234567890123456789012345678912', 'ACTIVE');`

## 5. MODELS & REPOSITORY (`com.infosys.trustvault.registry`)
*   **`model/Issuer.java`:** 
    *   Annotations: `@Entity`, `@Data`, `@NoArgsConstructor`.
    *   Fields: `Long id` (`@Id`, `@GeneratedValue(strategy = GenerationType.IDENTITY)`), `String did`, `String name`, `String publicKey`, `String status`.
*   **`repository/IssuerRepository.java`:** 
    *   Extends `JpaRepository<Issuer, Long>`.
    *   Requires a custom method: `Optional<Issuer> findByDid(String did);`

## 6. CORE SERVICES & CONTROLLER (`RegistryController.java`)
**Base Mapping:** `@RequestMapping("/api/v1/registry")`

*   **`GET /issuers`**: Returns `ResponseEntity<List<Issuer>>` using `issuerRepository.findAll()`.
*   **`POST /issuers`**: Accepts `@RequestBody Issuer issuer`.
    *   *Validation:* Must check `if (issuerRepository.findByDid(issuer.getDid()).isPresent())` and return `400 Bad Request` if the DID already exists.
    *   *Logic:* Forces `issuer.setStatus("ACTIVE")` before saving.
*   **`PUT /issuers/{id}/status`**: Accepts `@PathVariable Long id` and `@RequestBody java.util.Map<String, String> body`.
    *   *Logic:* Fetches issuer by ID. Updates the status field from `body.get("status")` (e.g., "SUSPENDED" or "REVOKED"). Saves and returns the updated issuer.
*   **`GET /resolve/{did}` (THE MOST IMPORTANT API):**
    *   *Logic:* Uses `findByDid(did)`. If found, it MUST return a strictly mapped JSON object representing the DID Document fragment.
    *   *Returns:* `ResponseEntity.ok(java.util.Map.of("did", issuer.getDid(), "name", issuer.getName(), "status", issuer.getStatus(), "publicKey", issuer.getPublicKey()));`
    *   *Error:* Returns `404 Not Found` if the DID is not in the registry.

## 7. GLOBAL CONFIGURATION (`CorsConfig.java`)
Because this registry must be queried by multiple different frontends and backends (Issuers, Verifiers, Mobile Web Views), Cross-Origin Resource Sharing (CORS) must be globally disabled.
*   Create a `@Bean` returning `WebMvcConfigurer`.
*   Override `addCorsMappings`.
*   `registry.addMapping("/**").allowedOrigins("*").allowedMethods("*");`

## 8. DEPLOYMENT (`Dockerfile`)
*   Multi-stage build.
*   **Stage 1:** `FROM eclipse-temurin:21-jdk-alpine AS build`. Run `./mvnw clean package -DskipTests`.
*   **Stage 2:** `FROM eclipse-temurin:21-jre-alpine`. Copy `registry-0.0.1-SNAPSHOT.jar` to `app.jar`. Expose `8080`. Set `ENTRYPOINT ["java", "-jar", "app.jar"]`.
